# RSS 多源抓取与缓存协作详解

## 整体架构概览

RSS Widget 的核心数据流如下：

```
HTTP 请求触发页面刷新
        ↓
Page.updateOutdatedWidgets()  ──→  判断 requiresUpdate()
        ↓                              (组件级缓存过期检查)
启动多 Widget 并发更新 (每个 Widget 独立 goroutine)
        ↓
RSS Widget.update()
        ↓
Worker Pool (30 并发) ──→ 单源抓取 (fetchItemsFromFeedTask)
       │                           │
       │                    ┌──────┴──────┐
       │                    ↓             ↓
       │               HTTP 请求 + 缓存协商    解析 + 字符集处理
       │                    │
       │                    ↓
       └──────────→ 合并结果 + 去重
                          ↓
                     按时间排序 / 裁剪数量
                          ↓
              成功 → scheduleNextUpdate() / 失败 → scheduleEarlyUpdate() (指数退避重试)
```

---

## 一、组件级更新缓存（Widget 级 TTL 缓存）

RSS Widget 并不是每次页面请求都重新抓取，而是由组件级缓存机制控制更新频率。这是最外层的缓存，决定 `update()` 是否真正执行。

### 1.1 缓存配置字段

定义在 `widgetBase` 结构体中，见 [internal/glance/widget.go#L132-L167](internal/glance/widget.go#L132-L167)：

```go
type cacheType int

const (
    cacheTypeInfinite cacheType = iota   // 永不更新
    cacheTypeDuration                    // 按固定时长
    cacheTypeOnTheHour                   // 到整点自动更新
)

type widgetBase struct {
    CustomCacheDuration durationField    `yaml:"cache"`     // 用户可配置的缓存时长（YAML 字段）
    cacheDuration       time.Duration    `yaml:"-"`         // 实际生效的缓存时长（运行时字段）
    cacheType           cacheType        `yaml:"-"`         // 缓存类型
    nextUpdate          time.Time        `yaml:"-"`         // 下次允许更新的时间点
    updateRetriedTimes  int              `yaml:"-"`         // 已重试次数（用于指数退避）
    ContentAvailable    bool             `yaml:"-"`         // 是否已有有效内容
    Error               error            `yaml:"-"`         // 最近一次致命错误
    Notice              error            `yaml:"-"`         // 提示性错误（如部分内容缺失）
}
```

`durationField` 是自定义 YAML 解析类型，支持 `30s`、`5m`、`2h`、`1d` 四种后缀，见 [internal/glance/config-fields.go#L96-L130](internal/glance/config-fields.go#L96-L130)：

```go
var durationFieldPattern = regexp.MustCompile(`^(\d+)(s|m|h|d)$`)

type durationField time.Duration
```

### 1.2 RSS Widget 的默认缓存配置

在 `initialize()` 中设置默认 2 小时缓存，见 [internal/glance/widget-rss.go#L49-L78](internal/glance/widget-rss.go#L49-L78)：

```go
func (widget *rssWidget) initialize() error {
    widget.withTitle("RSS Feed").withCacheDuration(2 * time.Hour)

    if widget.Limit <= 0 {
        widget.Limit = 25
    }
    // ...
    widget.cachedFeeds = make(map[string]*cachedRSSFeed)
    return nil
}
```

`withCacheDuration()` 的逻辑：用户配置优先，否则使用默认值，见 [internal/glance/widget.go#L259-L269](internal/glance/widget.go#L259-L269)：

```go
func (w *widgetBase) withCacheDuration(duration time.Duration) *widgetBase {
    w.cacheType = cacheTypeDuration

    if duration == -1 || w.CustomCacheDuration == 0 {
        w.cacheDuration = duration          // 传入默认值（RSS 为 2h）
    } else {
        w.cacheDuration = time.Duration(w.CustomCacheDuration)  // 用户 YAML 配置优先
    }

    return w
}
```

**优先级：** 用户 YAML `cache:` 字段（如 `cache: 30m`）> Widget 默认值（2 小时）。

### 1.3 更新判断：requiresUpdate()

每次页面请求时调用，决定是否需要真正执行 `update()`，见 [internal/glance/widget.go#L173-L183](internal/glance/widget.go#L173-L183)：

```go
func (w *widgetBase) requiresUpdate(now *time.Time) bool {
    if w.cacheType == cacheTypeInfinite {
        return false                          // 无限缓存，永不更新
    }

    if w.nextUpdate.IsZero() {
        return true                           // 首次运行，nextUpdate 为零值，必须更新
    }

    return now.After(w.nextUpdate)            // 当前时间超过 nextUpdate 才允许更新
}
```

### 1.4 更新触发入口：Page 级调度

页面内容请求时 `handlePageContentRequest` 会先获取 `page.mu` 互斥锁，再调用 `page.updateOutdatedWidgets()`，见 [internal/glance/glance.go#L334-L367](internal/glance/glance.go#L334-L367) 和 [internal/glance/glance.go#L233-L270](internal/glance/glance.go#L233-L270)：

```go
func (a *application) handlePageContentRequest(w http.ResponseWriter, r *http.Request) {
    // ...
    func() {
        page.mu.Lock()                 // 同一 Page 的更新被串行化
        defer page.mu.Unlock()

        page.updateOutdatedWidgets()   // 更新所有过期 Widget
        err = pageContentTemplate.Execute(&responseBytes, pageData)
    }()
    // ...
}
```

`page` 结构体含互斥锁 `mu`，见 [internal/glance/config.go#L77-L92](internal/glance/config.go#L77-L92)：

```go
type page struct {
    Title   string        `yaml:"name"`
    Columns []pageColumn  `yaml:"columns"`
    // ...
    mu      sync.Mutex    `yaml:"-"`
}
```

`updateOutdatedWidgets()` 遍历所有过期 Widget 并并发启动 `update()`，见 [internal/glance/glance.go#L233-L270](internal/glance/glance.go#L233-L270)：

```go
func (p *page) updateOutdatedWidgets() {
    now := time.Now()
    var wg sync.WaitGroup
    context := context.Background()

    for w := range p.HeadWidgets {
        widget := p.HeadWidgets[w]
        if !widget.requiresUpdate(&now) {   // 未过期则跳过
            continue
        }
        wg.Add(1)
        go func() {                          // 每个过期 Widget 一个 goroutine
            defer wg.Done()
            widget.update(context)
        }()
    }

    for c := range p.Columns {
        for w := range p.Columns[c].Widgets {
            // Column 中 Widget 同理
        }
    }

    wg.Wait()                                // 阻塞等待全部完成
}
```

容器类 Widget（group/split-column）内部也有完全相同的并发逻辑，见 [internal/glance/widget-container.go#L23-L42](internal/glance/widget-container.go#L23-L42)。

### 1.5 成功/失败后的调度

RSS Widget 的 `update()` 最终会调用 `canContinueUpdateAfterHandlingErr()`，该方法决定下一次更新时间，见 [internal/glance/widget-rss.go#L80-L96](internal/glance/widget-rss.go#L80-L96) 和 [internal/glance/widget.go#L293-L325](internal/glance/widget.go#L293-L325)：

```go
func (widget *rssWidget) update(ctx context.Context) {
    items, err := widget.fetchItemsFromFeeds()

    if !widget.canContinueUpdateAfterHandlingErr(err) {
        return
    }
    // 排序、裁剪、赋值
}
```

`withError()` 方法在 `err == nil` 且当前 `ContentAvailable == false` 时会将 `ContentAvailable` 置为 `true`（标记首次成功），见 [internal/glance/widget.go#L283-L291](internal/glance/widget.go#L283-L291)：

```go
func (w *widgetBase) withError(err error) *widgetBase {
    if err == nil && !w.ContentAvailable {
        w.ContentAvailable = true
    }
    if err != nil {
        w.Error = err
        w.ContentAvailable = false
    } else {
        w.Error = nil
    }
    return w
}
```

调度分支：

```
无错误     → scheduleNextUpdate()        → nextUpdate = now + cacheDuration（2h），重试计数清零
部分错误   → scheduleEarlyUpdate()     → 指数退避，标记为 Notice，仍渲染已有内容
全部错误   → scheduleEarlyUpdate()     → 指数退避，标记为 Error，不渲染内容
```

---

## 二、部分源失败后的重试机制（指数退避）

### 2.1 错误分级与调度策略

在 `fetchItemsFromFeeds()` 中对失败源逐个计数，见 [internal/glance/widget-rss.go#L152-L190](internal/glance/widget-rss.go#L152-L190)：

```go
func (widget *rssWidget) fetchItemsFromFeeds() (rssFeedItemList, error) {
    requests := widget.FeedRequests

    job := newJob(widget.fetchItemsFromFeedTask, requests).withWorkers(30)
    feeds, errs, err := workerPoolDo(job)
    if err != nil {
        return nil, fmt.Errorf("%w: %v", errNoContent, err)
    }

    failed := 0
    entries := make(rssFeedItemList, 0, len(feeds)*10)
    seen := make(map[string]struct{})

    for i := range feeds {
        if errs[i] != nil {
            failed++
            slog.Error("Failed to get RSS feed", "url", requests[i].URL, "error", errs[i])
            continue
        }
        // ... 合并成功的条目，去重
    }

    if failed == len(requests) {
        return nil, errNoContent              // 全部失败：致命错误
    }
    if failed > 0 {
        return entries, fmt.Errorf("%w: missing %d RSS feeds", errPartialContent, failed)  // 部分失败
    }
    return entries, nil                       // 全部成功
}
```

`errNoContent` 与 `errPartialContent` 是包级预定义错误值，见 [internal/glance/widget-utils.go#L20-L21](internal/glance/widget-utils.go#L20-L21)：

```go
var (
    errNoContent      = errors.New("failed to retrieve any content")
    errPartialContent = errors.New("failed to retrieve some of the content")
)
```

随后在 `canContinueUpdateAfterHandlingErr()` 中根据错误类型分支，见 [internal/glance/widget.go#L293-L325](internal/glance/widget.go#L293-L325)：

```go
func (w *widgetBase) canContinueUpdateAfterHandlingErr(err error) bool {
    if err != nil {
        w.scheduleEarlyUpdate()               // 有错误 → 安排提前重试

        if !errors.Is(err, errPartialContent) {
            w.withError(err)                  // 全部失败：标记 Error，不渲染内容
            w.withNotice(nil)
            return false
        }

        w.withError(nil)                      // 部分失败：仅标记 Notice，仍渲染已有内容
        w.withNotice(err)
        return true
    }

    w.withNotice(nil)
    w.withError(nil)
    w.scheduleNextUpdate()                    // 全部成功：按正常 TTL 调度
    return true
}
```

### 2.2 指数退避算法

`scheduleEarlyUpdate()` 实现了基于重试次数的**平方退避**，见 [internal/glance/widget.go#L350-L367](internal/glance/widget.go#L350-L367)：

```go
func (w *widgetBase) scheduleEarlyUpdate() *widgetBase {
    w.updateRetriedTimes++
    if w.updateRetriedTimes > 5 {
        w.updateRetriedTimes = 5              // 重试次数上限封顶为 5
    }

    // 退避时间 = (重试次数)² 分钟
    nextEarlyUpdate := time.Now().Add(
        time.Duration(math.Pow(float64(w.updateRetriedTimes), 2)) * time.Minute,
    )
    nextUsualUpdate := w.getNextUpdateTime()

    // 取两者中更早的，避免退避时间超过正常 TTL
    if nextEarlyUpdate.After(nextUsualUpdate) {
        w.nextUpdate = nextUsualUpdate
    } else {
        w.nextUpdate = nextEarlyUpdate
    }

    return w
}
```

`scheduleNextUpdate()` 在成功时将重试计数清零，见 [internal/glance/widget.go#L343-L348](internal/glance/widget.go#L343-L348)：

```go
func (w *widgetBase) scheduleNextUpdate() *widgetBase {
    w.nextUpdate = w.getNextUpdateTime()
    w.updateRetriedTimes = 0
    return w
}
```

`getNextUpdateTime()` 根据缓存类型计算下次更新时间，见 [internal/glance/widget.go#L327-L341](internal/glance/widget.go#L327-L341)：

```go
func (w *widgetBase) getNextUpdateTime() time.Time {
    now := time.Now()
    if w.cacheType == cacheTypeDuration {
        return now.Add(w.cacheDuration)           // RSS：now + 2h
    }
    if w.cacheType == cacheTypeOnTheHour {
        return now.Add(time.Duration(
            ((60-now.Minute())*60)-now.Second(),
        ) * time.Second)
    }
    return time.Time{}
}
```

**退避时间表（RSS 默认 2 小时 TTL）：**

| 连续失败次数 | 退避间隔 | 实际下次更新时间 |
|-------------|---------|----------------|
| 第 1 次失败 | 1² = 1 分钟 | 约 1 分钟后 |
| 第 2 次失败 | 2² = 4 分钟 | 约 4 分钟后 |
| 第 3 次失败 | 3² = 9 分钟 | 约 9 分钟后 |
| 第 4 次失败 | 4² = 16 分钟 | 约 16 分钟后 |
| 第 5 次及以上 | 5² = 25 分钟 | 约 25 分钟后（封顶） |
| 任一时间若退避超过 2h | 以正常 TTL 为准 | 约 2 小时后 |

### 2.3 注意事项（来自代码 TODO）

代码注释说明了重试机制的当前局限，见 [internal/glance/widget.go#L294-L305](internal/glance/widget.go#L294-L305)：

- 部分内容 + 提前重试时，后续重试可能返回更少内容
- 未区分服务端错误（如 504 网关超时，应重试）和客户端限流（如 429 Too Many Requests，不应立即重试）
- 未来可能改为资源级缓存，仅重抓失败的源而非全部重跑

---

## 三、请求流程

### 3.1 请求入口

多源抓取从 `fetchItemsFromFeeds()` 函数启动，见 [internal/glance/widget-rss.go#L152-L190](internal/glance/widget-rss.go#L152-L190)：

```go
func (widget *rssWidget) fetchItemsFromFeeds() (rssFeedItemList, error) {
    requests := widget.FeedRequests
    job := newJob(widget.fetchItemsFromFeedTask, requests).withWorkers(30)
    feeds, errs, err := workerPoolDo(job)
    // ...
}
```

### 3.2 每个 RSS 源的请求配置

`rssFeedRequest` 结构体支持丰富的单源配置，见 [internal/glance/widget-rss.go#L131-L140](internal/glance/widget-rss.go#L131-L140)：

```go
type rssFeedRequest struct {
    URL             string            `yaml:"url"`              // RSS 源 URL
    Title           string            `yaml:"title"`            // 自定义源名称（覆盖 feed.Title）
    HideCategories  bool              `yaml:"hide-categories"`  // detailed-list 模式下隐藏分类
    HideDescription bool              `yaml:"hide-description"` // detailed-list 模式下隐藏描述
    Limit           int               `yaml:"limit"`            // 单源最多取多少条目
    ItemLinkPrefix  string            `yaml:"item-link-prefix"` // 给条目链接加前缀
    Headers         map[string]string `yaml:"headers"`          // 自定义 HTTP Header（鉴权等）
    IsDetailed      bool              `yaml:"-"`                // 是否详细模式（由 Widget.Style 决定）
}
```

### 3.3 HTTP 请求构建

每个 RSS 源的实际请求发生在 `fetchItemsFromFeedTask()`，见 [internal/glance/widget-rss.go#L192-L234](internal/glance/widget-rss.go#L192-L234)：

```go
func (widget *rssWidget) fetchItemsFromFeedTask(request rssFeedRequest) ([]rssFeedItem, error) {
    req, err := http.NewRequest("GET", request.URL, nil)
    if err != nil {
        return nil, err
    }

    req.Header.Add("User-Agent", glanceUserAgentString)

    // （此处插入 HTTP 缓存协商 Header，详见第四章）

    for key, value := range request.Headers {
        req.Header.Set(key, value)          // 用户自定义 Header 最后应用，可覆盖默认值
    }

    resp, err := defaultHTTPClient.Do(req)
    // ...
}
```

**关键点（均有直接代码支撑）：**
- User-Agent：`Glance/{version} +https://github.com/glanceapp/glance`，定义在 [internal/glance/widget-utils.go#L46](internal/glance/widget-utils.go#L46-L46)
- HTTP Client：使用包级 `defaultHTTPClient`，5 秒超时 + 每 Host 10 个空闲连接，定义在 [internal/glance/widget-utils.go#L24-L32](internal/glance/widget-utils.go#L24-L32)
- 自定义 Header 后设置，可覆盖默认的 User-Agent 等（代码顺序可见）

---

## 四、缓存协作机制（HTTP 级 ETag/Last-Modified）

这是 RSS Widget 内部针对**单个 RSS 源**的 HTTP 协商缓存，与第一章的组件级 TTL 缓存是两层独立且互补的缓存机制。

### 4.1 缓存数据结构

每个 Widget 实例维护一个内存缓存 `cachedFeeds`，见 [internal/glance/widget-rss.go#L45-L47](internal/glance/widget-rss.go#L45-L47) 和 [internal/glance/widget-rss.go#L114-L118](internal/glance/widget-rss.go#L114-L118)：

```go
type rssWidget struct {
    // ...
    cachedFeedsMutex sync.Mutex
    cachedFeeds      map[string]*cachedRSSFeed `yaml:"-"`
}

type cachedRSSFeed struct {
    etag         string           // HTTP ETag（实体标签）
    lastModified string           // HTTP Last-Modified（最后修改时间）
    items        []rssFeedItem    // 已解析好的条目列表
}
```

缓存以 `URL → *cachedRSSFeed` 为键值对存储，通过 `cachedFeedsMutex` 互斥锁保护并发读写（因为多个 Worker goroutine 可能同时访问）。

### 4.2 请求阶段：发送缓存协商 Header

请求前先查本地缓存并附带条件请求 Header，见 [internal/glance/widget-rss.go#L200-L210](internal/glance/widget-rss.go#L200-L210)：

```go
widget.cachedFeedsMutex.Lock()
cache, isCached := widget.cachedFeeds[request.URL]
if isCached {
    if cache.etag != "" {
        req.Header.Add("If-None-Match", cache.etag)
    }
    if cache.lastModified != "" {
        req.Header.Add("If-Modified-Since", cache.lastModified)
    }
}
widget.cachedFeedsMutex.Unlock()
```

**工作原理（可由代码直接证明）：**
- `If-None-Match`：当 `cache.etag` 非空时附加到请求
- `If-Modified-Since`：当 `cache.lastModified` 非空时附加到请求
- 两者独立存在，互不依赖

### 4.3 响应阶段：命中 304 Not Modified

收到响应后先检查状态码，见 [internal/glance/widget-rss.go#L222-L228](internal/glance/widget-rss.go#L222-L228)：

```go
if resp.StatusCode == http.StatusNotModified && isCached {
    return cache.items, nil           // 304 + 有本地缓存 → 直接复用
}

if resp.StatusCode != http.StatusOK {
    return nil, fmt.Errorf("unexpected status code %d from %s", resp.StatusCode, request.URL)
}
```

命中 304 时无需下载响应体、无需解析，直接返回已缓存的 `items`。注意两个条件必须**同时**满足：状态码为 304，且该 URL 有本地缓存（`isCached == true`）。

### 4.4 写入缓存

成功获取并解析新内容后，将 ETag/Last-Modified/items 三者一同回写缓存，见 [internal/glance/widget-rss.go#L333-L341](internal/glance/widget-rss.go#L333-L341)：

```go
if resp.Header.Get("ETag") != "" || resp.Header.Get("Last-Modified") != "" {
    widget.cachedFeedsMutex.Lock()
    widget.cachedFeeds[request.URL] = &cachedRSSFeed{
        etag:         resp.Header.Get("ETag"),
        lastModified: resp.Header.Get("Last-Modified"),
        items:        items,
    }
    widget.cachedFeedsMutex.Unlock()
}
```

**注意：** 只有当响应至少包含 `ETag` 或 `Last-Modified` 其中之一时才写入缓存——否则下一次无法发起条件请求。

### 4.5 两层缓存的协作关系

```
┌─────────────────────────────────────────────────────────────────┐
│                 第 1 层：组件级 TTL 缓存                         │
│  widgetBase.nextUpdate / cacheDuration（默认 2 小时）            │
│  控制 Widget.update() 是否执行（页面请求时判断 requiresUpdate） │
│  命中时直接复用内存中的 widget.Items                             │
└──────────────────────────────────────────────┬──────────────────┘
                                               │
                                     TTL 到期，执行 update()
                                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                 第 2 层：HTTP 协商缓存                            │
│  cachedFeeds[url] → ETag / Last-Modified / items                 │
│  控制单个 RSS 源是否真正下载（If-None-Match / 304 命中）         │
│  命中时跳过下载和解析，直接返回已缓存 items                        │
└─────────────────────────────────────────────────────────────────┘
```

**协作效果（全部可由代码直接支撑）：**
- 2 小时内页面刷新 → `requiresUpdate()` 返回 false，第 1 层拦截
- 超过 2 小时 → 触发 `update()`，第 2 层对每个源发协商请求，所有源都 304 时直接返回 `cache.items`
- 某源真正更新 → 仅该源下载 + 解析，其余源仍可能 304 命中

---

## 五、字符集处理与字段映射

### 5.1 读取原始字节并交给 gofeed

见 [internal/glance/widget-rss.go#L230-L238](internal/glance/widget-rss.go#L230-L238)：

```go
body, err := io.ReadAll(resp.Body)     // 以 []byte 读取原始响应体
if err != nil {
    return nil, err
}

feed, err := feedParser.ParseString(string(body))  // 转为 string 后交给 gofeed Parser
```

### 5.2 gofeed Parser 单例初始化

Parser 在包级别通过 `var` 声明初始化一次，被所有 Worker goroutine 共享调用，见 [internal/glance/widget-rss.go#L29](internal/glance/widget-rss.go#L29-L29)：

```go
var feedParser = gofeed.NewParser()
```

> **说明**：本项目代码仅能证明 Parser 为包级单例初始化且被并发使用。关于 gofeed 库内部的字符集自动检测、编码转换、线程安全等行为属于第三方库实现细节，不在本仓库代码范围内，本分析不做超出代码的推断。

### 5.3 单源 Limit 裁剪

在 gofeed 解析完成后、字段映射之前，会先按单源 `Limit` 裁剪条目数，见 [internal/glance/widget-rss.go#L240-L242](internal/glance/widget-rss.go#L240-L242)：

```go
if request.Limit > 0 && len(feed.Items) > request.Limit {
    feed.Items = feed.Items[:request.Limit]
}
```

这是**第一层裁剪**（单源级别）。最终所有源合并后还有**第二层裁剪**（Widget 全局 `widget.Limit`），见 [internal/glance/widget-rss.go#L91-L93](internal/glance/widget-rss.go#L91-L93)。

### 5.4 字段完整映射

`rssFeedItem` 结构体定义了 8 个字段，见 [internal/glance/widget-rss.go#L120-L129](internal/glance/widget-rss.go#L120-L129)：

```go
type rssFeedItem struct {
    ChannelName string        // 所属 RSS 源名称
    ChannelURL  string        // 所属 RSS 源主页 URL
    Title       string        // 条目标题
    Link        string        // 条目 URL
    ImageURL    string        // 缩略图 URL
    Categories  []string      // 分类标签（仅 detailed 模式）
    Description string        // 描述摘要（仅 detailed 模式）
    PublishedAt time.Time     // 发布时间
}
```

每个字段的映射逻辑如下：

**ChannelURL**：直接取自 `feed.Link`，见 [internal/glance/widget-rss.go#L249-L251](internal/glance/widget-rss.go#L249-L251)。

**ChannelName**：用户 `request.Title` 优先，否则用 `feed.Title`，见 [internal/glance/widget-rss.go#L306-L310](internal/glance/widget-rss.go#L306-L310)：

```go
if request.Title != "" {
    rssItem.ChannelName = request.Title
} else {
    rssItem.ChannelName = feed.Title
}
```

**Link**：三级回退补全，见 [internal/glance/widget-rss.go#L253-L274](internal/glance/widget-rss.go#L253-L274)：

```go
if request.ItemLinkPrefix != "" {
    rssItem.Link = request.ItemLinkPrefix + item.Link          // 1. 用户配置了前缀则直接拼接
} else if strings.HasPrefix(item.Link, "http://") || strings.HasPrefix(item.Link, "https://") {
    rssItem.Link = item.Link                                    // 2. 已是绝对 URL
} else {
    parsedUrl, err := url.Parse(feed.Link)                      // 3. 先尝试用 feed.Link 作为 base
    if err != nil {
        parsedUrl, err = url.Parse(request.URL)                 //    失败则用 RSS 源 URL 本身
    }
    if err == nil {
        var link string
        if len(item.Link) > 0 && item.Link[0] == '/' {
            link = item.Link
        } else {
            link = "/" + item.Link                              //    非 / 开头补 /
        }
        rssItem.Link = parsedUrl.Scheme + "://" + parsedUrl.Host + link
    }
}
```

若以上三种方式均失败（base URL 解析全部失败且无前缀且非绝对 URL），则 `rssItem.Link` 保持零值空字符串。

**Title**：`item.Title` 非空则用其反转义，否则用描述前 100 字符替代，见 [internal/glance/widget-rss.go#L276-L280](internal/glance/widget-rss.go#L276-L280)：

```go
if item.Title != "" {
    rssItem.Title = html.UnescapeString(item.Title)
} else {
    rssItem.Title = shortenFeedDescriptionLen(item.Description, 100)
}
```

**Description**：仅在三个条件同时满足时才填充——`request.IsDetailed`、`!request.HideDescription`、`item.Description != ""`、`item.Title != ""`，见 [internal/glance/widget-rss.go#L282-L285](internal/glance/widget-rss.go#L282-L285)：

```go
if request.IsDetailed {
    if !request.HideDescription && item.Description != "" && item.Title != "" {
        rssItem.Description = shortenFeedDescriptionLen(item.Description, 200)
    }
    // ...
}
```

描述最大长度 200 字符，经过完整的 HTML 清理流程（见下节）。

**Categories**：仅在 `request.IsDetailed && !request.HideCategories` 时填充。每条限制最多 6 个分类，每个分类字符串长度必须在 1~30 之间，见 [internal/glance/widget-rss.go#L287-L303](internal/glance/widget-rss.go#L287-L303)：

```go
if !request.HideCategories {
    var categories = make([]string, 0, 6)
    for _, category := range item.Categories {
        if len(categories) == 6 {
            break
        }
        if len(category) == 0 || len(category) > 30 {
            continue
        }
        categories = append(categories, category)
    }
    rssItem.Categories = categories
}
```

**ImageURL**：三级回退查找，见 [internal/glance/widget-rss.go#L312-L322](internal/glance/widget-rss.go#L312-L322)：

```go
if item.Image != nil {
    rssItem.ImageURL = item.Image.URL                           // 1. 条目自带的 image 字段
} else if url := findThumbnailInItemExtensions(item); url != "" {
    rssItem.ImageURL = url                                      // 2. media:thumbnail / media:image 扩展
} else if feed.Image != nil {
    // 3. feed 级别的 image，处理相对路径
    if len(feed.Image.URL) > 0 && feed.Image.URL[0] == '/' {
        rssItem.ImageURL = strings.TrimRight(feed.Link, "/") + feed.Image.URL
    } else {
        rssItem.ImageURL = feed.Image.URL
    }
}
```

`findThumbnailInItemExtensions()` 递归扫描 `media` 命名空间下的扩展节点，见 [internal/glance/widget-rss.go#L346-L374](internal/glance/widget-rss.go#L346-L374)。

**PublishedAt**：`item.PublishedParsed` 非 nil 则用其解析值，否则回退为当前时间 `time.Now()`，见 [internal/glance/widget-rss.go#L324-L328](internal/glance/widget-rss.go#L324-L328)：

```go
if item.PublishedParsed != nil {
    rssItem.PublishedAt = *item.PublishedParsed
} else {
    rssItem.PublishedAt = time.Now()
}
```

### 5.5 HTML 实体转义与标签清理

描述的清理流程在 `sanitizeFeedDescription()` 中，见 [internal/glance/widget-rss.go#L376-L402](internal/glance/widget-rss.go#L376-L402)：

```go
var htmlTagsWithAttributesPattern = regexp.MustCompile(
    `<\/?[a-zA-Z0-9-]+ *(?:[a-zA-Z-]+=(?:"|').*?(?:"|') ?)* *\/?>`,
)

func sanitizeFeedDescription(description string) string {
    if description == "" {
        return ""
    }
    description = strings.ReplaceAll(description, "\n", " ")
    description = htmlTagsWithAttributesPattern.ReplaceAllString(description, "")
    description = sequentialWhitespacePattern.ReplaceAllString(description, " ")  // \s+ → 单个空格
    description = strings.TrimSpace(description)
    description = html.UnescapeString(description)
    return description
}

func shortenFeedDescriptionLen(description string, maxLen int) string {
    description, _ = limitStringLength(description, 1000)   // 先限制到 1000 字符防超长
    description = sanitizeFeedDescription(description)
    description, limited := limitStringLength(description, maxLen)
    if limited {
        description += "…"
    }
    return description
}
```

`sequentialWhitespacePattern` 定义在 [internal/glance/utils.go#L17](internal/glance/utils.go#L17-L17)，`limitStringLength` 定义在 [internal/glance/utils.go#L112](internal/glance/utils.go#L112-L112)。

---

## 六、去重与排序

### 6.1 基于 Link 的去重

所有源抓取完成后统一去重，见 [internal/glance/widget-rss.go#L163-L179](internal/glance/widget-rss.go#L163-L179)：

```go
seen := make(map[string]struct{})

for i := range feeds {
    if errs[i] != nil {
        continue
    }

    for _, item := range feeds[i] {
        if _, exists := seen[item.Link]; exists {
            continue                              // Link 已出现过 → 跳过
        }
        entries = append(entries, item)
        seen[item.Link] = struct{}{}              // 标记已见
    }
}
```

**设计要点（全部可由代码支撑）：**
- 使用 `map[string]struct{}` 作为集合，空结构体零值不占额外内存（仅 map 开销）
- 以条目最终 URL（`item.Link`，已在字段映射阶段补全为绝对路径或拼接前缀）作为唯一标识
- 不同 RSS 源若出现相同 URL，仅保留**首次出现**的条目
- 去重发生在所有源抓取完成之后、合并结果阶段

### 6.2 去重与顺序

- 去重保留首次出现：遍历 `feeds` 顺序即用户配置的源顺序（YAML `feeds:` 列表顺序）
- `feeds[i]` 在 Worker Pool 中按 `index` 回填，与输入顺序严格一致，见 [internal/glance/widget-utils.go#L235-L240](internal/glance/widget-utils.go#L235-L240)

### 6.3 排序逻辑

若未设置 `PreserveOrder`（零值为 `false`，即默认排序），则在去重合并后按发布时间降序排序，见 [internal/glance/widget-rss.go#L87-L89](internal/glance/widget-rss.go#L87-L89) 和 [internal/glance/widget-rss.go#L144-L150](internal/glance/widget-rss.go#L144-L150)：

```go
if !widget.PreserveOrder {
    items.sortByNewest()
}
```

```go
func (f rssFeedItemList) sortByNewest() rssFeedItemList {
    sort.Slice(f, func(i, j int) bool {
        return f[i].PublishedAt.After(f[j].PublishedAt)
    })

    return f
}
```

**关于排序的代码事实（可由代码直接证明）：**
- 使用 Go 标准库 `sort.Slice`，**该排序是不稳定的**（相同 `PublishedAt` 的条目之间相对顺序不保证保留）
- 比较规则：`f[i].PublishedAt.After(f[j].PublishedAt)`，即发布时间**更新者**排在前面（降序）
- 排序是**原地修改** slice，同时 receiver 返回自身以支持链式调用；调用方也可忽略返回值
- `PreserveOrder` 字段零值为 `false`，即默认会执行排序
- `PublishedAt` 为零值的条目在实际数据中不会出现——字段映射阶段已保证要么取 `PublishedParsed`，要么取 `time.Now()`

### 6.4 全局 Limit 裁剪

排序后若条目数超过 `widget.Limit`（默认 25），则按排序后的顺序裁剪前 N 条，见 [internal/glance/widget-rss.go#L91-L93](internal/glance/widget-rss.go#L91-L93)：

```go
if len(items) > widget.Limit {
    items = items[:widget.Limit]
}
```

这是**第二层裁剪**（Widget 全局），与 5.3 节的**单源裁剪**互相独立。

---

## 七、并发限制的四个层级

整个系统存在四层并发/连接限制，层层嵌套：

```
Level 1: Page 级 Widget 并发   ──── 无主动上限（但 page.mu 确保同 Page 串行）
           │
           ▼
Level 2: RSS Worker Pool       ──── 最多 30 个 goroutine 并发抓取
           │
           ▼
Level 3: HTTP Transport 连接池  ──── 每 Host 最多 10 个空闲 Keep-Alive 连接
           │
           ▼
Level 4: HTTP 请求超时         ──── 单次请求 5 秒超时
```

### 7.1 Level 1：Page 级 Widget 并发更新

见 [internal/glance/glance.go#L233-L270](internal/glance/glance.go#L233-L270)：

```go
func (p *page) updateOutdatedWidgets() {
    var wg sync.WaitGroup
    // ...
    for w := range p.HeadWidgets {
        if !widget.requiresUpdate(&now) { continue }
        wg.Add(1)
        go func() {
            defer wg.Done()
            widget.update(context)
        }()
    }
    // Columns 同理
    wg.Wait()
}
```

**可由代码直接证明的事实：**
- 一个页面有多少个过期 Widget，就启动多少个 goroutine（代码中无 goroutine 数量上限的主动限制）
- 受 `page.mu` 互斥锁保护（见 [internal/glance/config.go#L77-L92](internal/glance/config.go#L77-L92)），同一 Page 同一时间只有一个 `updateOutdatedWidgets()` 在运行
- 容器类 Widget（group/split-column）内部也有相同的并发逻辑，见 [internal/glance/widget-container.go#L23-L42](internal/glance/widget-container.go#L23-L42)

### 7.2 Level 2：RSS Widget 内部 Worker Pool

Worker Pool 采用泛型实现，定义在 [internal/glance/widget-utils.go#L141-L242](internal/glance/widget-utils.go#L141-L242)。RSS Widget 固定配置 **30 个 Worker**，见 [internal/glance/widget-rss.go#L155-L156](internal/glance/widget-rss.go#L155-L156)：

```go
job := newJob(widget.fetchItemsFromFeedTask, requests).withWorkers(30)
```

核心数据结构（见 [internal/glance/widget-utils.go#L141-L155](internal/glance/widget-utils.go#L141-L155)）：

```go
type workerPoolTask[I any, O any] struct {
    index  int      // 保持结果与输入顺序对应
    input  I
    output O
    err    error
}

type workerPoolJob[I any, O any] struct {
    data    []I
    workers int
    task    func(I) (O, error)
    ctx     context.Context
}
```

`withWorkers()` 会做 `min(workers, len(data))` 裁剪，见 [internal/glance/widget-utils.go#L157-L165](internal/glance/widget-utils.go#L157-L165)：

```go
func (job *workerPoolJob[I, O]) withWorkers(workers int) *workerPoolJob[I, O] {
    if workers == 0 {
        job.workers = defaultNumWorkers       // 默认 10
    } else {
        job.workers = min(workers, len(job.data))  // 源数 < 30 时按源数启动
    }
    return job
}
```

`workerPoolDo()` 核心流程见 [internal/glance/widget-utils.go#L184-L242](internal/glance/widget-utils.go#L184-L242)：

```
主 goroutine → 创建 tasksQueue / resultsQueue
     │
     ├─ 启动 N 个 Worker goroutine：for t := range tasksQueue { 执行任务 → resultsQueue }
     │
     ├─ 独立 goroutine：遍历输入发 tasksQueue → close(tasksQueue) → wg.Wait() → close(resultsQueue)
     │
     └─ 主 goroutine：for task := range resultsQueue { 按 task.index 回填到 results / errs }
```

**单任务优化**：源数 = 1 时直接同步执行，完全跳过 channel 与 goroutine 开销，见 [internal/glance/widget-utils.go#L192-L195](internal/glance/widget-utils.go#L192-L195)：

```go
if len(job.data) == 1 {
    results[0], errs[0] = job.task(job.data[0])
    return results, errs, nil
}
```

### 7.3 Level 3：HTTP Transport 连接池

`defaultHTTPClient` 的显式配置，见 [internal/glance/widget-utils.go#L24-L32](internal/glance/widget-utils.go#L24-L32)：

```go
const defaultClientTimeout = 5 * time.Second

var defaultHTTPClient = &http.Client{
    Transport: &http.Transport{
        MaxIdleConnsPerHost: 10,              // 每 Host 最多保留 10 个空闲持久连接
        Proxy:               http.ProxyFromEnvironment,
    },
    Timeout: defaultClientTimeout,            // 全流程超时（连接 + 重定向 + 读响应体）
}
```

**本项目代码中显式设置的参数**：
| 参数 | 显式设置值 | 代码位置 |
|-----|-----------|---------|
| `Transport.MaxIdleConnsPerHost` | 10 | [internal/glance/widget-utils.go#L28](internal/glance/widget-utils.go#L28-L28) |
| `Transport.Proxy` | `http.ProxyFromEnvironment` | [internal/glance/widget-utils.go#L29](internal/glance/widget-utils.go#L29-L29) |
| `Timeout` | 5 秒 | [internal/glance/widget-utils.go#L31](internal/glance/widget-utils.go#L31-L31) |

**未显式设置的参数**（如 `MaxConnsPerHost`、`MaxIdleConns`、`IdleConnTimeout`、`TLSHandshakeTimeout` 等）使用 Go 标准库 `http.Transport` 的默认值，具体默认行为属于标准库范畴，不在本仓库分析范围。

另有一个不校验证书的客户端 `defaultInsecureHTTPClient`，RSS Widget 代码中未使用，见 [internal/glance/widget-utils.go#L34-L40](internal/glance/widget-utils.go#L34-L40)。

### 7.4 Level 4：HTTP 请求超时

- 单次 HTTP 请求全流程超时 **5 秒**（`defaultClientTimeout`），由 `http.Client.Timeout` 控制，包含 DNS 解析、TCP 握手、TLS 握手、重定向、读取响应体
- 超时后 `http.Client.Do()` 返回错误，该 RSS 源计入 `failed`，触发 Level 1 的指数退避重试调度

### 7.5 并发层级汇总表

| 层级 | 位置 | 限制值（代码可证） | 控制对象 |
|-----|------|-------|---------|
| L1 Page 级 | [internal/glance/glance.go#L233-L270](internal/glance/glance.go#L233-L270) | 无主动上限（page.mu 串行化同 Page 更新） | 同一 Page 内多个 Widget 的并发更新 |
| L2 Worker Pool | [internal/glance/widget-rss.go#L155-L156](internal/glance/widget-rss.go#L155-L156) | 30 goroutine（实际 `min(30, 源数)`，单源同步优化） | 单个 RSS Widget 内多个源的并发抓取 |
| L3 连接池 | [internal/glance/widget-utils.go#L24-L32](internal/glance/widget-utils.go#L24-L32) | 每 Host 10 个空闲连接（显式设置） | TCP 连接复用（HTTP Keep-Alive） |
| L4 超时 | [internal/glance/widget-utils.go#L24](internal/glance/widget-utils.go#L24-L24) | 5 秒 | 单次 HTTP 请求的最长耗时 |

---

## 八、完整数据流时序

```
Browser → GET /api/pages/{page}/content/
    │
    ├─ page.mu.Lock()                        （同一 Page 串行化）
    ├─ page.updateOutdatedWidgets()
    │      │
    │      ├─ now := time.Now()
    │      ├─ 遍历所有 Widget
    │      │    ├─ widget.requiresUpdate(&now)
    │      │    │    └─ cacheTypeInfinite? / nextUpdate.IsZero()? / now.After(nextUpdate)?
    │      │    └─ 需要更新的每个 Widget 起一个 goroutine
    │      │         └─ [Goroutine] rssWidget.update(ctx)
    │      │                │
    │      │                ├─ widget.fetchItemsFromFeeds()
    │      │                │      │
    │      │                │      ├─ requests := widget.FeedRequests
    │      │                │      ├─ job := newJob(fetchItemsFromFeedTask, requests).withWorkers(30)
    │      │                │      └─ workerPoolDo(job)
    │      │                │             │
    │      │                │             ├─ 单任务优化（len==1 同步执行）或启动 N 个 Worker
    │      │                │             │
    │      │                │             ├─ [Worker k] fetchItemsFromFeedTask(requests[k])
    │      │                │             │      ├─ cachedFeedsMutex.Lock()
    │      │                │             │      ├─ cache, isCached := cachedFeeds[url]
    │      │                │             │      ├─ isCached? → 附加 If-None-Match / If-Modified-Since
    │      │                │             │      ├─ cachedFeedsMutex.Unlock()
    │      │                │             │      ├─ 应用用户自定义 Headers
    │      │                │             │      ├─ defaultHTTPClient.Do(req)  (L4: 5s 超时)
    │      │                │             │      ├─ 304 Not Modified && isCached?
    │      │                │             │      │    └─ 是 → 直接 return cache.items
    │      │                │             │      ├─ 200 OK?
    │      │                │             │      │    ├─ io.ReadAll(resp.Body) → []byte
    │      │                │             │      │    ├─ feedParser.ParseString()
    │      │                │             │      │    ├─ per-source Limit 裁剪
    │      │                │             │      │    ├─ 字段映射（ChannelURL/ChannelName/Title/Link/
    │      │                │             │      │    │              Categories/Description/ImageURL/PublishedAt）
    │      │                │             │      │    ├─ 响应有 ETag 或 Last-Modified?
    │      │                │             │      │    │    └─ 是 → cachedFeeds[url] = {etag, lastMod, items}
    │      │                │             │      │    └─ return items
    │      │                │             │      └─ 其他状态码 / 错误 → return error
    │      │                │             │
    │      │                │             ├─ 合并所有 feeds[k] 与 errs[k]
    │      │                │             ├─ seen map 去重 (按 item.Link)
    │      │                │             ├─ failed 统计 → errNoContent / errPartialContent / nil
    │      │                │             └─ return entries, err
    │      │                │
    │      │                ├─ !PreserveOrder? → items.sortByNewest() (sort.Slice 降序，不稳定)
    │      │                ├─ len(items) > Limit? → 全局裁剪
    │      │                ├─ widget.Items = items
    │      │                │
    │      │                └─ widget.canContinueUpdateAfterHandlingErr(err)
    │      │                     ├─ err == nil
    │      │                     │    ├─ withError(nil) / withNotice(nil)
    │      │                     │    └─ scheduleNextUpdate() → nextUpdate=now+2h, retriedTimes=0
    │      │                     ├─ errors.Is(err, errPartialContent)
    │      │                     │    ├─ withError(nil) / withNotice(err)
    │      │                     │    └─ scheduleEarlyUpdate() → 退避 n² 分钟
    │      │                     └─ 否则 (errNoContent 等)
    │      │                          ├─ withError(err) / withNotice(nil)
    │      │                          └─ scheduleEarlyUpdate() → 退避 n² 分钟
    │      │
    │      └─ wg.Wait()                      （等待所有 Widget goroutine 完成）
    │
    ├─ page.mu.Unlock()
    └─ 渲染 pageContentTemplate → 返回 HTML
```

---

## 九、关键文件索引

| 功能模块 | 文件与位置 |
|----------|-----------|
| RSS Widget 主体（抓取/解析/去重/HTTP 缓存/排序） | [internal/glance/widget-rss.go](internal/glance/widget-rss.go) |
| Widget 基类（TTL 缓存/重试调度/错误处理） | [internal/glance/widget.go](internal/glance/widget.go) |
| Worker Pool / HTTP Client / User-Agent / 通用错误 | [internal/glance/widget-utils.go](internal/glance/widget-utils.go) |
| Page 级 Widget 更新调度 | [internal/glance/glance.go#L233-L270](internal/glance/glance.go#L233-L270) |
| Page 内容 API（含 page.mu 加锁） | [internal/glance/glance.go#L334-L367](internal/glance/glance.go#L334-L367) |
| Page 结构体与互斥锁 | [internal/glance/config.go#L77-L92](internal/glance/config.go#L77-L92) |
| durationField（自定义 YAML 时长解析） | [internal/glance/config-fields.go#L96-L130](internal/glance/config-fields.go#L96-L130) |
| 容器类 Widget 并发更新 | [internal/glance/widget-container.go](internal/glance/widget-container.go) |
| 通用 Singleflight 工具 | [internal/glance/singleflight.go](internal/glance/singleflight.go) |
| 通用正则与工具（空白合并、长度截断） | [internal/glance/utils.go](internal/glance/utils.go) |
| 组件级缓存调度逻辑完整 | [internal/glance/widget.go#L173-L367](internal/glance/widget.go#L173-L367) |
| RSS HTTP 协商缓存数据结构 | [internal/glance/widget-rss.go#L114-L118](internal/glance/widget-rss.go#L114-L118) |
| RSS HTTP 协商缓存读（请求阶段） | [internal/glance/widget-rss.go#L200-L210](internal/glance/widget-rss.go#L200-L210) |
| RSS HTTP 协商缓存命中判断 | [internal/glance/widget-rss.go#L222-L224](internal/glance/widget-rss.go#L222-L224) |
| RSS HTTP 协商缓存写（成功后） | [internal/glance/widget-rss.go#L333-L341](internal/glance/widget-rss.go#L333-L341) |
| RSS 去重逻辑 | [internal/glance/widget-rss.go#L163-L179](internal/glance/widget-rss.go#L163-L179) |
| RSS 排序逻辑（sort.Slice，不稳定） | [internal/glance/widget-rss.go#L144-L150](internal/glance/widget-rss.go#L144-L150) |
| gofeed Parser 包级单例初始化 | [internal/glance/widget-rss.go#L29](internal/glance/widget-rss.go#L29-L29) |
| 字符集读取与 gofeed 调用 | [internal/glance/widget-rss.go#L230-L238](internal/glance/widget-rss.go#L230-L238) |
| 单源 Limit 裁剪 | [internal/glance/widget-rss.go#L240-L242](internal/glance/widget-rss.go#L240-L242) |
| 全局 Limit 裁剪 | [internal/glance/widget-rss.go#L91-L93](internal/glance/widget-rss.go#L91-L93) |
| 完整字段映射（标题/链接/分类/描述/时间/缩略图） | [internal/glance/widget-rss.go#L249-L330](internal/glance/widget-rss.go#L249-L330) |
| HTML 标签剥离与描述清理 | [internal/glance/widget-rss.go#L376-L402](internal/glance/widget-rss.go#L376-L402) |
| RSS 30 Worker 配置 | [internal/glance/widget-rss.go#L155-L156](internal/glance/widget-rss.go#L155-L156) |
| HTTP 连接池与超时配置 | [internal/glance/widget-utils.go#L24-L32](internal/glance/widget-utils.go#L24-L32) |
| Worker Pool 泛型实现（含单任务优化） | [internal/glance/widget-utils.go#L141-L242](internal/glance/widget-utils.go#L141-L242) |
