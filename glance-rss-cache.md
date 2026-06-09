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

RSS Widget 并不是每次页面请求都重新抓取，而是由组件级缓存机制控制更新频率。

### 1.1 缓存配置字段

定义在 `widgetBase` 结构体中，见 [internal/glance/widget.go:149](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget.go#L149-L167)：

```go
type widgetBase struct {
    CustomCacheDuration durationField    `yaml:"cache"`     // 用户可配置的缓存时长
    cacheDuration       time.Duration    `yaml:"-"`         // 实际生效的缓存时长
    cacheType           cacheType        `yaml:"-"`         // 缓存类型（按时长 / 整点 / 无限）
    nextUpdate          time.Time        `yaml:"-"`         // 下次允许更新的时间点
    updateRetriedTimes  int              `yaml:"-"`         // 已重试次数（用于退避）
    ContentAvailable    bool             `yaml:"-"`         // 是否已有有效内容
    Error               error            `yaml:"-"`         // 最近一次错误
    Notice              error            `yaml:"-"`         // 提示性错误（如部分内容缺失）
}
```

### 1.2 RSS Widget 的默认缓存配置

在 `initialize()` 中设置默认 2 小时缓存，见 [internal/glance/widget-rss.go:49](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L49-L78)：

```go
func (widget *rssWidget) initialize() error {
    widget.withTitle("RSS Feed").withCacheDuration(2 * time.Hour)
    // ...
}
```

`withCacheDuration()` 的逻辑见 [internal/glance/widget.go:259](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget.go#L259-L269)：

```go
func (w *widgetBase) withCacheDuration(duration time.Duration) *widgetBase {
    w.cacheType = cacheTypeDuration

    if duration == -1 || w.CustomCacheDuration == 0 {
        w.cacheDuration = duration  // 使用默认值（RSS 为 2h）
    } else {
        w.cacheDuration = time.Duration(w.CustomCacheDuration)  // 用户配置优先
    }

    return w
}
```

**优先级：** 用户 YAML `cache:` 字段 > Widget 默认值（2 小时）。

### 1.3 更新判断：requiresUpdate()

每次页面请求时调用，决定是否需要真正执行 `update()`，见 [internal/glance/widget.go:173](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget.go#L173-L183)：

```go
func (w *widgetBase) requiresUpdate(now *time.Time) bool {
    if w.cacheType == cacheTypeInfinite {
        return false                          // 无限缓存，永不更新
    }

    if w.nextUpdate.IsZero() {
        return true                           // 首次运行，必须更新
    }

    return now.After(w.nextUpdate)            // 当前时间超过 nextUpdate 才更新
}
```

### 1.4 更新触发入口：Page 级调度

页面内容请求时调用 `page.updateOutdatedWidgets()`，见 [internal/glance/glance.go:233](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/glance.go#L233-L270) 和 [internal/glance/glance.go:334](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/glance.go#L334-L367)：

```go
func (p *page) updateOutdatedWidgets() {
    now := time.Now()
    var wg sync.WaitGroup

    // 遍历所有 Widget，过期的并发启动 update()
    for w := range p.HeadWidgets {
        widget := p.HeadWidgets[w]
        if !widget.requiresUpdate(&now) {
            continue
        }
        wg.Add(1)
        go func() {
            defer wg.Done()
            widget.update(context.Background())
        }()
    }
    // Column 中的 Widget 同理
    wg.Wait()
}
```

每次 `GET /api/pages/{page}/content/` 请求都会先获取 `page.mu` 锁，再调用 `updateOutdatedWidgets()`，确保同一页不会并发更新。

### 1.5 成功/失败后的调度

RSS Widget 的 `update()` 最终会调用 `canContinueUpdateAfterHandlingErr()`，该方法决定下一次更新的时间，见 [internal/glance/widget.go:293](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget.go#L293-L325)：

```
无错误 → scheduleNextUpdate()       → nextUpdate = now + cacheDuration（2 小时），重试计数清零
部分错误 → scheduleEarlyUpdate()  → 指数退避，设置为 Notice（仍显示已有内容）
全部错误 → scheduleEarlyUpdate()  → 设置为 Error，不显示内容
```

---

## 二、部分源失败后的重试机制（指数退避）

### 2.1 错误分级与调度策略

在 `fetchItemsFromFeeds()` 中对失败源计数，见 [internal/glance/widget-rss.go:161](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L161-L189)：

```go
failed := 0
for i := range feeds {
    if errs[i] != nil {
        failed++
        slog.Error("Failed to get RSS feed", "url", requests[i].URL, "error", errs[i])
        continue
    }
    // ... 合并成功的条目
}

if failed == len(requests) {
    return nil, errNoContent          // 全部失败
}
if failed > 0 {
    return entries, fmt.Errorf("%w: missing %d RSS feeds", errPartialContent, failed)  // 部分失败
}
return entries, nil                   // 全部成功
```

随后在 `canContinueUpdateAfterHandlingErr()` 中根据错误类型分支，见 [internal/glance/widget.go:293](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget.go#L293-L325)：

```go
func (w *widgetBase) canContinueUpdateAfterHandlingErr(err error) bool {
    if err != nil {
        w.scheduleEarlyUpdate()        // 有错误，安排提前重试

        if !errors.Is(err, errPartialContent) {
            w.withError(err)           // 全部失败：标记为 Error，不渲染内容
            w.withNotice(nil)
            return false
        }

        w.withError(nil)               // 部分失败：标记为 Notice，仍渲染已获取内容
        w.withNotice(err)
        return true
    }

    w.withNotice(nil)
    w.withError(nil)
    w.scheduleNextUpdate()             // 全部成功：按正常 TTL 调度
    return true
}
```

### 2.2 指数退避算法

`scheduleEarlyUpdate()` 实现了基于重试次数的平方退避，见 [internal/glance/widget.go:350](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget.go#L350-L367)：

```go
func (w *widgetBase) scheduleEarlyUpdate() *widgetBase {
    w.updateRetriedTimes++
    if w.updateRetriedTimes > 5 {
        w.updateRetriedTimes = 5       // 重试次数上限封顶为 5
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

**退避时间表（RSS 默认 2 小时 TTL）：**

| 连续失败次数 | 退避间隔 | 下次更新时间 |
|-------------|---------|-------------|
| 第 1 次失败 | 1² = 1 分钟 | 约 1 分钟后 |
| 第 2 次失败 | 2² = 4 分钟 | 约 4 分钟后 |
| 第 3 次失败 | 3² = 9 分钟 | 约 9 分钟后 |
| 第 4 次失败 | 4² = 16 分钟 | 约 16 分钟后 |
| 第 5 次及以上 | 5² = 25 分钟 | 约 25 分钟后（封顶） |
| 任一时间若退避超过正常 TTL | 以正常 TTL 为准 | 约 2 小时后 |

一旦某次 `update()` 全部成功，`scheduleNextUpdate()` 会将 `updateRetriedTimes` 清零，见 [internal/glance/widget.go:343](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget.go#L343-L348)。

### 2.3 注意事项（来自代码 TODO）

代码中有一段注释说明了重试机制的当前局限，见 [internal/glance/widget.go:294](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget.go#L294-L305)：

- 部分内容 + 提前重试时，后续重试可能返回更少内容
- 未区分服务端错误（如 504，应重试）和客户端限流（如 429，不应立即重试）
- 未来可能改为资源级缓存，仅重抓失败的源而非全部重跑

---

## 三、请求流程

### 3.1 请求入口

多源抓取从 `fetchItemsFromFeeds()` 函数启动，见 [internal/glance/widget-rss.go:152](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L152-L190)。

```go
func (widget *rssWidget) fetchItemsFromFeeds() (rssFeedItemList, error) {
    requests := widget.FeedRequests
    job := newJob(widget.fetchItemsFromFeedTask, requests).withWorkers(30)
    feeds, errs, err := workerPoolDo(job)
    // ... 后续处理
}
```

### 3.2 HTTP 请求构建

每个 RSS 源的实际请求发生在 `fetchItemsFromFeedTask()` 函数中，见 [internal/glance/widget-rss.go:192](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L192-L234)。

请求构建步骤：

```go
req, err := http.NewRequest("GET", request.URL, nil)
req.Header.Add("User-Agent", glanceUserAgentString)

// 自定义 Header (用户可配置)
for key, value := range request.Headers {
    req.Header.Set(key, value)
}

resp, err := defaultHTTPClient.Do(req)
```

**关键点：**
- User-Agent 使用 `Glance/{version} +https://github.com/glanceapp/glance`，定义在 [internal/glance/widget-utils.go:46](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L46-L46)
- 用户可在配置中为每个源自定义 HTTP Header（用于鉴权等场景）
- HTTP 客户端使用 `defaultHTTPClient`，定义在 [internal/glance/widget-utils.go:26](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L26-L32)，超时 5 秒，每 Host 最多 10 个空闲连接

---

## 四、缓存协作机制（HTTP 级 ETag/Last-Modified）

这是 RSS Widget 内部针对单个 RSS 源的 HTTP 协商缓存，与组件级 TTL 缓存是两层独立的缓存机制。

### 4.1 缓存数据结构

每个 Widget 实例维护一个内存缓存 `cachedFeeds`，见 [internal/glance/widget-rss.go:45](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L45-L47) 和 [internal/glance/widget-rss.go:114](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L114-L118)：

```go
type rssWidget struct {
    cachedFeedsMutex sync.Mutex
    cachedFeeds      map[string]*cachedRSSFeed
}

type cachedRSSFeed struct {
    etag         string           // HTTP ETag
    lastModified string           // HTTP Last-Modified
    items        []rssFeedItem    // 已解析的条目列表
}
```

缓存以 `URL → cachedRSSFeed` 为键值对存储，通过 `cachedFeedsMutex` 互斥锁保护并发读写。

### 4.2 请求阶段：发送缓存协商 Header

请求前先查缓存并附带条件请求 Header，见 [internal/glance/widget-rss.go:200](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L200-L210)：

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
- `If-None-Match` + ETag：服务器如果内容未变则返回 304
- `If-Modified-Since` + Last-Modified：基于时间戳的缓存验证

### 4.3 响应阶段：命中 304 Not Modified

收到响应后检查状态码判断，见 [internal/glance/widget-rss.go:222](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L222-L224)：

```go
if resp.StatusCode == http.StatusNotModified && isCached {
    return cache.items, nil
}
```

如果服务器返回 304，直接使用本地缓存条目，无需重新下载和解析。

### 4.4 写入缓存

成功获取新内容后，将 ETag/Last-Modified 回写缓存，见 [internal/glance/widget-rss.go:333](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L333-L341)：

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

### 4.5 两层缓存的协作关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    第 1 层：组件级 TTL 缓存                      │
│  widgetBase.nextUpdate / cacheDuration（默认 2h）                │
│  控制 Widget.update() 是否执行（页面请求时判断 requiresUpdate）│
└──────────────────────────────────────────────┬──────────────────┘
                                               │
                                     TTL 到期，执行 update()
                                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                    第 2 层：HTTP 协商缓存                         │
│  cachedFeeds[url] → ETag / Last-Modified / items                 │
│  控制单个 RSS 源是否真正下载（If-None-Match / 304 命中）         │
└─────────────────────────────────────────────────────────────────┘
```

**协作效果：**
- 2 小时内页面刷新，第 1 层拦截，直接复用 Widget 内存中的 `Items`
- 超过 2 小时触发 `update()`，第 2 层对每个源发协商请求，支持 304 命中省流量
- 即使第 1 层 TTL 到期，若所有源都返回 304，整体抓取几乎零成本

---

## 五、字符集处理

### 5.1 读取原始字节

见 [internal/glance/widget-rss.go:230](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L230-L238)：

```go
body, err := io.ReadAll(resp.Body)
feed, err := feedParser.ParseString(string(body))
```

### 5.2 gofeed 自动字符集检测

项目使用 `github.com/mmcdole/gofeed` 库，Parser 在包级别初始化一次，见 [internal/glance/widget-rss.go:29](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L29-L29)：

```go
var feedParser = gofeed.NewParser()
```

`gofeed.Parser.ParseString()` 内部会：
1. 检测 XML 声明中的 `encoding` 属性（如 `<?xml version="1.0" encoding="GB2312"?>`）
2. 若 XML 声明缺失，使用 `golang.org/x/net/html/charset` 进行自动字符集转换
3. 支持 GBK、GB2312、ISO-8859-1、UTF-8 等多种编码

### 5.3 HTML 实体转义与标签清理

在解析后还对标题和描述做二次处理，见 [internal/glance/widget-rss.go:277](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L276-L280) 和 [internal/glance/widget-rss.go:378](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L378-L402)：

```go
rssItem.Title = html.UnescapeString(item.Title)
```

描述清理流程 `sanitizeFeedDescription()`：
1. 换行符替换为空格
2. 正则 `<\/?[a-zA-Z0-9-]+ ... ?>` 剥离所有 HTML 标签及属性
3. 合并连续空白为单空格
4. `html.UnescapeString()` 反转义 HTML 实体

---

## 六、去重机制

### 6.1 基于 Link 的去重

见 [internal/glance/widget-rss.go:163](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L163-L179)：

```go
seen := make(map[string]struct{})

for i := range feeds {
    for _, item := range feeds[i] {
        if _, exists := seen[item.Link]; exists {
            continue
        }
        entries = append(entries, item)
        seen[item.Link] = struct{}{}
    }
}
```

**设计要点：**
- 使用 `map[string]struct{}` 作为集合，空结构体零值不占内存
- 以条目 URL (Link) 作为唯一标识
- 不同 RSS 源若出现相同 URL，仅保留首次出现的条目
- 去重发生在所有源抓取完成之后、合并结果阶段

### 6.2 去重与顺序

- 去重保留首次出现：遍历 feeds 数组顺序即源配置顺序
- 之后可选择按发布时间重新排序（`PreserveOrder: false` 时），见 [internal/glance/widget-rss.go:87](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L87-L89)

---

## 七、并发限制的四个层级

整个系统存在四层并发/连接限制，层层嵌套：

```
Level 1: Page 级 Widget 并发  ──── 无上限（sync.WaitGroup 等待所有 Widget 完成）
           │
           ▼
Level 2: RSS Widget Worker Pool ──── 最多 30 个 goroutine 并发抓取
           │
           ▼
Level 3: HTTP Transport 连接池  ──── 每 Host 最多 10 个空闲连接
           │
           ▼
Level 4: HTTP 请求超时         ──── 单次请求 5 秒超时
```

### 7.1 Level 1：Page 级 Widget 并发更新

见 [internal/glance/glance.go:233](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/glance.go#L233-L270)：

```go
func (p *page) updateOutdatedWidgets() {
    var wg sync.WaitGroup
    // ...
    for w := range p.HeadWidgets {
        // 每个过期 Widget 起一个 goroutine
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
- 一个页面有多少个过期 Widget，就启动多少个 goroutine（无主动上限）
- 受 `page.mu` 互斥锁保护（见 [internal/glance/config.go:77](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/config.go#L77-L92)），同一 Page 同一时间只有一个 `updateOutdatedWidgets()` 在跑
- 容器类 Widget（group/split-column）内部也有类似的并发逻辑，见 [internal/glance/widget-container.go:23](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-container.go#L23-L42)

### 7.2 Level 2：RSS Widget 内部 Worker Pool

定义在 [internal/glance/widget-utils.go:141](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L141-L242)。RSS Widget 固定配置 **30 个 Worker**，见 [internal/glance/widget-rss.go:155](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L155-L156)：

```go
job := newJob(widget.fetchItemsFromFeedTask, requests).withWorkers(30)
```

Worker Pool 核心参数 `withWorkers()` 会做 `min(workers, len(data))` 裁剪，见 [internal/glance/widget-utils.go:157](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L157-L165)：

```go
func (job *workerPoolJob[I, O]) withWorkers(workers int) *workerPoolJob[I, O] {
    if workers == 0 {
        job.workers = defaultNumWorkers  // 默认 10
    } else {
        job.workers = min(workers, len(job.data))
    }
    return job
}
```

**Worker Pool 工作流程：**

```
                      ┌─────────────────────────────────────┐
                      │         主 goroutine                  │
                      │  tasksQueue / resultsQueue           │
                      └──────────────┬───────────────────────┘
                                     │
              ┌──────────────────────┼─────┬──────────────────┐
              ↓                      ↓     ↓                  ↓
         Worker 1              Worker 2  Worker 3   ...    Worker 30
              │                      │     │                  │
              └──────────────────────┴─────┴──────────────────┘
                                     │
                                     ↓
                           结果按 index 回填，保持输入顺序
```

单任务优化：源数 = 1 时直接同步执行，跳过 channel 开销（见 [internal/glance/widget-utils.go:192](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L192-L195)）。

### 7.3 Level 3：HTTP Transport 连接池

`defaultHTTPClient` 的配置，见 [internal/glance/widget-utils.go:26](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L26-L32)：

```go
const defaultClientTimeout = 5 * time.Second

var defaultHTTPClient = &http.Client{
    Transport: &http.Transport{
        MaxIdleConnsPerHost: 10,              // 每 Host 最多 10 个空闲连接
        Proxy:               http.ProxyFromEnvironment,
    },
    Timeout: defaultClientTimeout,
}
```

**参数说明：**
- `MaxIdleConnsPerHost: 10`：单个目标主机最多保留 10 个空闲持久连接（HTTP Keep-Alive）
- 未设置 `MaxConnsPerHost`：对每个 Host 的并发活跃连接数无硬限制（Go 默认值）
- 未设置 `MaxIdleConns`：全局空闲连接无上限（Go 默认值）
- `Proxy: http.ProxyFromEnvironment`：支持 HTTP_PROXY / HTTPS_PROXY 环境变量

**与 Level 2 的交互：**
- RSS Worker Pool 最多 30 个并发请求
- 若多个源指向同一 Host（如多个 reddit.com RSS），该 Host 最多可同时创建远超 10 个活跃连接（10 只是空闲保留数）
- 真正的瓶颈更多来自 Level 2 的 30 Worker 上限

### 7.4 Level 4：HTTP 请求超时

- 单次 HTTP 请求超时 5 秒（`defaultClientTimeout`），包含连接、重定向、读响应体全流程
- 超时后请求返回错误，该源计入 `failed`，触发 Level 1 的指数退避重试

### 7.5 并发层级汇总表

| 层级 | 位置 | 限制值 | 控制对象 |
|-----|------|-------|---------|
| L1 Page 级 | [internal/glance/glance.go:233](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/glance.go#L233-L270) | 无主动上限（但被 page.mu 串行化） | 同一 Page 内多个 Widget 的并发更新 |
| L2 Worker Pool | [internal/glance/widget-rss.go:155](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L155-L156) | 30 个 goroutine | 单个 RSS Widget 内多个源的并发抓取 |
| L3 连接池 | [internal/glance/widget-utils.go:26](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L26-L32) | 每 Host 10 个空闲连接 | TCP 连接复用（Keep-Alive） |
| L4 超时 | [internal/glance/widget-utils.go:24](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L24-L24) | 5 秒 | 单次 HTTP 请求的最长耗时 |

---

## 八、完整数据流时序

```
Browser → GET /api/pages/{page}/content/
    │
    ├─ page.mu.Lock()                       （同一 Page 串行）
    ├─ page.updateOutdatedWidgets()
    │      │
    │      ├─ 遍历所有 Widget
    │      │    ├─ widget.requiresUpdate(now)
    │      │    │    └─ 首次 / now > nextUpdate ? → 需要更新
    │      │    └─ 需要更新的每个 Widget 起 goroutine
    │      │         └─ [Goroutine] rssWidget.update(ctx)
    │      │                │
    │      │                ├─ fetchItemsFromFeeds()
    │      │                │      │
    │      │                │      ├─ newJob(...).withWorkers(30)
    │      │                │      └─ workerPoolDo(job)
    │      │                │             │
    │      │                │             ├─ [Worker 1] fetchItemsFromFeedTask(url1)
    │      │                │             │      ├─ cachedFeedsMutex.Lock()
    │      │                │             │      ├─ 查 cachedFeeds[url1]
    │      │                │             │      ├─ 附加 If-None-Match / If-Modified-Since
    │      │                │             │      ├─ HTTP GET (defaultHTTPClient, 5s 超时)
    │      │                │             │      ├─ 304 Not Modified?
    │      │                │             │      │    └─ 是 → 返回 cache.items
    │      │                │             │      └─ 200 OK?
    │      │                │             │           ├─ io.ReadAll → []byte
    │      │                │             │           ├─ feedParser.ParseString (自动字符集)
    │      │                │             │           ├─ 字段映射 / 链接补全 / 缩略图查找
    │      │                │             │           ├─ 写入 cachedFeeds[url1] (ETag+LastMod+items)
    │      │                │             │           └─ 返回 items
    │      │                │             │
    │      │                │             ├─ [Worker 2] fetchItemsFromFeedTask(url2)
    │      │                │             ├─ ...
    │      │                │             │
    │      │                │             ├─ 合并所有 feeds[i]
    │      │                │             ├─ seen map 去重 (按 item.Link)
    │      │                │             └─ 返回 entries + failed count
    │      │                │
    │      │                ├─ 可选：sortByNewest()
    │      │                ├─ 裁剪 Limit 条
    │      │                ├─ widget.Items = items
    │      │                │
    │      │                └─ canContinueUpdateAfterHandlingErr(err)
    │      │                     ├─ 全部成功 → scheduleNextUpdate()
    │      │                     │       └─ nextUpdate = now + 2h, retriedTimes = 0
    │      │                     ├─ 部分失败 → scheduleEarlyUpdate()
    │      │                     │       ├─ retriedTimes++ (封顶 5)
    │      │                     │       ├─ 退避 = retriedTimes² 分钟
    │      │                     │       ├─ nextUpdate = min(退避时间, 正常 TTL)
    │      │                     │       └─ 标记 Notice，仍渲染内容
    │      │                     └─ 全部失败 → scheduleEarlyUpdate()
    │      │                             └─ 标记 Error，不渲染内容
    │      │
    │      └─ wg.Wait()                    （等待所有 Widget goroutine 完成）
    │
    ├─ page.mu.Unlock()
    └─ 返回页面渲染结果
```

---

## 九、关键文件索引

| 功能模块 | 文件与位置 |
|----------|-----------|
| RSS Widget 主体 | [internal/glance/widget-rss.go](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go) |
| Widget 基类（TTL 缓存/重试） | [internal/glance/widget.go](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget.go) |
| Worker Pool / HTTP Client | [internal/glance/widget-utils.go](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go) |
| Page 级更新调度 | [internal/glance/glance.go:233-270](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/glance.go#L233-L270) |
| Page 结构体与互斥锁 | [internal/glance/config.go:77-92](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/config.go#L77-L92) |
| 容器 Widget 并发更新 | [internal/glance/widget-container.go](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-container.go) |
| 通用 Singleflight 工具 | [internal/glance/singleflight.go](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/singleflight.go) |
| 组件级缓存调度逻辑 | [internal/glance/widget.go:173-367](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget.go#L173-L367) |
| RSS HTTP 协商缓存 | [internal/glance/widget-rss.go:114-118, 200-210, 222-224, 333-341](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L114-L118) |
| RSS 去重逻辑 | [internal/glance/widget-rss.go:163-179](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L163-L179) |
| 字符集/HTML 清理 | [internal/glance/widget-rss.go:230-238, 378-402](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L230-L238) |
| RSS 30 Worker 配置 | [internal/glance/widget-rss.go:155-156](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L155-L156) |
| HTTP 连接池与超时 | [internal/glance/widget-utils.go:24-32](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L24-L32) |
