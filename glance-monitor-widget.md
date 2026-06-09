# Glance Monitor Widget 探测流程分析

## 概述

Monitor Widget 是 Glance 提供的站点可用性监控组件，通过并发 HTTP 探测来检查多个目标站点的在线状态、响应时间和 TLS 证书有效性，并以可视化方式呈现。核心代码分布在 [widget-monitor.go](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go)、[widget-utils.go](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-utils.go)、[widget.go](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget.go) 以及 [config-fields.go](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/config-fields.go)。

---

## 一、数据结构

### 1.1 主组件结构

```go
type monitorWidget struct {
    widgetBase `yaml:",inline"`
    Sites      []struct {
        *SiteStatusRequest `yaml:",inline"`
        Status             *siteStatus     `yaml:"-"`
        URL                string          `yaml:"-"`
        ErrorURL           string          `yaml:"error-url"`
        Title              string          `yaml:"title"`
        Icon               customIconField `yaml:"icon"`
        SameTab            bool            `yaml:"same-tab"`
        StatusText         string          `yaml:"-"`
        StatusStyle        string          `yaml:"-"`
        AltStatusCodes     []int           `yaml:"alt-status-codes"`
    } `yaml:"sites"`
    Style           string `yaml:"style"`
    ShowFailingOnly bool   `yaml:"show-failing-only"`
    HasFailing      bool   `yaml:"-"`
}
```
参考 [widget-monitor.go#L18-L35](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L18-L35)

关键字段说明：
- `SiteStatusRequest`：内嵌请求配置（URL、超时、TLS 等）
- `Status`：运行时探测结果（运行时字段，不参与 YAML 序列化）
- `ErrorURL`：站点失败时跳转的备用 URL
- `AltStatusCodes`：可被视为"正常"的额外 HTTP 状态码列表
- `ShowFailingOnly`：仅展示异常站点的开关
- `HasFailing`：标记是否存在异常站点

### 1.2 请求配置

```go
type SiteStatusRequest struct {
    DefaultURL    string        `yaml:"url"`
    CheckURL      string        `yaml:"check-url"`
    AllowInsecure bool          `yaml:"allow-insecure"`
    Timeout       durationField `yaml:"timeout"`
    BasicAuth     struct {
        Username string `yaml:"username"`
        Password string `yaml:"password"`
    } `yaml:"basic-auth"`
}
```
参考 [widget-monitor.go#L117-L126](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L117-L126)

### 1.3 探测结果

```go
type siteStatus struct {
    Code         int
    TimedOut     bool
    ResponseTime time.Duration
    Error        error
}
```
参考 [widget-monitor.go#L128-L133](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L128-L133)

---

## 二、初始化与缓存策略

### 2.1 初始化

`initialize()` 在组件创建时被调用，设置默认标题为 "Monitor"，缓存周期为 **5 分钟**：

```go
func (widget *monitorWidget) initialize() error {
    widget.withTitle("Monitor").withCacheDuration(5 * time.Minute)
    return nil
}
```
参考 [widget-monitor.go#L37-L41](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L37-L41)

### 2.2 刷新调度机制总览

刷新调度是整个 Widget 体系中最容易产生误解的部分。整个调度链路横跨三层：HTTP 请求层 → Page 层 → Widget 层。

#### 2.2.1 整体调度框架

**触发入口**：浏览器加载页面时会请求 `/api/pages/{page}/content/`，由 [handlePageContentRequest()](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/glance.go#L334-L367) 处理：

```go
func (a *application) handlePageContentRequest(w http.ResponseWriter, r *http.Request) {
    // ...
    func() {
        page.mu.Lock()
        defer page.mu.Unlock()
        page.updateOutdatedWidgets()   // ← 核心调度入口
        err = pageContentTemplate.Execute(&responseBytes, pageData)
    }()
}
```

注意：刷新是**由页面访问驱动的被动刷新**，没有后台定时器主动刷新。如果 1 小时内没人访问页面，Widget 的数据也不会更新。

#### 2.2.2 Page 层调度：updateOutdatedWidgets()

[page.updateOutdatedWidgets()](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/glance.go#L233-L270) 遍历该页面所有 Widget，决定哪些需要更新：

```go
func (p *page) updateOutdatedWidgets() {
    now := time.Now()
    var wg sync.WaitGroup
    ctx := context.Background()

    for w := range p.HeadWidgets {
        widget := p.HeadWidgets[w]
        if !widget.requiresUpdate(&now) {   // ← 判定是否需要更新
            continue
        }
        wg.Add(1)
        go func() {
            defer wg.Done()
            widget.update(ctx)   // ← 每个需更新的 Widget 在独立 goroutine 中执行
        }()
    }
    // Columns 中的 Widget 同理
    wg.Wait()
}
```

关键点：
- 多个过期 Widget 会**并发**执行 update
- 同一个页面的 Widget 共享 `page.mu` 全局锁，保证渲染时数据一致性
- 传入的 `context` 是 `context.Background()`，永不超时、永不取消

#### 2.2.3 是否需要更新的判定：requiresUpdate()

[widgetBase.requiresUpdate()](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget.go#L173-L183) 逻辑极为简单：

```go
func (w *widgetBase) requiresUpdate(now *time.Time) bool {
    if w.cacheType == cacheTypeInfinite {
        return false
    }
    if w.nextUpdate.IsZero() {   // 首次运行，nextUpdate 未设置
        return true
    }
    return now.After(w.nextUpdate)   // 当前时间是否晚于计划更新时间
}
```

Monitor Widget 的 `cacheType` 为 `cacheTypeDuration`，缓存 5 分钟，因此每 5 分钟（或首次访问）会触发一次更新。

#### 2.2.4 调度决策核心：canContinueUpdateAfterHandlingErr()

[widgetBase.canContinueUpdateAfterHandlingErr()](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget.go#L293-L325) 是所有 Widget 共用的调度决策枢纽。它接收 update 过程中产生的 error，同时完成三件事：
1. 设置 Widget 级错误/提示状态（`Error` vs `Notice`）
2. 决定是否继续处理本次已获取的数据
3. 调度下一次更新的时间

完整决策树：

```
canContinueUpdateAfterHandlingErr(err)
         │
         ▼
    err == nil?
         │
         ├── 是（无错误）
         │     ├── withNotice(nil)        清除提示
         │     ├── withError(nil)         清除错误
         │     ├── scheduleNextUpdate()   正常周期刷新
         │     └── return true            继续处理数据
         │
         └── 否（有错误）
               ├── scheduleEarlyUpdate()  指数退避加速重试
               │
               └── err == errPartialContent?
                     │
                     ├── 是（部分内容获取失败）
                     │     ├── withError(nil)     不标记整体错误
                     │     ├── withNotice(err)    标记为提示（黄色警告）
                     │     └── return true        继续处理已获取的数据
                     │
                     └── 否（完全失败）
                           ├── withError(err)     标记整体错误（红色错误）
                           ├── withNotice(nil)    无提示
                           └── return false       中止后续处理
```

#### 2.2.5 两种调度策略的算法

**正常调度 scheduleNextUpdate()** [widget.go#L343-L348](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget.go#L343-L348)：
```go
func (w *widgetBase) scheduleNextUpdate() *widgetBase {
    w.nextUpdate = w.getNextUpdateTime()   // now + cacheDuration
    w.updateRetriedTimes = 0                // 重置退避计数器
    return w
}
```

**退避调度 scheduleEarlyUpdate()** [widget.go#L350-L367](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget.go#L350-L367)：
```go
func (w *widgetBase) scheduleEarlyUpdate() *widgetBase {
    w.updateRetriedTimes++
    if w.updateRetriedTimes > 5 {
        w.updateRetriedTimes = 5          // 计数器上限 5
    }
    // 指数退避：1²=1分钟, 2²=4分钟, 3²=9分钟, 4²=16分钟, 5²=25分钟
    nextEarlyUpdate := time.Now().Add(
        time.Duration(math.Pow(float64(w.updateRetriedTimes), 2)) * time.Minute)
    nextUsualUpdate := w.getNextUpdateTime()

    // 取更早的那个时间点
    if nextEarlyUpdate.After(nextUsualUpdate) {
        w.nextUpdate = nextUsualUpdate    // 退避到 25 分钟时不会超过正常 5 分钟周期
    } else {
        w.nextUpdate = nextEarlyUpdate    // 连续失败时：1分钟 → 4分钟 → 5分钟（被正常周期截断）
    }
    return w
}
```

对 Monitor Widget（正常周期 5 分钟）而言，实际退避序列为：
- 第 1 次失败：1 分钟后重试
- 第 2 次失败：4 分钟后重试
- 第 3 次及以后：5 分钟后重试（因 9 分钟 > 正常 5 分钟，被截断）

`updateRetriedTimes` 计数器**只在 scheduleNextUpdate() 时重置**，即只有一次完全成功的更新才能清零退避计数。

---

## 三、站点检查流程

### 3.1 更新入口 `update()`

```go
func (widget *monitorWidget) update(ctx context.Context) {
    requests := make([]*SiteStatusRequest, len(widget.Sites))
    for i := range widget.Sites {
        requests[i] = widget.Sites[i].SiteStatusRequest
    }

    statuses, err := fetchStatusForSites(requests)
    if !widget.canContinueUpdateAfterHandlingErr(err) {
        return
    }

    widget.HasFailing = false
    for i := range widget.Sites {
        site := &widget.Sites[i]
        status := &statuses[i]
        site.Status = status

        if !slices.Contains(site.AltStatusCodes, status.Code) && 
           (status.Code >= 400 || status.Error != nil) {
            widget.HasFailing = true
        }

        if status.Error != nil && site.ErrorURL != "" {
            site.URL = site.ErrorURL
        } else {
            site.URL = site.DefaultURL
        }

        site.StatusText = statusCodeToText(status.Code, site.AltStatusCodes)
        site.StatusStyle = statusCodeToStyle(status.Code, site.AltStatusCodes)
    }
}
```
参考 [widget-monitor.go#L43-L76](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L43-L76)

**判定异常的条件**：
1. 状态码不在 `AltStatusCodes` 白名单中，**且**
2. 状态码 >= 400 **或** 存在请求错误

### 3.2 单站点探测 `fetchSiteStatusTask()`

```go
func fetchSiteStatusTask(statusRequest *SiteStatusRequest) (siteStatus, error) {
    var url string
    if statusRequest.CheckURL != "" {
        url = statusRequest.CheckURL
    } else {
        url = statusRequest.DefaultURL
    }

    timeout := ternary(statusRequest.Timeout > 0, 
        time.Duration(statusRequest.Timeout), 3*time.Second)
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()

    request, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return siteStatus{Error: err}, nil
    }

    if statusRequest.BasicAuth.Username != "" || 
       statusRequest.BasicAuth.Password != "" {
        request.SetBasicAuth(statusRequest.BasicAuth.Username, 
            statusRequest.BasicAuth.Password)
    }

    requestSentAt := time.Now()
    var response *http.Response

    if !statusRequest.AllowInsecure {
        response, err = defaultHTTPClient.Do(request)
    } else {
        response, err = defaultInsecureHTTPClient.Do(request)
    }

    status := siteStatus{ResponseTime: time.Since(requestSentAt)}

    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            status.TimedOut = true
        }
        status.Error = err
        return status, nil
    }

    defer response.Body.Close()
    status.Code = response.StatusCode
    return status, nil
}
```
参考 [widget-monitor.go#L135-L183](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L135-L183)

**核心流程**：
1. **URL 选择**：优先使用 `CheckURL`（探测专用地址），否则回退到 `DefaultURL`
2. **超时设置**：使用配置的超时值，默认 **3 秒**
3. **上下文控制**：通过 `context.WithTimeout()` 实现请求级超时
4. **Basic Auth**：如配置了用户名/密码，自动添加 `Authorization` 头
5. **TLS 路由**：根据 `AllowInsecure` 选择不同的 HTTP Client
6. **响应时间**：从发送请求到收到响应头的耗时
7. **超时识别**：通过 `errors.Is(err, context.DeadlineExceeded)` 判断是否超时

---

## 四、超时控制机制

Monitor Widget 实现了**两层请求级超时**，此外 `widgetBase` 基类还提供了通用的指数退避调度能力（但在 Monitor 中实际不会触发，详见 4.3 节）。

### 4.1 第一层：请求级超时（单站点）

位置：[widget-monitor.go#L143-L145](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L143-L145)

```go
timeout := ternary(statusRequest.Timeout > 0, 
    time.Duration(statusRequest.Timeout), 3*time.Second)
ctx, cancel := context.WithTimeout(context.Background(), timeout)
```

- 默认值：**3 秒**
- 可配置：通过 YAML 的 `timeout` 字段，支持 `s/m/h/d` 后缀
- 解析逻辑见 [config-fields.go#L96-L130](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/config-fields.go#L96-L130) 中的 `durationField`

### 4.2 第二层：HTTP Client 全局超时

位置：[widget-utils.go#L24-L40](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-utils.go#L24-L40)

```go
const defaultClientTimeout = 5 * time.Second

var defaultHTTPClient = &http.Client{
    Transport: &http.Transport{
        MaxIdleConnsPerHost: 10,
        Proxy:               http.ProxyFromEnvironment,
    },
    Timeout: defaultClientTimeout,
}
```

- Client 级超时为 **5 秒**，覆盖连接、重定向、读取响应体的全过程
- 与请求级超时形成**双重保险**，取先触发者

### 4.3 基类退避调度与 Monitor 的实际行为

`widgetBase` 基类提供了通用的指数退避能力，位置：[widget.go#L350-L367](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget.go#L350-L367)，算法为：重试次数 n，间隔为 n² 分钟，计数器上限 5。

**但对 Monitor Widget 而言，该退避机制永远不会被触发**。原因在于错误传递路径的设计：

1. 单站失败（超时、DNS 错误等）→ 错误存储在 `siteStatus.Error` 中，属于**站点级数据**
2. `fetchStatusForSites()` 只检查 `workerPoolDo()` 返回的第三个值（Worker Pool 级错误），忽略单站错误
3. `workerPoolDo()` 的第三个返回值仅在 `job.ctx.Done()` 触发时非 nil，而 Monitor 使用 `context.Background()`，永不取消
4. 因此传给 `canContinueUpdateAfterHandlingErr()` 的 err **永远为 nil** → 始终走 `scheduleNextUpdate()` → 固定 5 分钟周期

完整的错误传递分析见第十章 10.3 节，以下是所有失败场景与下一次刷新时间的因果关系矩阵：

### 4.4 失败场景与刷新时间因果关系矩阵

| 失败场景 | siteStatus.Error | siteStatus.TimedOut | fetchStatusForSites 返回 err | 调度函数 | 下一次刷新时间 |
|---------|------------------|---------------------|------------------------------|---------|--------------|
| **正常 200** | nil | false | nil | scheduleNextUpdate | T + 5min |
| **HTTP 404/500** | nil | false | nil | scheduleNextUpdate | T + 5min |
| **单站请求超时** | context.DeadlineExceeded | true | nil | scheduleNextUpdate | T + 5min |
| **单站 DNS 解析失败** | DNSError | false | nil | scheduleNextUpdate | T + 5min |
| **单站 TLS 握手失败** | TLSError | false | nil | scheduleNextUpdate | T + 5min |
| **所有站点全部超时** | 多站均为 DeadlineExceeded | 多站 true | nil | scheduleNextUpdate | T + 5min |
| **所有站点全部挂掉** | 多站均为非 nil | 多站 false | nil | scheduleNextUpdate | T + 5min |
| **Worker Pool context 被取消（需改源码）** | 未执行的站点为零值 | false | context.Canceled 或 DeadlineExceeded | scheduleEarlyUpdate | 1min → 4min → 5min（被正常周期截断） |

**结论**：Monitor Widget 在所有正常使用场景下，下一次刷新时间恒为「当前时间 + 缓存周期（默认 5 分钟）」，与被监控站点的健康状态无关。指数退避机制仅存在于基类的理论路径中，对 Monitor 是不可达的死代码。

---

## 五、TLS 处理机制

### 5.1 双 Client 架构

系统维护两个全局 HTTP Client，分别处理安全与不安全 TLS 请求：

| Client | TLS 配置 | 用途 |
|--------|----------|------|
| `defaultHTTPClient` | 标准 TLS 验证（默认） | 正常 HTTPS 站点 |
| `defaultInsecureHTTPClient` | `InsecureSkipVerify: true` | 自签名证书/内部站点 |

参考 [widget-utils.go#L26-L40](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-utils.go#L26-L40)

### 5.2 Client 选择逻辑

位置：[widget-monitor.go#L161-L165](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L161-L165)

```go
if !statusRequest.AllowInsecure {
    response, err = defaultHTTPClient.Do(request)
} else {
    response, err = defaultInsecureHTTPClient.Do(request)
}
```

每个站点可通过 YAML 的 `allow-insecure: true` 独立配置是否跳过 TLS 证书验证。

### 5.3 连接池复用

两个 Client 均配置了 `MaxIdleConnsPerHost: 10`，支持对同一主机的连接复用，减少频繁探测时的 TLS 握手开销。

---

## 六、并发调度机制

### 6.1 Worker Pool 架构

Monitor Widget 使用**固定大小 Worker Pool** 并发执行探测任务，核心实现位于 [widget-utils.go#L141-L242](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-utils.go#L141-L242)。

### 6.2 调用入口

```go
func fetchStatusForSites(requests []*SiteStatusRequest) ([]siteStatus, error) {
    job := newJob(fetchSiteStatusTask, requests).withWorkers(20)
    results, _, err := workerPoolDo(job)
    if err != nil {
        return nil, err
    }
    return results, nil
}
```
参考 [widget-monitor.go#L185-L193](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L185-L193)

Monitor Widget 指定了 **20 个 Worker**，但实际数量不会超过待探测站点数。

### 6.3 Worker Pool 执行流程

```
                    ┌───────────────────┐
                    │   tasksQueue      │  (chan *workerPoolTask)
                    └─────────▲─────────┘
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
    ┌──────┴──────┐    ┌──────┴──────┐    ┌──────┴──────┐
    │  Worker 1   │    │  Worker 2   │    │  Worker N   │
    │  (goroutine)│    │  (goroutine)│    │  (goroutine)│
    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
           │                  │                  │
           └──────────────────┼──────────────────┘
                              ▼
                    ┌───────────────────┐
                    │  resultsQueue     │  (chan *workerPoolTask)
                    └───────────────────┘
```

关键实现细节 [widget-utils.go#L184-L242](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-utils.go#L184-L242)：

1. **任务分发 Goroutine**：将输入数据封装为任务写入 `tasksQueue`，支持通过 `ctx` 取消
2. **Worker Goroutine**：每个 Worker 从 `tasksQueue` 消费任务，执行后写入 `resultsQueue`
3. **结果收集**：主 Goroutine 从 `resultsQueue` 读取，按原始索引回填到结果切片
4. **同步机制**：
   - `close(tasksQueue)` 通知 Worker 退出
   - `sync.WaitGroup` 等待所有 Worker 完成
   - `close(resultsQueue)` 通知主 Goroutine 收集结束
5. **短路优化**：当仅有 1 个任务时，直接同步执行，跳过协程池

### 6.4 并发参数

| 参数 | 值 | 说明 |
|------|----|------|
| Worker 数量 | 20（最多） | `withWorkers(20)` 指定 |
| 默认 Worker 数 | 10 | `defaultNumWorkers` |
| 每主机空闲连接 | 10 | `MaxIdleConnsPerHost` |

---

## 七、状态展示逻辑

### 7.1 状态码映射

`statusCodeToText()` 将 HTTP 状态码转为人类可读文本：

| 状态码 | 显示文本 |
|--------|----------|
| 200 或在 AltStatusCodes 中 | `OK` |
| 404 | `Not Found` |
| 403 | `Forbidden` |
| 401 | `Unauthorized` |
| >= 500 | `Server Error` |
| >= 400 | `Client Error` |
| 其他 | 状态码数字本身 |

参考 [widget-monitor.go#L86-L107](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L86-L107)

### 7.2 样式分类

`statusCodeToStyle()` 仅返回两种样式：`ok` 或 `error`：

```go
func statusCodeToStyle(status int, altStatusCodes []int) string {
    if status == 200 || slices.Contains(altStatusCodes, status) {
        return "ok"
    }
    return "error"
}
```
参考 [widget-monitor.go#L109-L115](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L109-L115)

### 7.3 模板渲染

Monitor Widget 支持两种展示样式：

#### 7.3.1 标准模式 `monitor.html`

参考 [monitor.html](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/templates/monitor.html)

展示内容：
- 站点图标（可选）
- 站点标题（超链接）
- 状态文本 + 响应时间（毫秒）
- 右侧状态图标（绿色对勾 / 红色感叹号）

异常情况展示：
- 超时：显示 `Timed Out`（红色）
- 其他错误：显示 `ERROR`，鼠标悬浮显示错误详情

#### 7.3.2 紧凑模式 `monitor-compact.html`

参考 [monitor-compact.html](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/templates/monitor-compact.html)

精简展示，仅保留：
- 站点标题
- 响应时间（超时则不显示）
- 状态小圆点图标

#### 7.3.3 仅展示异常模式

当 `ShowFailingOnly: true` 且存在异常站点时，只展示状态为 `error` 的站点。若所有站点均正常，则显示统一的 "All sites are online" 提示。

### 7.4 样式表

状态图标的 CSS 定义见 [widget-monitor.css](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/static/css/widget-monitor.css)：
- `.monitor-site-status-icon`：标准模式图标（2rem）
- `.monitor-site-status-icon-compact`：紧凑模式图标（1.8rem）
- 图标颜色通过 CSS 变量 `--color-positive` 和 `--color-negative` 控制

---

## 八、完整流程图

```
用户请求页面
    │
    ▼
widget.requiresUpdate()? ──否──► 使用缓存渲染
    │是
    ▼
monitorWidget.update(ctx)
    │
    ├── 提取所有 SiteStatusRequest
    │
    ▼
fetchStatusForSites(requests)
    │
    ├── newJob(fetchSiteStatusTask, requests).withWorkers(20)
    │
    ▼
workerPoolDo(job)
    │
    ├── 启动 20 个 Worker Goroutine
    │
    ├── 并发执行 fetchSiteStatusTask()
    │       │
    │       ├── 选择 CheckURL 或 DefaultURL
    │       ├── context.WithTimeout (默认 3s)
    │       ├── 设置 Basic Auth (如配置)
    │       ├── 根据 AllowInsecure 选择 HTTP Client
    │       ├── 发送 GET 请求
    │       ├── 记录 ResponseTime
    │       ├── 识别超时 (context.DeadlineExceeded)
    │       └── 返回 siteStatus{Code, TimedOut, ResponseTime, Error}
    │
    ▼
收集所有结果，按原始索引排序
    │
    ▼
逐站点处理结果
    │
    ├── 判定异常：Code>=400 或 Error!=nil，且不在 AltStatusCodes
    │
    ├── 设置跳转 URL：有错误且配置 ErrorURL 则使用 ErrorURL
    │
    ├── 生成 StatusText 和 StatusStyle
    │
    └── 设置 widget.HasFailing 标记
    │
    ▼
scheduleNextUpdate() 或 scheduleEarlyUpdate()
    │
    ▼
根据 Style 渲染 monitor.html 或 monitor-compact.html
```

---

## 九、关键设计决策总结

| 设计点 | 方案 | 权衡 |
|--------|------|------|
| 并发模型 | 固定 Worker Pool (20) | 避免大量站点时 goroutine 爆炸 |
| 超时控制 | 两层请求级（请求级3s + Client级5s），基类退避机制对 Monitor 为死代码 | 请求级与 Client 级双重保险；退避机制因错误传递路径设计不可达 |
| TLS 处理 | 双 Client 全局复用 | 避免每个请求创建 Transport，同时隔离安全/非安全请求 |
| 状态判定 | AltStatusCodes 白名单 + Code>=400/Error | 灵活适配非标准部署（如 401 表示需要登录但服务正常） |
| URL 分离 | CheckURL vs DefaultURL vs ErrorURL | 探测地址、展示链接、失败跳转三者解耦 |
| 结果保序 | 通过 index 回填切片 | 保证渲染顺序与配置顺序一致 |

---

## 十、边界条件深度解析

### 10.1 三种异常场景的本质区别

`siteStatus` 的三个字段 `Code`、`TimedOut`、`Error` 组合出三种完全不同的异常路径，它们的触发来源和后续处理截然不同：

| 场景 | `Code` | `TimedOut` | `Error` | 触发来源 |
|------|--------|------------|---------|----------|
| 请求报错（非超时） | `0`（int 零值） | `false` | `!nil` | DNS 解析失败、TCP 连接拒绝、TLS 握手失败、证书校验失败等 |
| **超时** | `0`（int 零值） | `true` | `!nil` | `context.DeadlineExceeded`（请求级或 Client 级超时） |
| **HTTP 状态异常** | `>=400` | `false` | `nil` | 服务端返回了 4xx/5xx 响应，但 HTTP 请求本身成功 |

**关键区分点**：HTTP 状态异常（如 502 Bad Gateway）虽然在语义上是"服务不可用"，但从代码层面看 `Error == nil`，因为 TCP/TLS 握手和 HTTP 传输都成功完成了，只是业务层返回了错误码。这个差异会传导到后续所有处理逻辑。

参考 [fetchSiteStatusTask()](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L135-L183) 的错误分支：
```go
if err != nil {
    if errors.Is(err, context.DeadlineExceeded) {
        status.TimedOut = true
    }
    status.Error = err
    return status, nil   // Code 保持零值 0
}
// 只有走到这里 Code 才被赋值
defer response.Body.Close()
status.Code = response.StatusCode
```

---

### 10.2 失败跳转地址 ErrorURL 的精确触发条件

位置：[widget-monitor.go#L67-L71](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L67-L71)

```go
if status.Error != nil && site.ErrorURL != "" {
    site.URL = site.ErrorURL
} else {
    site.URL = site.DefaultURL
}
```

**真值表**：

| 场景 | `status.Error` | `site.ErrorURL` | 最终跳转 URL |
|------|----------------|-----------------|--------------|
| 正常响应 200 | nil | 任意 | `DefaultURL` |
| HTTP 404/500 等 | nil | 已配置 | `DefaultURL` |
| HTTP 404/500 等 | nil | 未配置 | `DefaultURL` |
| 请求报错（DNS 失败等） | !nil | 已配置 | **`ErrorURL`** |
| 请求报错（DNS 失败等） | !nil | 未配置 | `DefaultURL` |
| 超时 | !nil | 已配置 | **`ErrorURL`** |
| 超时 | !nil | 未配置 | `DefaultURL` |

**容易踩坑的边界**：HTTP 502/503/504 这类典型的"服务挂了"状态码，**不会**触发 ErrorURL 跳转，因为 `Error` 仍为 nil。如果希望这些状态码也跳转到备用地址，需要自行在 ErrorURL 上游做处理，或通过配置反向代理把故障转为 TCP 层面的连接失败。

---

### 10.3 刷新调度深度解析：探活失败会触发提前重试吗？

这是最容易产生误解的问题。简短答案是：**不会**。被监控的站点无论超时、DNS 失败还是返回 500，都不会启动指数退避机制。以下是完整的推导过程。

#### 10.3.1 Error 的三层传递模型

Monitor Widget 中存在三个完全独立的 error 通道，各自语义不同，绝不能混淆：

```
┌─────────────────────────────────────────────────────────────────┐
│  层级 1：站点级错误 (siteStatus.Error)                           │
│  ┌─────────┐  ┌─────────┐      ┌─────────┐                     │
│  │站点 A   │  │站点 B   │ ...  │站点 N   │                     │
│  │Error: nil│  │Error: DNSError │  │Error: nil│                     │
│  └────┬────┘  └────┬────┘      └────┬────┘                     │
│       │            │                │                           │
│       └────────────┴──────┬─────────┘                           │
│                           ▼                                     │
│                  每个站点独立处理，                              │
│                  用于单站状态展示、ErrorURL 跳转                 │
│                  不向上冒泡                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  层级 2：Worker Pool 级错误 (workerPoolDo 第三个返回值)          │
│                                                                 │
│  workerPoolDo() 返回 (results, errs, err)                       │
│                              │                                   │
│                              ├── results: []siteStatus          │
│                              │   (含各自的站点级 Error)          │
│                              │                                   │
│                              ├── errs: []error (被忽略!)         │
│                              │   fetchSiteStatusTask 的第二个    │
│                              │   返回值，Monitor 中始终为 nil   │
│                              │                                   │
│                              └── err: error                      │
│                                  仅 job.ctx.Done() 触发          │
│                                  Monitor 中始终为 nil            │
│                                                                 │
│  这是传给 canContinueUpdateAfterHandlingErr 的 err               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  层级 3：Widget 级错误 (widgetBase.Error / widgetBase.Notice)   │
│                                                                 │
│  由 canContinueUpdateAfterHandlingErr 根据层级 2 的 err 设置     │
│  影响整个 Widget 的头部错误提示和渲染分支                        │
└─────────────────────────────────────────────────────────────────┘
```

**关键发现**：层级 1 的站点级错误被完全"吸收"在 `results` 数组内部，永远不会变成层级 2 的 err，因此永远不会到达层级 3 的调度决策器。

#### 10.3.2 与其他 Widget 的对比：为什么 RSS 会退避而 Monitor 不会？

Glance 中的 Widget 分为两种错误处理范式：

**范式 A：整体失败型**（Weather、Custom-API 等单源 Widget）
```go
// widget-weather.go
weather, err := fetchWeatherForOpenMeteoPlace(...)
if !widget.canContinueUpdateAfterHandlingErr(err) {
    return   // err != nil → 整体失败，触发退避
}
```
- 数据源单一，失败就意味着整个 Widget 没有内容
- `err != nil` 会被直接传给调度器 → 触发 `scheduleEarlyUpdate()`

**范式 B：部分失败型**（RSS、Videos、Twitch、Markets、Monitor 等多源 Widget）

这些 Widget 的特点是：多个数据源中部分失败是常态，不应该因为一个源挂了就退避整个 Widget。但它们在实现上分为两派：

| Widget | 多源失败时返回的 err | 调度行为 |
|--------|---------------------|----------|
| RSS/Videos/Twitch/Markets | `fmt.Errorf("%w: missing %d feeds", errPartialContent, failed)` | **触发退避**（scheduleEarlyUpdate），但 return true 继续展示已有数据 |
| **Monitor** | **nil** | **不触发退避**，正常 5 分钟调度 |

Monitor 是多源 Widget 中**唯一不使用 errPartialContent** 的。原因在于设计语义不同：
- RSS/Videos 等：失败的源意味着"内容缺失"，可能是临时网络波动，值得加速重试补全
- Monitor：站点失败正是 Widget 需要展示的**正常业务状态**，不是"数据获取异常"。所有站点都挂了也是合法的监控结果，Widget 的内容完整无缺

#### 10.3.3 workerPoolDo 中 err 的赋值路径

[workerPoolDo()](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-utils.go#L184-L242) 返回的第三个 error 有且仅有一个赋值点：

```go
var err error
go func() {
loop:
    for i := range job.data {
        select {
        default:
            tasksQueue <- ...
        case <-job.ctx.Done():
            err = job.ctx.Err()   // ← 唯一赋值点
            break loop
        }
    }
    close(tasksQueue)
    wg.Wait()
    close(resultsQueue)
}()
```

而 Monitor 调用链中的 job 创建于 [newJob()](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-utils.go#L175-L182)：

```go
func newJob[I any, O any](task func(I) (O, error), data []I) *workerPoolJob[I, O] {
    return &workerPoolJob[I, O]{
        workers: defaultNumWorkers,
        task:    task,
        data:    data,
        ctx:     context.Background(),   // ← 永不超时、永不取消
    }
}
```

虽然 `workerPoolJob` 预留了 `withContext()` 方法（当前被注释掉），但 Monitor Widget 没有传入自定义 context。因此 `job.ctx.Done()` 永远不会触发，`workerPoolDo` 的第三个返回值永远是 `nil`。

此外，`fetchSiteStatusTask` 的签名是 `func(...) (siteStatus, error)`，它**永远把 error 放在第一个返回值的 Error 字段中，第二个返回值始终为 nil**。即使某个站点探测完全失败，`workerPoolDo` 返回的 `errs`（第二个返回值）切片中也全是 nil，而 Monitor 在 `fetchStatusForSites` 中直接丢弃了这个返回值：

```go
results, _, err := workerPoolDo(job)   // ← 第二个返回值被下划线忽略
```

#### 10.3.4 退避机制真正启动的充要条件

对 Monitor Widget 而言，要让 `scheduleEarlyUpdate()` 被调用，需要**同时满足**以下条件：

| # | 条件 | Monitor 中是否可能 |
|---|------|-------------------|
| 1 | `fetchStatusForSites` 返回的 err != nil | 几乎不可能（需修改源码传入可取消的 context） |
| 2 | 该 err 被传入 `canContinueUpdateAfterHandlingErr` | 依赖条件 1 |
| 3 | 页面被用户访问，触发 `updateOutdatedWidgets` | 是（被动刷新的前提） |
| 4 | `widget.requiresUpdate()` 返回 true | 是（到达计划更新时间） |

**结论**：在不修改源码的情况下，Monitor Widget 的指数退避机制是**死代码**。无论监控多少个站点、它们失败得多么彻底，Widget 始终按 `scheduleNextUpdate()` 的 5 分钟周期执行。

#### 10.3.5 完整时间线示例

假设一个 Monitor Widget 监控 3 个站点，配置缓存 5 分钟。站点 A 在 T=0 后宕机：

```
T=0min   用户首次访问页面
         → requiresUpdate() 返回 true (nextUpdate 为零值)
         → update() 执行：A=200, B=200, C=200
         → canContinueUpdateAfterHandlingErr(nil)
         → scheduleNextUpdate() → nextUpdate = T+5min
         → updateRetriedTimes = 0

T=3min   用户访问（未到 5 分钟）
         → requiresUpdate() 返回 false
         → 使用缓存渲染，不执行 update()

T=5min   用户访问
         → requiresUpdate() 返回 true
         → update() 执行：A=DNS 失败, B=200, C=200
         → fetchStatusForSites 返回 err=nil
         → canContinueUpdateAfterHandlingErr(nil)
         → scheduleNextUpdate() → nextUpdate = T+10min
         → updateRetriedTimes 保持 0 (无退避)

T=6min   用户访问
         → requiresUpdate() 返回 false

T=10min  用户访问
         → update() 执行：A=DNS 失败, B=200, C=502
         → err 仍为 nil
         → scheduleNextUpdate() → nextUpdate = T+15min
         → updateRetriedTimes 仍为 0

T=15min  A 恢复，update() 执行：A=200, B=200, C=200
         → 与失败时调度无差异，仍是 5 分钟周期
```

即使 A 连续宕机数小时，更新间隔也始终是 5 分钟，`updateRetriedTimes` 永远停留在 0，退避算法的 `math.Pow(..., 2)` 计算永不执行。

#### 10.3.6 Widget 级错误状态 vs 单站状态

需要特别区分两个完全独立的错误概念：

| 概念 | 存储位置 | 触发条件 | UI 表现 |
|------|---------|---------|--------|
| **Widget 级错误** | `widgetBase.Error` | `canContinueUpdateAfterHandlingErr` 收到非 errPartialContent 的错误 | Widget 头部显示红色错误条，覆盖整个内容区 |
| **单站状态异常** | `site[i].Status.Error` + `site[i].Status.Code` | 单站探测失败或 HTTP>=400 | 仅该站点显示红色状态图标和文本 |

Monitor Widget 在正常运行中只会出现单站异常，Widget 级错误永远不会被设置（除非 Worker Pool 框架本身出问题）。而 RSS Widget 在部分源失败时会出现 Widget 级 Notice（黄色提示），整体错误时出现 Widget 级 Error。

---

### 10.4 模板层展示分支详解

#### 10.4.1 标准模式分支逻辑

位置：[monitor.html#L29-L37](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/templates/monitor.html#L29-L37)

```html
{{ if not .Status.Error }}
    <!-- 分支 A：无请求错误 -->
    <li title="{{ .Status.Code }}">{{ .StatusText }}</li>
    <li>{{ .Status.ResponseTime.Milliseconds | formatNumber }}ms</li>
{{ else if .Status.TimedOut }}
    <!-- 分支 B：超时（Error != nil 且 TimedOut == true） -->
    <li class="color-negative">Timed Out</li>
{{ else }}
    <!-- 分支 C：其他请求错误（Error != nil 且 TimedOut == false） -->
    <li class="color-negative" title="{{ .Status.Error }}">ERROR</li>
{{ end }}
```

**各场景展示效果**：

| 场景 | 走哪个分支 | 状态文本 | 响应时间 | 右侧图标颜色 |
|------|-----------|----------|----------|-------------|
| 正常 200 | A | `OK` | `123ms` | 绿色 |
| HTTP 404 | A | `Not Found` | `45ms` | 红色 |
| HTTP 502 | A | `Server Error` | `89ms` | 红色 |
| HTTP 401 + AltStatusCodes=[401] | A | `OK` | `34ms` | 绿色 |
| 超时 | B | `Timed Out`（红色） | **不显示** | 红色 |
| DNS 解析失败 | C | `ERROR`（红色，悬浮看详情） | **不显示** | 红色 |
| TLS 证书错误 | C | `ERROR`（红色，悬浮看详情） | **不显示** | 红色 |

**注意**：HTTP 状态异常（>=400）虽然在业务上是故障，但在模板层走分支 A，仍会显示响应时间，这是因为 TCP 请求确实完成了。

#### 10.4.2 紧凑模式分支差异

位置：[monitor-compact.html#L23-L38](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/templates/monitor-compact.html#L23-L38)

紧凑模式只做两个判断：

```html
{{ if not .Status.TimedOut }}<div>...ms</div>{{ end }}
{{ if eq .StatusStyle "ok" }}
    <div ... title="{{ .Status.Code }}">
{{ else }}
    <div ... title="{{ if .Status.Error }}{{ .Status.Error }}{{ else }}{{ .Status.Code }}{{ end }}">
```

与标准模式的关键差异：

| 差异点 | 标准模式 | 紧凑模式 |
|--------|---------|---------|
| 响应时间显示条件 | `not .Status.Error` | `not .Status.TimedOut` |
| DNS 失败时 | 不显示响应时间 | **显示响应时间** |
| TLS 错误时 | 不显示响应时间 | **显示响应时间** |
| 超时时 | 不显示响应时间 | 不显示响应时间 |
| 状态 tooltip 内容 | 分支 A:Code，分支 B/C:无 | ok:Code，error:优先 Error 其次 Code |

**紧凑模式的小瑕疵**：非超时的请求错误（如 DNS 失败）也会显示响应时间，但这个时间实际上是从请求发起到底层报错的耗时（通常很短，如几毫秒），参考意义有限。标准模式在这个处理上更严谨。

#### 10.4.3 ShowFailingOnly 与 HasFailing 的联动

```
外层判断 {{ if not (and .ShowFailingOnly (not .HasFailing)) }}
    │
    ├── ShowFailingOnly=false → 始终渲染完整列表
    │
    └── ShowFailingOnly=true
            │
            ├── HasFailing=true → 渲染列表，内层 {{ if eq .StatusStyle "ok" }} 过滤正常站点
            └── HasFailing=false → 显示 "All sites are online" 统一提示
```

内层循环中的过滤 [monitor.html#L7](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/templates/monitor.html#L7)：
```html
{{ if and $.ShowFailingOnly (eq .StatusStyle "ok" ) }} {{ continue }} {{ end }}
```

**StatusStyle 的计算** [widget-monitor.go#L109-L115](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L109-L115)：
```go
func statusCodeToStyle(status int, altStatusCodes []int) string {
    if status == 200 || slices.Contains(altStatusCodes, status) {
        return "ok"
    }
    return "error"
}
```

请求报错和超时场景下 `status.Code == 0`，必然返回 `"error"`，会被正确保留在过滤后的列表中。

---

### 10.5 HasFailing 标记的完整判定

位置：[widget-monitor.go#L63-L65](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-monitor.go#L63-L65)

```go
if !slices.Contains(site.AltStatusCodes, status.Code) && 
   (status.Code >= 400 || status.Error != nil) {
    widget.HasFailing = true
}
```

判定逻辑为 **AND( 不在白名单 , OR(Code>=400, 有错误) )**：

| 场景 | Code | Code 在白名单 | Error | HasFailing |
|------|------|--------------|-------|------------|
| 正常 200 | 200 | 否 | nil | **false** |
| HTTP 401 + 白名单=[401] | 401 | **是** | nil | **false** |
| HTTP 404 | 404 | 否 | nil | **true** |
| HTTP 502 + 白名单=[502] | 502 | **是** | nil | **false** |
| 超时 | 0 | 否 | !nil | **true** |
| DNS 失败 | 0 | 否 | !nil | **true** |
| HTTP 0（不可能，仅理论） | 0 | 否 | nil | false |

注意第二行和第五行的对比：HTTP 401 只要在白名单中就不算异常，哪怕业务上可能表示鉴权失效。设计上假设用户在配置白名单时已理解该状态码的含义。

---

### 10.6 全链路关系总览图

```
fetchSiteStatusTask() 返回 siteStatus
         │
         ├───────────────────────────────────────────────────────┐
         │                                                       │
         ▼                                                       ▼
  status.Error?                                        status.Code 值
         │                                                       │
         ├── !nil ──┬── TimedOut?                                │
         │          ├── true  → "Timed Out" 分支(模板)           │
         │          └── false → "ERROR" 分支(模板)               │
         │                                                       │
         │          ┌────────────────────────────────────────────┘
         │          │
         ▼          ▼
  ErrorURL 切换?  statusCodeToText() / statusCodeToStyle()
         │          │
         │          ├── 200 或在白名单 → "OK" + "ok"
         │          ├── 404 → "Not Found" + "error"
         │          ├── 403 → "Forbidden" + "error"
         │          ├── 401 → "Unauthorized" + "error"
         │          ├── >=500 → "Server Error" + "error"
         │          ├── >=400 → "Client Error" + "error"
         │          └── 其他 → strconv.Itoa(Code) + "error"
         │
         ├── true  (Error!=nil && ErrorURL!="") → URL=ErrorURL
         └── false (其他所有情况)               → URL=DefaultURL

         │                                    │
         ▼                                    ▼
  canContinueUpdateAfterHandlingErr()    HasFailing 判定
         │                                    │
         ├── fetchStatusForSites err 恒为 nil  ├── 白名单外 AND (Code>=400 OR Error!=nil)
         └── scheduleNextUpdate() → 5 分钟     └── 任一站点命中即 HasFailing=true

