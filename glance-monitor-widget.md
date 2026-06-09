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

### 2.2 缓存调度机制

缓存调度由 `widgetBase` 基类统一管理，涉及以下关键方法：

- [requiresUpdate()](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget.go#L173-L183)：判断是否需要执行更新
- [scheduleNextUpdate()](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget.go#L343-L348)：成功时按正常周期调度
- [scheduleEarlyUpdate()](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget.go#L350-L367)：失败时采用**指数退避**策略重试

指数退避算法：重试间隔 = `retryCount²` 分钟，最大重试次数为 5，即最长间隔为 25 分钟，但不超过正常缓存周期。

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

Monitor Widget 实现了**三层超时控制**：

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

### 4.3 第三层：缓存更新重试退避

位置：[widget.go#L350-L367](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget.go#L350-L367)

当更新失败时，采用指数退避：重试次数 n，间隔为 n² 分钟，最多 5 次。

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
| 超时控制 | 三层（请求级3s + Client级5s + 退避重试） | 兼顾灵敏度与稳定性 |
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

### 10.3 刷新调度的真实触发逻辑

刷新调度由 [canContinueUpdateAfterHandlingErr()](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget.go#L293-L325) 控制，但这里存在一个极易误解的点：

#### 调度决策路径

```
fetchStatusForSites(requests)
        │
        ▼
  返回 (results, err)
        │
        ▼
  canContinueUpdateAfterHandlingErr(err)
        │
        ├── err == nil  → scheduleNextUpdate()   正常5分钟后刷新
        └── err != nil  → scheduleEarlyUpdate()  指数退避重试
```

#### 关键问题：`fetchStatusForSites` 什么时候返回 err？

追踪调用链：

```
fetchStatusForSites()
  → workerPoolDo(job)         返回 (results, errs, err)
     → 只有 job.ctx.Done() 触发时，第三个返回值 err 才非 nil
```

位置：[widget-utils.go#L214-L234](file:///d:/fz/0601/solo-dogfeeding/code/140-glance/internal/glance/widget-utils.go#L214-L234)

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
    ...
}()
```

而 `newJob()` 创建 job 时使用的是 `context.Background()`：

```go
func newJob[I any, O any](task func(I) (O, error), data []I) *workerPoolJob[I, O] {
    return &workerPoolJob[I, O]{
        ...
        ctx: context.Background(),   // ← 永不取消的 context
    }
}
```

#### 结论

| 故障场景 | `fetchStatusForSites` 返回 err | 调度行为 |
|----------|-------------------------------|----------|
| 单个站点超时 | **否** | 正常 5 分钟刷新 |
| 单个站点 DNS 失败 | **否** | 正常 5 分钟刷新 |
| 单个站点返回 500 | **否** | 正常 5 分钟刷新 |
| 所有站点全部超时 | **否** | 正常 5 分钟刷新 |
| Worker Pool context 被取消（极罕见） | **是** | 指数退避重试 |

**设计要点**：Monitor Widget 采用的是"**结果内聚错误**"模型——每个站点的错误存储在各自 `siteStatus.Error` 中，由上层 UI 逐站展示；Worker Pool 层面的错误（即 `fetchStatusForSites` 返回的 err）仅用于表示"整个调度框架出了问题"，而非"被监控站点出了问题"。因此**被监控的站点故障永远不会触发 early update**，始终按 5 分钟正常周期刷新。

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

