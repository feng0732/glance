# Glance 缓存与刷新调度协作机制

本文档从代码层面梳理 Glance 中缓存键、TTL、后台刷新、失败短缓存和击穿保护五个方面的协作关系。

## 一、整体架构概览

Glance 的缓存体系是一个**以 Widget 为基本单位的内存缓存系统**，没有使用外部 Redis 等缓存中间件。核心协作流程如下：

```
HTTP 请求到达
    ↓
handlePageContentRequest()  [glance.go#L334-L367]
    ↓ 加 page.mu 互斥锁
page.updateOutdatedWidgets()  [glance.go#L233-L270]
    ↓
widget.requiresUpdate(now)  [widget.go#L173-L183]
    ├─ cacheTypeInfinite → 永不更新
    ├─ cacheTypeDuration → now.After(nextUpdate) 判定
    └─ cacheTypeOnTheHour → 整点触发
    ↓ 需要更新？
    ├─ 否 → 直接用已渲染内容
    └─ 是 → 启动 goroutine 调用 widget.update(ctx)
              ↓
         数据获取（可能嵌套 Singleflight）
              ↓
         canContinueUpdateAfterHandlingErr(err)  [widget.go#L293-L325]
              ├─ 成功 → scheduleNextUpdate()        正常 TTL
              ├─ 部分成功 → scheduleEarlyUpdate()   指数退避短 TTL
              └─ 完全失败 → scheduleEarlyUpdate()   指数退避短 TTL
```

---

## 二、缓存键（Cache Key）

Glance 采用多层级缓存键设计，不同层级使用不同的键策略。

### 2.1 Widget 实例级缓存键

每个 Widget 通过**原子自增的 uint64 ID** 唯一标识：

