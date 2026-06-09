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
