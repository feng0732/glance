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

定义在 `widgetBase` 结构体中，见 [internal/glance/widget.go#L141-L167](internal/glance/widget.go#L141-L167)：

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
    Title   string   `yaml:"name"`
    // ...
    mu      sync.Mutex `yaml:"-"`
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

`withError()` 还会在无错误时将 `ContentAvailable` 置为 `true`，见 [internal/glance/widget.go#L283-L291](internal/glance/widget.go#L283-L291)。

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

`getNextUpdateTime()` 根据缓存类型计算，见 [internal/glance/widget.go#L327-L341](internal/glance/widget.go#L327-L341)：

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

**关键点：**
- User-Agent：`Glance/{version} +https://github.com/glanceapp/glance`，定义在 [internal/glance/widget-utils.go#L46](internal/glance/widget-utils.go#L46-L46)
- HTTP Client：使用包级 `defaultHTTPClient`，5 秒超时 + 每 Host 10 个空闲连接，定义在 [internal/glance/widget-utils.go#L24-L32](internal/glance/widget-utils.go#L24-L32)
- 自定义 Header 后设置，可覆盖默认的 User-Agent 等

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

**工作原理：**
- `If-None-Match` + ETag：服务器比较 ETag，内容未变则返回 `304 Not Modified`
- `If-Modified-Since` + Last-Modified：服务器比较时间戳，文件未改动则返回 `304`
- 两者可同时存在，ETag 优先级通常更高

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

命中 304 时无需下载响应体、无需解析，直接返回已缓存的 `items`，几乎零成本。

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

**协作效果：**
- 2 小时内页面刷新 → 第 1 层拦截，零开销
- 超过 2 小时 → 第 2 层对每个源发协商请求，所有源都 304 时几乎零成本
- 某源真正更新 → 仅该源下载 + 解析，其余源仍可能 304 命中

---

## 五、字符集处理与字段映射

### 5.1 读取原始字节

见 [internal/glance/widget-rss.go#L230-L238](internal/glance/widget-rss.go#L230-L238)：

```go
body, err := io.ReadAll(resp.Body)     // 以 []byte 读取原始响应体
if err != nil {
    return nil, err
}

feed, err := feedParser.ParseString(string(body))  // 转为 string 后交给 gofeed
```

### 5.2 gofeed 自动字符集检测

项目使用 `github.com/mmcdole/gofeed` 库，Parser 在包级别初始化一次（线程安全），见 [internal/glance/widget-rss.go#L29](internal/glance/widget-rss.go#L29-L29)：

```go
var feedParser = gofeed.NewParser()
```

`gofeed.Parser.ParseString()` 内部处理逻辑：
1. 检测 XML 声明中的 `encoding` 属性（如 `<?xml version="1.0" encoding="GB2312"?>`）
2. 若 XML 声明缺失或声明编码无法识别，使用 `golang.org/x/net/html/charset.DetermineEncoding` 自动嗅探
3. 通过 `x/text/encoding` 将源编码字节流转换为 UTF-8
4. 再交给标准库 `encoding/xml` 解析
5. 支持 GBK、GB2312、ISO-8859-1、Shift_JIS、UTF-8 等常见编码

### 5.3 HTML 实体转义与标签清理

标题和描述做二次处理。

**标题**：直接反转义 HTML 实体，见 [internal/glance/widget-rss.go#L276-L280](internal/glance/widget-rss.go#L276-L280)：

```go
if item.Title != "" {
    rssItem.Title = html.UnescapeString(item.Title)
} else {
    rssItem.Title = shortenFeedDescriptionLen(item.Description, 100)  // 无标题则用描述前 100 字符
}
```

**描述**：经过更复杂的清理流程 `shortenFeedDescriptionLen()`，见 [internal/glance/widget-rss.go#L376-L402](internal/glance/widget-rss.go#L376-L402)：

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

`sequentialWhitespacePattern` 定义在 [internal/glance/utils.go#L17](internal/glance/utils.go#L17-L17)，`limitStringLength` 在 [internal/glance/utils.go#L112](internal/glance/utils.go#L112-L112)。

### 5.4 链接补全与缩略图查找

RSS 条目中的链接可能是相对路径，代码做了三级回退补全，见 [internal/glance/widget-rss.go#L253-L274](internal/glance/widget-rss.go#L253-L274)：

```go
if request.ItemLinkPrefix != "" {
    rssItem.Link = request.ItemLinkPrefix + item.Link          // 用户配置了前缀则直接拼接
} else if strings.HasPrefix(item.Link, "http://") || strings.HasPrefix(item.Link, "https://") {
    rssItem.Link = item.Link                                    // 已是绝对 URL
} else {
    parsedUrl, err := url.Parse(feed.Link)                      // 先尝试用 feed.Link 作为 base
    if err != nil {
        parsedUrl, err = url.Parse(request.URL)                 // 失败则用 RSS 源 URL 本身
    }
    if err == nil {
        var link string
        if len(item.Link) > 0 && item.Link[0] == '/' {
            link = item.Link
        } else {
            link = "/" + item.Link                              // 非 / 开头补 /
        }
        rssItem.Link = parsedUrl.Scheme + "://" + parsedUrl.Host + link
    }
}
```

缩略图查找三级回退，见 [internal/glance/widget-rss.go#L312-L322](internal/glance/widget-rss.go#L312-L322)：

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

---

## 六、去重机制

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

**设计要点：**
- 使用 `map[string]struct{}` 作为集合，空结构体零值不占额外内存（仅 map 开销）
- 以条目最终 URL（`item.Link`，已补全为绝对路径）作为唯一标识
- 不同 RSS 源若出现相同 URL，仅保留**首次出现**的条目
- 去重发生在所有源抓取完成之后、合并结果阶段

### 6.2 去重与顺序

- 去重保留首次出现：遍历 `feeds` 顺序即用户配置的源顺序（`yaml:"feeds"` 顺序）
- 之后可选择按发布时间重新排序（`PreserveOrder: false` 时默认行为），见 [internal/glance/widget-rss.go#L87-L96](internal/glance/widget-rss.go#L87-L96)：

```go
if !widget.PreserveOrder {
    items.sortByNewest()
}
if len(items) > widget.Limit {
    items = items[:widget.Limit]
}
widget.Items = items
```

`sortByNewest()` 使用标准库稳定排序按 `PublishedAt` 降序，见 [internal/glance/widget-rss.go#L144-L150](internal/glance/widget-rss.go#L144-L150)。

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

**特点：**
- 一个页面有多少个过期 Widget，就启动多少个 goroutine（**无主动上限**）
- 受 `page.mu` 互斥锁保护（见 [internal/glance/config.go#L77-L92](internal/glance/config.go#L77-L92)），同一 Page 同一时间只有一个 `updateOutdatedWidgets()` 在运行
- 容器类 Widget（group/split-column）内部也有相同的并发逻辑，见 [internal/glance/widget-container.go#L23-L42](internal/glance/widget-container.go#L23-L42)

### 7.2 Level 2：RSS Widget 内部 Worker Pool

Worker Pool 采用泛型实现，定义在 [internal/glance/widget-utils.go#L141-L242](internal/glance/widget-utils.go#L141-L242)。RSS Widget 固定配置 **30 个 Worker**，见 [internal/glance/widget-rss.go#L155-L156](internal/glance/widget-rss.go#L155-L156)：

```go
job := newJob(widget.fetchItemsFromFeedTask, requests).withWorkers(30)
```

核心数据结构：

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
     └─ 主 goroutine：for task := range resultsQueue { 按 index 回填到 results / errs }
```

**单任务优化**：源数 = 1 时直接同步执行，完全跳过 channel 开销，见 [internal/glance/widget-utils.go#L192-L195](internal/glance/widget-utils.go#L192-L195)：

```go
if len(job.data) == 1 {
    results[0], errs[0] = job.task(job.data[0])
    return results, errs, nil
}
```

### 7.3 Level 3：HTTP Transport 连接池

`defaultHTTPClient` 的配置，见 [internal/glance/widget-utils.go#L24-L32](internal/glance/widget-utils.go#L24-L32)：

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

**参数详解（Go `http.Transport` 默认行为）：**
| 参数 | RSS 配置值 | 说明 |
|-----|-----------|------|
| `MaxIdleConnsPerHost` | **10** | 单个目标主机最多保留 10 个空闲 Keep-Alive 连接 |
| `MaxConnsPerHost` | 未设置（0） | 每个 Host 的并发活跃连接数无硬限制 |
| `MaxIdleConns` | 未设置（0） | 全局空闲连接无上限 |
| `IdleConnTimeout` | 未设置（90s 默认） | 空闲连接 90 秒后自动关闭 |
| `Proxy` | `http.ProxyFromEnvironment` | 支持 `HTTP_PROXY` / `HTTPS_PROXY` 环境变量 |

**与 Level 2 的交互：**
- RSS Worker Pool 最多 30 个并发 HTTP 请求
- 若多个源指向同一 Host（如多个 `reddit.com` RSS），该 Host 可能瞬时建立远超 10 个活跃 TCP 连接（10 仅限制空闲保留数）
- 实际瓶颈更多来自 Level 2 的 30 Worker 上限和 Level 4 的 5 秒超时

另有一个不校验证书的客户端 `defaultInsecureHTTPClient`，RSS Widget 未使用，见 [internal/glance/widget-utils.go#L34-L40](internal/glance/widget-utils.go#L34-L40)。

### 7.4 Level 4：HTTP 请求超时

- 单次 HTTP 请求全流程超时 **5 秒**（`defaultClientTimeout`），包含 DNS 解析、TCP 握手、TLS 握手、重定向、读取响应体
- 超时后 `http.Client.Do()` 返回 `context.DeadlineExceeded` 或 `net.Error` Timeout，该 RSS 源计入 `failed`，触发 Level 1 的指数退避重试

### 7.5 并发层级汇总表

| 层级 | 位置 | 限制值 | 控制对象 |
|-----|------|-------|---------|
| L1 Page 级 | [internal/glance/glance.go#L233-L270](internal/glance/glance.go#L233-L270) | 无主动上限（page.mu 串行化） | 同一 Page 内多个 Widget 的并发更新 |
| L2 Worker Pool | [internal/glance/widget-rss.go#L155-L156](internal/glance/widget-rss.go#L155-L156) | 30 goroutine（min(30, 源数)） | 单个 RSS Widget 内多个源的并发抓取 |
| L3 连接池 | [internal/glance/widget-utils.go#L24-L32](internal/glance/widget-utils.go#L24-L32) | 每 Host 10 个空闲连接 | TCP 连接复用（HTTP Keep-Alive） |
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
    │      │                │             │      │    ├─ feedParser.ParseString()  (自动字符集)
    │      │                │             │      │    ├─ per-source Limit 裁剪
    │      │                │             │      │    ├─ 字段映射（标题/链接/描述/分类/时间/缩略图）
    │      │                │             │      │    ├─ 响应有 ETag/Last-Modified?
    │      │                │             │      │    │    └─ 是 → cachedFeeds[url] = {etag, lastMod, items}
    │      │                │             │      │    └─ return items
    │      │                │             │      └─ 其他状态码 → return error
    │      │                │             │
    │      │                │             ├─ 合并所有 feeds[k] 与 errs[k]
    │      │                │             ├─ seen map 去重 (按 item.Link)
    │      │                │             ├─ failed 统计 → errNoContent / errPartialContent / nil
    │      │                │             └─ return entries, err
    │      │                │
    │      │                ├─ !PreserveOrder? → items.sortByNewest()
    │      │                ├─ len(items) > Limit? → 裁剪
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
| RSS Widget 主体（抓取/解析/去重/HTTP 缓存） | [internal/glance/widget-rss.go](internal/glance/widget-rss.go) |
| Widget 基类（TTL 缓存/重试调度/错误处理） | [internal/glance/widget.go](internal/glance/widget.go) |
| Worker Pool / HTTP Client / User-Agent | [internal/glance/widget-utils.go](internal/glance/widget-utils.go) |
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
| 字符集读取与 gofeed 调用 | [internal/glance/widget-rss.go#L230-L238](internal/glance/widget-rss.go#L230-L238) |
| HTML 标签剥离与描述清理 | [internal/glance/widget-rss.go#L376-L402](internal/glance/widget-rss.go#L376-L402) |
| 链接补全与缩略图查找 | [internal/glance/widget-rss.go#L253-L322](internal/glance/widget-rss.go#L253-L322) |
| RSS 30 Worker 配置 | [internal/glance/widget-rss.go#L155-L156](internal/glance/widget-rss.go#L155-L156) |
| HTTP 连接池与超时配置 | [internal/glance/widget-utils.go#L24-L32](internal/glance/widget-utils.go#L24-L32) |
| Worker Pool 泛型实现 | [internal/glance/widget-utils.go#L141-L242](internal/glance/widget-utils.go#L141-L242) |
