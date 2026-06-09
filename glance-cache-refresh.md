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
              ├─ 成功(err=nil) → withError(nil) 置 ContentAvailable=true, scheduleNextUpdate(), return true → 覆盖数据字段
              ├─ 部分成功(errPartialContent) → withNotice(err), scheduleEarlyUpdate(), return true → 覆盖部分数据
              └─ 完全失败(errNoContent) → withError(err) 不碰 ContentAvailable, scheduleEarlyUpdate(), return false → 不覆盖数据(保留旧值)
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

## 五、失败短缓存、内容保留与错误提示的协作

这是整个缓存系统最精巧、最容易误解的部分。三个核心机制互相配合：
1. `withError()` / `withNotice()` — 设置状态标记
2. `canContinueUpdateAfterHandlingErr()` 的返回值 — 控制是否覆盖旧数据
3. `widget-base.html` 模板 — 根据状态决定渲染方式

### 5.1 三个关键状态字段

每个 Widget 实例维护三个状态位（定义于 [widget.go#L149-L167](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget.go#L149-L167)）：

| 字段 | 类型 | 含义 |
|------|------|------|
| `ContentAvailable` | `bool` | 是否有**可展示的内容**。这是决定渲染"内容"还是"错误页"的总开关 |
| `Error` | `error` | 严重错误。仅当 `ContentAvailable=true` 时以图标形式展示 |
| `Notice` | `error` | 轻微提示（部分数据缺失）。以黄色小图标展示 |

### 5.2 withError 的微妙逻辑

`withError()` 定义于 [widget.go#L283-L291](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget.go#L283-L291)：

```go
func (w *widgetBase) withError(err error) *widgetBase {
    if err == nil && !w.ContentAvailable {
        w.ContentAvailable = true        // ← 只有"成功且之前没内容"时才置为 true
    }
    w.Error = err                        // ← err != nil 时只赋值 Error，不动 ContentAvailable
    return w
}
```

**关键洞察**：当 `err != nil` 时，`withError()` **绝对不会修改 `ContentAvailable`**。它既不会把 `true` 改成 `false`，也不会把 `false` 改成 `true`。这是"旧内容不被丢弃"的第一道防线。

### 5.3 canContinueUpdateAfterHandlingErr 的返回值决定数据是否被覆盖

完整逻辑在 [widget.go#L293-L325](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget.go#L293-L325)：

```go
func (w *widgetBase) canContinueUpdateAfterHandlingErr(err error) bool {
    if err != nil {
        w.scheduleEarlyUpdate()                    // ① 无论何种失败，都触发退避重试

        if !errors.Is(err, errPartialContent) {
            w.withError(err)                       // ② 设置 Error（不碰 ContentAvailable）
            w.withNotice(nil)
            return false                           // ③ ⚠ 返回 false：调用方不会覆盖数据字段
        }

        // errPartialContent 分支
        w.withError(nil)                           // ②' 清 Error，如无内容则把 ContentAvailable 置 true
        w.withNotice(err)                          // ②' 设置 Notice 提示
        return true                                // ③' ⚠ 返回 true：调用方会用部分数据覆盖字段
    }

    // 成功分支
    w.withNotice(nil)
    w.withError(nil)                               // 清除 Error，如无内容则把 ContentAvailable 置 true
    w.scheduleNextUpdate()                         // 正常 TTL，重置退避计数
    return true
}
```

`return false` 的真实含义是：**中止本次 update，不要把新获取的（可能为空的）数据写入 Widget 字段**。因为一旦 `widget.Videos = nil` 或 `widget.Posts = nil` 执行了，之前的旧数据就被覆盖丢失了。

看一个典型 Widget 调用（以 videos 为例，[widget-videos.go#L66-L78](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-videos.go#L66-L78)）：

```go
func (widget *videosWidget) update(ctx context.Context) {
    videos, err := fetchYoutubeChannelUploads(...)

    if !widget.canContinueUpdateAfterHandlingErr(err) {
        return                                    // ← 提前 return，下一行不会执行
    }                                             //    旧的 widget.Videos 原封不动

    if len(videos) > widget.Limit {
        videos = videos[:widget.Limit]
    }
    widget.Videos = videos                        // ← 只有成功/部分成功才走到这里覆盖数据
}
```

### 5.4 模板渲染：两种错误呈现

模板 [widget-base.html](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/templates/widget-base.html) 用两种截然不同的方式展示错误：

**分支 A：`ContentAvailable = true`（有旧内容可展示）**

```html
{{- if and .Error .ContentAvailable }}
<div class="notice-icon notice-icon-major" title="{{ .Error }}"></div>  <!-- 右上角红色小图标 -->
{{- else if .Notice }}
<div class="notice-icon notice-icon-minor" title="{{ .Notice }}"></div> <!-- 右上角黄色小图标 -->
{{- end }}
...
{{- if .ContentAvailable }}
{{ block "widget-content" . }}{{ end }}      <!-- ← 正常渲染旧内容 -->
```

效果：内容区域照常显示上一次成功的数据，Header 右上角多一个 ⚠️ 图标，鼠标悬浮可看错误信息。用户感知是"数据有点旧，但还能用"。

**分支 B：`ContentAvailable = false`（从未成功过，无内容可展示）**

```html
{{- else }}
    <div class="widget-error-header">
        <div class="color-negative size-h3">ERROR</div>
        <svg class="widget-error-icon">...</svg>
    </div>
    <p class="break-all">{{ if .Error }}{{ .Error }}{{ else }}No error information provided{{ end }}</p>
{{- end}}
```

效果：整个 Widget 区域渲染为一个大错误面板，带红色大图标和完整错误文本。

### 5.5 三种失败场景对比

下面用状态机形式说明各字段如何变化：

#### 场景一：首次加载完全失败（ContentAvailable=false）

| 步骤 | 操作 | ContentAvailable | Error | Notice | 数据字段 | nextUpdate |
|------|------|-----------------|-------|--------|---------|------------|
| 初始 | - | false | nil | nil | nil | 零值 |
| fetch 返回 errNoContent | - | false | nil | nil | nil | 零值 |
| scheduleEarlyUpdate | - | false | nil | nil | nil | now + 1min |
| withError(err) | 只赋值 Error | **false** | err | nil | nil | now + 1min |
| return false | 不覆盖数据 | false | err | nil | **nil（保持）** | now + 1min |
| 渲染 | - | false | err | nil | nil | now + 1min |

**呈现**：大 ERROR 页面，完整错误信息。

#### 场景二：已有内容，后续刷新完全失败（ContentAvailable=true）

| 步骤 | 操作 | ContentAvailable | Error | Notice | 数据字段 | nextUpdate |
|------|------|-----------------|-------|--------|---------|------------|
| 初始 | 上次成功 | **true** | nil | nil | [旧视频列表] | now+1h |
| fetch 返回 errNoContent | - | true | nil | nil | [旧视频列表] | now+1h |
| scheduleEarlyUpdate | - | true | nil | nil | [旧视频列表] | now+1min |
| withError(err) | 只赋值 Error，不动 ContentAvailable | **true** | err | nil | [旧视频列表] | now+1min |
| return false | 不覆盖数据 | true | err | nil | **[旧视频列表]（保持不变）** | now+1min |
| 渲染 | - | true | err | nil | [旧视频列表] | now+1min |

**呈现**：正常渲染旧的视频列表内容，Header 右上角红色小图标显示错误。

#### 场景三：部分失败（errPartialContent，如 RSS 10 个源有 2 个失败）

| 步骤 | 操作 | ContentAvailable | Error | Notice | 数据字段 | nextUpdate |
|------|------|-----------------|-------|--------|---------|------------|
| 初始 | 首次/后续 | false 或 true | nil | nil | nil/旧数据 | - |
| fetch 返回 8 条成功 + errPartialContent | - | - | nil | nil | [部分数据] | - |
| scheduleEarlyUpdate | - | - | nil | nil | [部分数据] | now+1min |
| withError(nil) | 清除 Error，如之前无内容则置 ContentAvailable=true | **true** | nil | nil | [部分数据] | now+1min |
| withNotice(err) | 设置 Notice | true | nil | err | [部分数据] | now+1min |
| return true | 允许覆盖数据 | true | nil | err | **[部分数据（覆盖旧值）]** | now+1min |

**呈现**：渲染已成功获取的 8 条内容，Header 右上角黄色小图标提示"部分源失败"。

### 5.6 scheduleEarlyUpdate 退避算法

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

| 连续失败次数 | 退避间隔 | 说明 |
|------------|---------|------|
| 第 1 次 | 1² = 1 分钟 | 快速重试 |
| 第 2 次 | 2² = 4 分钟 | - |
| 第 3 次 | 3² = 9 分钟 | - |
| 第 4 次 | 4² = 16 分钟 | - |
| 第 5 次及以后 | 5² = 25 分钟（封顶） | 不再继续拉长 |

注意：退避时间与正常 TTL 取较小值。例如一个 TTL=5 分钟的 Widget，第 3 次失败后不会等 9 分钟，而是等正常的 5 分钟。

成功后通过 `scheduleNextUpdate()` [widget.go#L343-L348](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget.go#L343-L348) 将 `updateRetriedTimes` 重置为 0。

### 5.7 对重试边界的影响

上述机制直接影响"何时会再次触发 update"，形成了以下行为边界：

**边界 1：首次加载永远重试激进**
- 首次失败时 `nextUpdate.IsZero()` 为 true → `requiresUpdate()` 直接返回 true
- 但一旦调用了 `scheduleEarlyUpdate()`，`nextUpdate` 就被设为 now+1min，之后 1 分钟内即使有新请求也不会重试
- 配合 `page.mu`，同一页面不会有两个并发刷新同时进行

**边界 2：有旧内容时，失败不会引发更激进的重试（和无内容时退避完全一样）**
- 无论 `ContentAvailable` 是 true 还是 false，指数退避的公式完全相同
- 有旧内容只是**展示层**降级（显示旧数据），**调度层**不受优待或歧视
- 这意味着：用户看到旧内容的同时，后台在按 1→4→9→16→25 分钟的节奏默默重试

**边界 3：部分成功会中断"连续失败"计数吗？不会**
- `errPartialContent` 同样走 `scheduleEarlyUpdate()`，退避计数仍然累加
- 只有**完全成功**（`err == nil`）时才会调用 `scheduleNextUpdate()` 将 `updateRetriedTimes` 归零
- 这是一个保守策略：只要不是完美成功，就继续保持警惕

**边界 4：return false 保护了数据，但也意味着 Widget 的业务字段永远不会被"清空"**
- 一旦某个 Widget 曾经成功过（`ContentAvailable=true`），即使 API 持续返回失败，它的 `widget.Posts`、`widget.Videos` 等字段将永久保留最后一次成功的数据
- 只有重启进程（内存丢失）或调用方手动置空才会清除
- 这属于"宁可显示旧数据也不显示空白"的设计哲学

### 5.8 错误提示展示边界：HideHeader、容器与静默失败

前面讨论的是**调度层**（何时重试）和**数据层**（是否保留旧数据），但**展示层**（用户能否看到错误提示图标）还受 `HideHeader` 和容器类型的制约，三者互相独立。

#### 5.8.1 图标的 DOM 归属：notice-icon 位于 header 内部

回顾 [widget-base.html#L2-L27](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/templates/widget-base.html#L2-L27)：

```html
{{- if not .HideHeader }}
<div class="widget-header">
    <h2>...</h2>
    {{- if and .Error .ContentAvailable }}
    <div class="notice-icon notice-icon-major" title="{{ .Error }}"></div>
    {{- else if .Notice }}
    <div class="notice-icon notice-icon-minor" title="{{ .Notice }}"></div>
    {{- end }}
</div>
{{- end }}
```

**关键结论**：`.notice-icon`（红/黄圆点）是 `<div class="widget-header">` 的子元素，整个 header 被 `{{- if not .HideHeader }}` 整块包裹。一旦 `HideHeader=true`，标题栏、错误图标、提示图标**一起整体消失**。

但大 ERROR 面板（`ContentAvailable=false` 时渲染）不在 header 内，它在 [widget-base.html#L28-L40](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/templates/widget-base.html#L28-L40) 的 `widget-content` 分支里，**不受 HideHeader 影响**。

#### 5.8.2 普通 Widget（非容器）在 HideHeader 下的表现

| 场景 | HideHeader | ContentAvailable | Error/Notice 状态 | 用户看到什么 |
|------|-----------|-----------------|-------------------|------------|
| A | false | true | Error=有 | 标题 + 内容 + **红色小圆点**（tooltip 看详情） |
| B | false | true | Notice=有 | 标题 + 内容 + **黄色小圆点** |
| C | false | true | 都无 | 标题 + 内容（正常） |
| D | false | false | Error=有 | 标题 + **大 ERROR 面板**（含完整错误信息） |
| E | **true** | true | Error=有 | **只显示内容（静默失败）**，无任何图标 |
| F | **true** | true | Notice=有 | **只显示内容（静默提示）**，无任何图标 |
| G | **true** | false | Error=有 | **大 ERROR 面板**（不受 HideHeader 影响） |

**场景 E/F 是"隐形失败"**：调度层在按指数退避默默重试、数据层保留了旧内容，但用户界面上**没有任何视觉线索**表明内容可能已过期。

#### 5.8.3 Group 容器：双层 HideHeader 导致图标彻底消失

Group 的初始化逻辑在 [widget-group.go#L17-L36](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-group.go#L17-L36)：

```go
func (widget *groupWidget) initialize() error {
    widget.withError(nil)
    widget.HideHeader = true                              // ① 容器自身隐藏 header

    for i := range widget.Widgets {
        widget.Widgets[i].setHideHeader(true)              // ② 强制所有子 widget 也隐藏 header
        ...
    }
    ...
}
```

Group 模板 [group.html](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/templates/group.html) 用自定义的 tab 栏替代了普通 header：

```html
<div class="widget-group-header">
    <div class="widget-header gap-20" role="tablist">
        {{- range $i, $widget := .Widgets }}
        <button class="widget-group-title...">{{ $widget.Title }}</button>
        {{- end }}
    </div>
</div>
```

这个自定义 tab header 中**完全没有 notice-icon 的渲染逻辑**。再加上子 widget 的 header 被集体隐藏，导致：

| 子 Widget 状态 | 用户看到什么 |
|--------------|------------|
| ContentAvailable=true + Error | tab 标题正常 + 旧内容正常渲染，**无任何图标**（静默失败） |
| ContentAvailable=true + Notice | 同上，无任何图标 |
| ContentAvailable=false + Error | 该 tab 页内显示**大 ERROR 面板**（子 widget 的 widget-content 不受 header 影响） |

**Group 容器下没有"小圆点"这种轻度提示**，要么完全静默（有旧内容时），要么整个 tab 内容区变成大 ERROR 面板（首次失败时）。

另外注意：容器自身的 `Error` / `Notice` 字段永远为 nil——`containerWidgetBase._update()` [widget-container.go#L23-L42](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-container.go#L23-L42) 只是并发调用子 widget 的 `update()`，**没有任何错误聚合逻辑**。容器不会把"第 3 个子 widget 失败了"反映到自己的 Error 字段上。

#### 5.8.4 Split-column 容器：只隐藏自身 header，子 widget 保留图标

Split-column 的初始化在 [widget-split-column.go#L17-L29](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-split-column.go#L17-L29)：

```go
func (widget *splitColumnWidget) initialize() error {
    widget.withError(nil).withTitle("Split Column").setHideHeader(true)  // 只隐藏容器自身
    // 没有对子 widget 调用 setHideHeader(true)！
    ...
}
```

模板 [split-column.html](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/templates/split-column.html) 用 masonry 布局直接渲染每个子 widget：

```html
<div class="masonry" data-max-columns="{{ .MaxColumns }}">
{{ range .Widgets }}
    {{ .Render }}   <!-- 每个子 widget 完整渲染自己的 widget-base.html -->
{{ end }}
</div>
```

因此子 widget 的行为和独立放置时完全一致：如果子 widget 自身没有设置 `hide-header: true`，它的 header 和错误图标都会正常显示。

| 子 Widget 状态 | HideHeader（子 widget 自身） | 用户看到什么 |
|--------------|---------------------------|------------|
| ContentAvailable=true + Error | false（默认） | 子 widget 完整 header + **红色小圆点** + 内容 |
| ContentAvailable=true + Error | true（用户手动配置） | 只显示内容，静默失败 |
| ContentAvailable=false + Error | 任意 | 该子 widget 区域显示大 ERROR 面板 |

Split-column 容器自身的 header 虽然被隐藏了，但容器自身没有 Error/Notice 状态（无聚合），所以没有影响。

#### 5.8.5 展示层与短缓存调度层的对应关系

将展示层的可见性与前面的调度/数据层机制对照：

| 组合场景 | 调度层（nextUpdate） | 数据层（ContentAvailable） | 展示层（用户可见） | 失败可感知吗？ |
|---------|-------------------|-------------------------|------------------|--------------|
| 普通 Widget 刷新失败，有旧内容，HideHeader=false | 1→4→9→16→25 min 退避 | true，保留旧数据 | 内容 + 红色小圆点 | ✅ 轻度感知 |
| 普通 Widget 刷新失败，有旧内容，HideHeader=true | 1→4→9→16→25 min 退避 | true，保留旧数据 | **只显示旧内容** | ❌ 完全不可感知 |
| Group 中子 Widget 刷新失败，有旧内容 | 1→4→9→16→25 min 退避 | true，保留旧数据 | **只显示旧内容** | ❌ 完全不可感知 |
| Split-column 中子 Widget 刷新失败，有旧内容，子 header 未隐藏 | 1→4→9→16→25 min 退避 | true，保留旧数据 | 子 widget header + 红色小圆点 + 内容 | ✅ 轻度感知 |
| 任何 Widget 首次加载失败 | 1→4→9→16→25 min 退避 | false，无数据 | **大 ERROR 面板** | ✅ 强感知 |
| 任何 Widget 部分失败（errPartialContent） | 1→4→9→16→25 min 退避 | true，部分数据被覆盖 | 内容 + 黄色小圆点（或静默，取决于 HideHeader） | ⚠️ 条件感知 |

**核心洞察**：
1. **调度层完全独立于展示层**：无论用户能否看到图标，后台重试节奏都是 1→4→9→16→25 分钟。HideHeader 和容器类型不影响 `scheduleEarlyUpdate()` 的调用。
2. **"有旧内容 + HideHeader=true" 或 "子 widget 在 Group 内" 是静默失败的两大来源**：用户看着正常内容，不知道 API 可能已经挂了几小时，后台在默默重试。
3. **大 ERROR 面板是唯一不受 HideHeader 影响的错误提示**——当 `ContentAvailable=false` 时，错误信息直接渲染在内容区，没有任何开关可以隐藏它。这保证了"首次加载就失败"的场景永远不会静默。
4. **容器没有错误聚合**：Group/Split-column 不会统计有多少个子 widget 失败了并展示汇总提示。每个子 widget 的错误状态只能由自己独立呈现（如果 header 可见）。

### 5.9 Extension Widget：不遵循通用模式的例外路径

Extension Widget 的异常处理在 [widget-extension.go#L45-L67](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-extension.go#L45-L67)，是全系统中唯一偏离"提前 return 保住旧内容"模式的实现。下面严格区分**代码可直接证实的实现现象**和**无直接证据、仅能谨慎推测的设计解释**。

---

#### 5.9.1 代码可直接证实的实现现象

以下每一条都有明确代码行作为证据，不含推测。

##### 现象 1：`canContinueUpdateAfterHandlingErr` 返回值被丢弃，后续代码无条件执行

通用模式（videos 为例，[widget-videos.go#L66-L78](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-videos.go#L66-L78)）：

```go
if !widget.canContinueUpdateAfterHandlingErr(err) {
    return          // 失败时中止，后续赋值不执行
}
widget.Videos = videos
```

Extension 模式 [widget-extension.go#L54-L66](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-extension.go#L54-L66)：

```go
widget.canContinueUpdateAfterHandlingErr(err)   // 返回值未赋值给任何变量，等价于丢弃

widget.Extension = extension                     // 后续三行无条件执行
widget.Title = ...
widget.cachedHTML = ...
```

`canContinueUpdateAfterHandlingErr` 的 side effects 仍然生效（`scheduleEarlyUpdate()` 设置 nextUpdate、`withError(err)` 设置 Error 字段），但 `return false` 的中止语义被忽略。

##### 现象 2：失败时 4 个字段的保留/覆盖行为各不相同

`update()` 方法中涉及 4 个字段的修改，它们的条件完全不同，下面逐一列出来源代码。

**字段 A：`widget.Extension`（整体）**

代码位置 [widget-extension.go#L56](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-extension.go#L56)：

```go
widget.Extension = extension
```

这是无任何条件的直接赋值。失败时 `fetchExtension` 返回的 `extension` 是零值结构体 [widget-extension.go#L99-L104](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-extension.go#L99-L104)：

```go
type extension struct {
    Title     string       // 零值 = ""
    TitleURL  string       // 零值 = ""
    Content   template.HTML // 零值 = template.HTML("")
    Frameless bool         // 零值 = false
}
```

结论：`widget.Extension` **无论成功失败都被覆盖**，失败时被零值覆盖，旧内容永久丢失。

---

**字段 B：`widget.Title`**

代码位置 [widget-extension.go#L58-L60](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-extension.go#L58-L60)：

```go
if widget.Title == extensionWidgetDefaultTitle && extension.Title != "" {
    widget.Title = extension.Title
}
```

`extensionWidgetDefaultTitle = "Extension"`，在初始化时通过 `widget.withTitle(extensionWidgetDefaultTitle)` 设置 [widget-extension.go#L32](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-extension.go#L32)。

更新 Title **必须同时满足两个条件**：
1. `widget.Title == "Extension"`（当前 Title 仍是默认值，从未被远端覆盖过）
2. `extension.Title != ""`（本次返回的 extension.Title 非空）

失败时 `extension.Title = ""`（零值），条件 2 恒为 false → **Title 保持原值不变**。

成功时 `extension.Title` 的值取决于远端响应头 [widget-extension.go#L145-L149](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-extension.go#L145-L149)：
- 远端未传 `Widget-Title` header → `extension.Title = "Extension"`（注意：不是空串，是默认值字符串）
- 远端传了 header → `extension.Title = 远端指定值`

结论：
- 失败时：**Title 永远保留原值**（不覆盖）
- 成功且 Title 曾被远端自定义过（`widget.Title != "Extension"`）：**保留原值**
- 成功且 Title 仍是默认值、远端传了自定义 header：**更新为远端标题**
- 成功且 Title 仍是默认值、远端未传 header：条件看似满足但赋值为相同字符串，无实际变化

---

**字段 C：`widget.TitleURL`**

代码位置 [widget-extension.go#L62-L64](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-extension.go#L62-L64)：

```go
if widget.TitleURL == "" && extension.TitleURL != "" {
    widget.TitleURL = extension.TitleURL
}
```

更新 TitleURL **必须同时满足两个条件**：
1. `widget.TitleURL == ""`（当前 TitleURL 为空，从未被设置过）
2. `extension.TitleURL != ""`（本次返回的 extension.TitleURL 非空）

失败时 `extension.TitleURL = ""`（零值），条件 2 恒为 false → **TitleURL 保持原值不变**。

成功时 `extension.TitleURL` 只有在远端传了 `Widget-Title-URL` header 时才非空 [widget-extension.go#L151-L153](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-extension.go#L151-L153)。

结论：
- 失败时：**TitleURL 永远保留原值**（不覆盖）
- 成功且 TitleURL 曾被设置过（非空）：**保留原值**
- 成功且 TitleURL 为空、远端传了 header：**更新为远端 URL**
- 成功且 TitleURL 为空、远端未传 header：**保持为空**

---

**字段 D：`widget.cachedHTML`**

代码位置 [widget-extension.go#L66](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-extension.go#L66)：

```go
widget.cachedHTML = widget.renderTemplate(widget, extensionWidgetTemplate)
```

这是无条件赋值。Extension 的 `Render()` [widget-extension.go#L69-L71](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-extension.go#L69-L71) 是全系统中**唯一返回预渲染缓存**的：

```go
func (widget *extensionWidget) Render() template.HTML {
    return widget.cachedHTML
}
```

其他 Widget（如 videos）的 `Render()` [widget-videos.go#L80-L93](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-videos.go#L80-L93) 每次都实时调用 `widget.renderTemplate()`。

失败时调用 `renderTemplate` 的时刻，各状态为：
- `Error = err`（由 `canContinueUpdateAfterHandlingErr` 设置）
- `ContentAvailable`：保持原值（`withError(err)` 不修改它）
- `Extension.Content = ""`（已被零值覆盖）
- `Extension.Title` / `Extension.TitleURL`：已被零值覆盖为 `""`

模板 [extension.html](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/templates/extension.html) 的内容块是 `{{ .Extension.Content }}`，即空字符串。

结论：`widget.cachedHTML` **无论成功失败都被覆盖重渲染**。失败时渲染结果取决于 `ContentAvailable`：
- `ContentAvailable = true`（之前成功过）：渲染出"空内容区域 + 红圆点图标"的 HTML
- `ContentAvailable = false`（首次失败）：渲染出大 ERROR 面板的 HTML

---

##### 现象 3：`fetchExtension` 不检查 HTTP 状态码

[fetchExtension()](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-extension.go#L119-L172) 的两个错误返回点：

```go
response, err := http.DefaultClient.Do(request)
if err != nil {
    return extension{}, fmt.Errorf("%w: request failed: %w", errNoContent, err)
}
// 代码中不存在 response.StatusCode / response.Status 的检查
body, err := io.ReadAll(response.Body)
if err != nil {
    return extension{}, fmt.Errorf("%w: could not read body: %w", errNoContent, err)
}
// 后续直接把 body 当作内容返回，err = nil
return extension, nil
```

结论：只有 TCP/TLS 连接失败、DNS 解析失败等**网络层错误**和**body 读取错误**才被视为失败。远端返回 HTTP 404/500/503 等状态时，只要 body 能读出字节，`err` 就为 `nil`，调度层走 `scheduleNextUpdate()` 正常 TTL 分支，退避计数被清零，错误页面的 HTML 被当作正常内容渲染。

##### 现象 4：`canContinueUpdateAfterHandlingErr` 自身承认尚未覆盖所有 edge case

该函数开头有一段 TODO 注释 [widget.go#L294-L305](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget.go#L294-L305)：

```go
// TODO: needs covering more edge cases.
// if there's partial content and we update early there's a chance
// the early update returns even less content than the initial update.
// ... will require reworking a good amount of code ...
// alternatively have a resource cache and only refetch the failed resources,
// then rebuild the widget.
```

结论：错误处理框架本身作者也标注为"未完成、需要重构"。Extension 的特殊模式是否属于有意识的设计决策，代码中没有直接证据。

---

#### 5.9.2 完整状态变化矩阵（所有字段，两种失败场景）

**场景 A：之前成功过（ContentAvailable=true，Title 曾被远端自定义为 "My Dashboard"，TitleURL 曾设为 "https://..."），现在网络失败**

| 步骤 | widget.Extension.Content | widget.Title | widget.TitleURL | widget.cachedHTML | ContentAvailable | Error | nextUpdate |
|------|--------------------------|-------------|-----------------|-------------------|-----------------|-------|------------|
| 初始 | `<p>上次成功内容</p>` | "My Dashboard" | "https://..." | 含正常内容的 HTML | true | nil | now+30min |
| fetchExtension 网络错误，返回 `(extension{}, err)` | `<p>上次成功内容</p>` | "My Dashboard" | "https://..." | 含正常内容的 HTML | true | nil | now+30min |
| `canContinueUpdateAfterHandlingErr(err)` | `<p>上次成功内容</p>` | "My Dashboard" | "https://..." | 含正常内容的 HTML | **true**（不变） | err | now+1min |
| `widget.Extension = extension`（无条件覆盖） | **""（空，丢失）** | "My Dashboard" | "https://..." | 含正常内容的 HTML | true | err | now+1min |
| `widget.Title = ...`（条件不满足：Title≠默认值） | "" | **"My Dashboard"（保留）** | "https://..." | 含正常内容的 HTML | true | err | now+1min |
| `widget.TitleURL = ...`（条件不满足：TitleURL≠空） | "" | "My Dashboard" | **"https://..."（保留）** | 含正常内容的 HTML | true | err | now+1min |
| `widget.cachedHTML = renderTemplate(...)`（无条件） | "" | "My Dashboard" | "https://..." | **空内容+红点的 HTML（覆盖）** | true | err | now+1min |
| 最终用户可见 | — | "My Dashboard" 显示在 header | 标题链接保留 | 空白框+红点 | — | tooltip 可见错误 | 1min 后重试 |

**场景 B：首次加载失败（ContentAvailable=false，Title="Extension" 默认值，TitleURL=""）**

| 步骤 | widget.Extension.Content | widget.Title | widget.TitleURL | widget.cachedHTML | ContentAvailable | Error | nextUpdate |
|------|--------------------------|-------------|-----------------|-------------------|-----------------|-------|------------|
| 初始 | "" | "Extension" | "" | "" | false | nil | 零值 |
| fetchExtension 网络错误 | "" | "Extension" | "" | "" | false | nil | 零值 |
| `canContinueUpdateAfterHandlingErr(err)` | "" | "Extension" | "" | "" | **false**（不变） | err | now+1min |
| `widget.Extension = extension` | ""（已是零值） | "Extension" | "" | "" | false | err | now+1min |
| `widget.Title = ...`（条件不满足：extension.Title=""） | "" | **"Extension"（保留）** | "" | "" | false | err | now+1min |
| `widget.TitleURL = ...`（条件不满足：extension.TitleURL=""） | "" | "Extension" | **""（保留）** | "" | false | err | now+1min |
| `widget.cachedHTML = renderTemplate(...)` | "" | "Extension" | "" | **大 ERROR 面板 HTML（覆盖）** | false | err | now+1min |
| 最终用户可见 | — | "Extension" 显示在 ERROR 面板上方 | 空（无链接） | 完整错误页 | — | 错误详情可见 | 1min 后重试 |

---

#### 5.9.3 仅能谨慎推测的设计解释

以下内容代码中无直接注释或文档佐证，属于基于现象的合理推断：

| 推测 | 支持的现象 | 反证/不确定性 |
|------|-----------|-------------|
| **推测 A：Extension 不保留旧内容是出于第三方 HTML 的安全考虑** | Extension.Content 是外部任意 HTML，旧内容可能含过期链接、失效 CSRF token、已移除的脚本。空白比"可能已坏掉的外部 HTML"更安全。 | 代码中没有任何安全相关注释。`withTitle`/`withTitleURL` 的"只写一次"模式暗示了对远端元数据的某种不信任，但这是对所有 Widget 的通用行为，非 Extension 独有。 |
| **推测 B：丢弃返回值是疏忽，而非刻意设计** | `canContinueUpdateAfterHandlingErr` 开头有 TODO 注释承认"需要覆盖更多 edge case、需要重构大量代码"；全系统 28 个 Widget 中只有 Extension 1 个不检查返回值，模式极不统一。 | 也可能作者在写 Extension 时确实想要"无论如何都重新渲染 cachedHTML"，所以故意绕过了提前 return。没有代码注释能区分这两种可能。 |
| **推测 C：cachedHTML 预渲染是性能优化** | Extension 内容是完整 HTML 片段（可能较大），每次页面请求都执行模板引擎有成本，update 时预渲染一次可节省后续渲染开销。 | 其他 Widget（如 RSS、Reddit）也渲染大量 HTML，但都采用实时渲染模式，没有类似优化。没有性能测试或注释支持此推测。 |
| **推测 D：不检查 HTTP 状态码是为了灵活性** | Extension 的设计目标是对接任意第三方服务，有些服务可能用非 2xx 状态码返回合法业务内容（如 207 Multi-Status、自定义业务码），不检查状态码留给远端自行决定内容语义。 | 大多数 HTTP 生态中 4xx/5xx 表示错误；RSS Widget 等有明确检查状态码的代码（需要验证），唯独 Extension 没有，也可能是遗漏。 |

---

#### 5.9.4 通用模式 vs Extension 模式对照表（仅含可证实部分）

| 维度 | 通用模式（videos、rss、weather 等） | Extension 模式 |
|------|-----------------------------------|---------------|
| `canContinueUpdateAfterHandlingErr` 返回值 | `if !xxx { return }` 严格检查 | 丢弃，不影响后续执行 |
| 失败时主数据字段（Videos/Posts/Extension 等） | 原封不动保留旧值 | 被零值无条件覆盖 |
| 失败时 `widget.Title` | —（通用 Widget 不在 update 中改 Title） | 保留原值（条件不满足） |
| 失败时 `widget.TitleURL` | —（通用 Widget 不在 update 中改 TitleURL） | 保留原值（条件不满足） |
| 失败时 `ContentAvailable` | `withError(err)` 不修改，保持原值 | 同左（保持原值） |
| 失败时有旧内容用户看到什么 | 旧内容原样显示 + 红/黄圆点 | 空白内容区域 + 红圆点 |
| 失败时首次加载用户看到什么 | 大 ERROR 面板 | 大 ERROR 面板 |
| `Render()` 实现 | 每次页面请求实时 `renderTemplate` | 返回 `update()` 时预渲染的 `cachedHTML` |
| HTTP 非 2xx 处理 | 各 Widget 通常自行检查并判为失败 | 不检查，错误响应 body 被当作正常内容，`err=nil` 走成功调度路径 |

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
        ├─ err == nil
        │     ├─ withError(nil): 如 ContentAvailable=false 则置为 true
        │     ├─ scheduleNextUpdate(): nextUpdate = now + 2h, retried=0
        │     └─ return true → 后续赋值 widget.Items = 新数据
        │
        ├─ errors.Is(err, errPartialContent)
        │     ├─ scheduleEarlyUpdate(): nextUpdate = now + retryCount² min
        │     ├─ withError(nil) + withNotice(err)
        │     └─ return true → 后续赋值 widget.Items = 部分数据
        │
        └─ errors.Is(err, errNoContent)
              ├─ scheduleEarlyUpdate(): nextUpdate = now + retryCount² min
              ├─ withError(err): 只赋值 Error, 不碰 ContentAvailable
              └─ return false → 提前 return, widget.Items 保持旧值不变
        │
        ▼
widget-base.html 模板渲染
        │
        ├─ ContentAvailable=true  → 渲染内容 + Header 图标(Error=红/Notice=黄)
        └─ ContentAvailable=false → 渲染大 ERROR 面板
        │
        ▼
返回响应，释放 page.mu
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
| [widget-base.html](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/templates/widget-base.html) | 模板层错误/Notice 两种呈现逻辑，notice-icon 归属 header |
| [widget-videos.go](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-videos.go) | 典型 update() 模式：return false 阻止覆盖旧数据 |
| [widget-group.go](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-group.go) | Group 容器：双层 HideHeader，强制子 widget 隐藏 header |
| [group.html](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/templates/group.html) | Group 模板：自定义 tab 栏无 notice-icon 渲染 |
| [widget-split-column.go](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-split-column.go) | Split-column 容器：仅隐藏自身 header，子 widget 保留图标 |
| [split-column.html](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/templates/split-column.html) | Split-column 模板：masonry 布局直接渲染子 widget |
| [site.css](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/static/css/site.css#L195-L207) | notice-icon 样式：major 红色实心 / minor 黄色空心边框 |
| [widget-extension.go](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/widget-extension.go) | Extension Widget：全系统唯一偏离通用模式的实现，丢弃返回值、零值覆盖、预渲染缓存 |
| [extension.html](file:///d:/fz/0601/solo-dogfeeding/code/147-glance/internal/glance/templates/extension.html) | Extension 模板：直接输出 `.Extension.Content`，失败时渲染空内容 |