- 定义位置：[widget.go#L18](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget.go#L18)
- 生成方式：`widgetIDCounter.Add(1)`，在 `newWidget()` 中分配
- 存储位置：`widgetBase.ID` 字段
- 全局索引：`application.widgetByID map[uint64]widget` [glance.go#L38](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/glance.go#L38)

### 2.2 Widget 内部资源缓存键

部分 Widget 在内部维护二级缓存，以业务标识为键：

| Widget | 缓存键类型 | 定义位置 |
|--------|-----------|---------|
| RSS | Feed URL 字符串 | [widget-rss.go#L45-L47](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-rss.go#L45-L47) `map[string]*cachedRSSFeed` |
| Reddit | 全局 loid cookie（单例） | [widget-reddit.go#L362-L387](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-reddit.go#L362-L387) |

RSS Widget 的 `cachedRSSFeed` 还包含 HTTP 协议级缓存字段 `etag` 和 `lastModified`，用于条件请求（`If-None-Match` / `If-Modified-Since`）。

### 2.3 静态资源缓存键

静态文件使用内容哈希 `staticFSHash` 嵌入 URL 路径：
- 位置：[glance.go#L428](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/glance.go#L428)
- 形式：`/static/{hash}/{path}`
- TTL：24 小时，`Cache-Control: public, max-age=86400`

---

## 三、TTL 与缓存类型

每个 Widget 有三种缓存类型，在 `initialize()` 阶段通过链式方法声明。

### 3.1 三种 cacheType

定义于 [widget.go#L141-L147](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget.go#L141-L147)：

| 类型 | 常量值 | 含义 | 设置方法 |
|------|--------|------|---------|
| Infinite | 0 | 永不过期（默认） | 不设置，或 `cacheDuration = -1` |
| Duration | 1 | 固定时长 | `withCacheDuration(d time.Duration)` |
| OnTheHour | 2 | 下一个整点刷新 | `withCacheOnTheHour()` |

### 3.2 各 Widget 默认 TTL

在各 Widget 的 `initialize()` 方法中设置：

| Widget | 缓存类型 | 默认 TTL | 代码位置 |
|--------|---------|---------|---------|
| Weather | OnTheHour | 每小时整点 | [widget-weather.go#L36](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-weather.go#L36) |
| RSS | Duration | 2 小时 | [widget-rss.go#L50](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-rss.go#L50) |
| Reddit | Duration | 30 分钟 | [widget-reddit.go#L96](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-reddit.go#L96) |
| Markets | Duration | 1 小时 | [widget-markets.go#L28](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-markets.go#L28) |
| Change Detection | Duration | 1 小时 | [widget-changedetection.go#L27](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-changedetection.go#L27) |
| Clock / Bookmarks / Iframe | Infinite | 永不过期 | 未设置缓存时长 |

### 3.3 用户自定义 TTL

用户可在 YAML 配置中通过 `cache` 字段覆盖：

```yaml
widgets:
  - type: rss
    cache: 30m  # 覆盖默认的 2h
```

解析逻辑在 [widget.go#L259-L269](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget.go#L259-L269)，优先级：**用户配置 > Widget 默认值**。

### 3.4 nextUpdate 的计算

`getNextUpdateTime()` [widget.go#L327-L341](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget.go#L327-L341)：

```go
func (w *widgetBase) getNextUpdateTime() time.Time {
    now := time.Now()
    if w.cacheType == cacheTypeDuration {
        return now.Add(w.cacheDuration)
    }
    if w.cacheType == cacheTypeOnTheHour {
        // 计算到下一个整点的秒数
        return now.Add(time.Duration(
            ((60-now.Minute())*60)-now.Second(),
        ) * time.Second)
    }
    return time.Time{}  // Infinite 类型返回零值
}
```

---

## 四、后台刷新调度

Glance 采用**请求驱动的惰性刷新**（lazy refresh on request），没有独立的后台定时 goroutine。

### 4.1 刷新触发点

唯一的刷新入口是 `handlePageContentRequest()` [glance.go#L334-L367](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/glance.go#L334-L367)，即前端轮询的页面内容 API。

关键执行流程：

```go
func (a *application) handlePageContentRequest(w http.ResponseWriter, r *http.Request) {
    // ...
    func() {
        page.mu.Lock()          // ① 页面级互斥锁
        defer page.mu.Unlock()

        page.updateOutdatedWidgets()  // ② 刷新所有过期 widget
        err = pageContentTemplate.Execute(&responseBytes, pageData)  // ③ 渲染响应
    }()
    // ...
}
```

`page.mu` 是每个 page 实例的 `sync.Mutex`，定义于 [config.go#L91](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/config.go#L91)。

### 4.2 updateOutdatedWidgets 的实现

[glance.go#L233-L270](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/glance.go#L233-L270)：

1. 获取当前时间 `now`
2. 遍历所有 HeadWidgets 和 Columns 下的 Widgets
3. 对每个 Widget 调用 `requiresUpdate(&now)` 判断
4. 需要更新的 Widget 启动独立 goroutine 执行 `widget.update(context)`
5. `wg.Wait()` 等待所有更新完成后再返回

容器类 Widget（如 group、split-column）在 `widget-container.go#L23-L42` 中遵循相同模式递归处理子 Widget。

### 4.3 requiresUpdate 判定逻辑

[widget.go#L173-L183](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget.go#L173-L183)：

```go
func (w *widgetBase) requiresUpdate(now *time.Time) bool {
    if w.cacheType == cacheTypeInfinite {
        return false                          // 永不更新
    }
    if w.nextUpdate.IsZero() {
        return true                           // 首次请求，必须更新
    }
    return now.After(w.nextUpdate)            // 已过 nextUpdate 时间点
}
```

---

## 五、失败短缓存（指数退避重试）

当数据获取失败时，系统不会等待完整 TTL，而是采用**指数退避（Exponential Backoff）** 策略调度更短的重试间隔。

### 5.1 统一错误处理入口

所有 Widget 的 `update()` 方法都通过 `canContinueUpdateAfterHandlingErr(err)` 处理结果，定义于 [widget.go#L293-L325](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget.go#L293-L325)。

错误分两个等级（定义于 [widget-utils.go#L20-L21](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-utils.go#L20-L21)）：

| 错误类型 | 含义 | 处理方式 |
|---------|------|---------|
| `errNoContent` | 完全获取失败 | 设置 `widget.Error`，显示错误，**不保留旧内容** |
| `errPartialContent` | 部分资源失败 | 设置 `widget.Notice`，**保留已获取内容**继续渲染 |

### 5.2 scheduleEarlyUpdate 退避算法

[widget.go#L350-L367](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget.go#L350-L367)：

```go
func (w *widgetBase) scheduleEarlyUpdate() *widgetBase {
    w.updateRetriedTimes++
    if w.updateRetriedTimes > 5 {
        w.updateRetriedTimes = 5              // 最大退避阶数：5
    }

    // 退避间隔 = retryCount^2 分钟
    nextEarlyUpdate := time.Now().Add(
        time.Duration(math.Pow(float64(w.updateRetriedTimes), 2)) * time.Minute,
    )
    nextUsualUpdate := w.getNextUpdateTime()

    // 取"更早"的那个时间点（退避间隔不会超过正常 TTL）
    if nextEarlyUpdate.After(nextUsualUpdate) {
        w.nextUpdate = nextUsualUpdate
    } else {
        w.nextUpdate = nextEarlyUpdate
    }
    return w
}
```

重试间隔序列：

| 失败次数 | 退避间隔 | 累计等待 |
|---------|---------|---------|
| 第 1 次 | 1² = 1 分钟 | 1 分钟 |
| 第 2 次 | 2² = 4 分钟 | 5 分钟 |
| 第 3 次 | 3² = 9 分钟 | 14 分钟 |
| 第 4 次 | 4² = 16 分钟 | 30 分钟 |
| 第 5 次+ | 5² = 25 分钟 | 封顶 25 分钟 |

成功后通过 `scheduleNextUpdate()` [widget.go#L343-L348](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget.go#L343-L348) 将 `updateRetriedTimes` 重置为 0。

---

## 六、击穿保护（Cache Stampede Protection）

缓存击穿指**热点 key 过期瞬间，大量并发请求同时穿透到后端数据源**的现象。Glance 通过两层机制防护。

### 6.1 第一层：Page 级互斥锁

在 `handlePageContentRequest()` 中获取 `page.mu`，确保**同一时刻同一页面只有一个请求在执行刷新逻辑**，其他请求被阻塞等待。

但这是粗粒度锁，锁定范围包含了模板渲染，防护精度为页面级别。

### 6.2 第二层：Singleflight 单飞模式

对于跨 Widget 共享的昂贵资源（如 Reddit 的 loid cookie），使用 `Singleflight` 机制。

实现于 [singleflight.go](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/singleflight.go)：

```go
func (s *Singleflight[T]) Do() (T, error) {
    s.mu.Lock()
    if s.current != nil {
        // 已有正在进行的调用，等待其结果
        c := s.current
        s.mu.Unlock()
        <-c.done                    // 阻塞等待 channel 关闭
        return c.val, c.err
    }
    // 第一个请求，发起实际调用
    c := &SingleflightCall[T]{done: make(chan struct{})}
    s.current = c
    s.mu.Unlock()

    c.val, c.err = s.fn()           // 执行实际函数

    s.mu.Lock()
    s.current = nil
    s.mu.Unlock()
    close(c.done)                   // 广播结果
    return c.val, c.err
}
```

典型应用：Reddit loid cookie 的获取 [widget-reddit.go#L362-L387](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-reddit.go#L362-L387)：

- 全局只有一个 `getRedditLoidCookie` 实例（通过 IIFE 闭包创建）
- 并发的多个 Reddit Widget 请求只会触发一次实际的 HTTP 挑战流程
- 其余请求等待第一个请求的结果
- 叠加了 6 小时本地缓存（`lastUpdate` + `cachedLoid`）

### 6.3 第三层：HTTP 条件请求

RSS Widget 利用标准 HTTP 缓存机制减轻源站压力 [widget-rss.go#L200-L224](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-rss.go#L200-L224)：

- 请求时带上 `If-None-Match`（ETag）和 `If-Modified-Since`
- 服务端返回 `304 Not Modified` 时直接复用 `cache.items`
- 避免重复解析相同内容

---

## 七、协作时序图（以 RSS Widget 为例）

```
前端轮询 /api/pages/{page}/content
        │
        ▼
handlePageContentRequest
        │  获取 page.mu
        ▼
page.updateOutdatedWidgets
        │
        ├─ widget.requiresUpdate(now)
        │     └─ now > nextUpdate? ──否──► 跳过，用缓存内容
        │                       
        ▼ 是
   启动 goroutine: rssWidget.update()
        │
        ▼
fetchItemsFromFeeds() ── workerPoolDo(30 workers) ──► 并发 fetchItemsFromFeedTask()
        │                                                  │
        │                                                  ├─ 查 cachedFeeds[url]
        │                                                  ├─ 带 If-None-Match / If-Modified-Since 发请求
        │                                                  ├─ 304? ──是──► 返回 cache.items
        │                                                  └─ 200? ──是──► 解析并更新 cachedFeeds[url]
        │
        ▼
canContinueUpdateAfterHandlingErr(err)
        │
        ├─ err == nil ──────────────────► scheduleNextUpdate()  → nextUpdate = now + 2h, retried=0
        ├─ errors.Is(err, errPartialContent) → scheduleEarlyUpdate() → nextUpdate = now + 1²~25min, 设置 Notice
        └─ errors.Is(err, errNoContent)    → scheduleEarlyUpdate() → nextUpdate = now + 1²~25min, 设置 Error
        │
        ▼
渲染模板，返回响应，释放 page.mu
```

---

## 八、关键文件速查表

| 文件 | 核心职责 |
|------|---------|
| [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget.go) | Widget 基类、缓存类型、TTL 计算、退避调度、错误处理 |
| [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/glance.go) | HTTP 入口、page 级锁、`updateOutdatedWidgets()` 调度器 |
| [singleflight.go](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/singleflight.go) | 泛型 Singleflight 实现，防击穿 |
| [widget-container.go](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-container.go) | 容器 Widget 的递归更新逻辑 |
| [widget-utils.go](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-utils.go) | 错误定义（`errNoContent` / `errPartialContent`）、Worker Pool |
| [config.go](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/config.go) | `page.mu` 定义、Widget 初始化 |
| [widget-rss.go](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-rss.go) | ETag/Last-Modified 二级缓存示例 |
| [widget-reddit.go](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-reddit.go) | Singleflight + 本地缓存组合示例 |
| [widget-weather.go](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-weather.go) | OnTheHour 整点缓存示例 |
