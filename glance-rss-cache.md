# RSS 多源抓取与缓存协作详解

## 整体架构概览

RSS Widget 的核心数据流如下：

```
多 RSS 源配置
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
                     按时间排序
                          ↓
                     裁剪数量
```

---

## 一、请求流程

### 1.1 请求入口

多源抓取从 `fetchItemsFromFeeds()` 函数启动，见 [widget-rss.go:152](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L152-L190)。

```go
func (widget *rssWidget) fetchItemsFromFeeds() (rssFeedItemList, error) {
    requests := widget.FeedRequests
    job := newJob(widget.fetchItemsFromFeedTask, requests).withWorkers(30)
    feeds, errs, err := workerPoolDo(job)
    // ... 后续处理
}
```

### 1.2 HTTP 请求构建

每个 RSS 源的实际请求发生在 `fetchItemsFromFeedTask()` 函数中，见 [widget-rss.go:192](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L192-L234)。

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
- User-Agent 使用 `Glance/{version} +https://github.com/glanceapp/glance`，定义在 [widget-utils.go:46](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L46-L46)
- 用户可在配置中为每个源自定义 HTTP Header（用于鉴权等场景）
- HTTP 客户端使用 `defaultHTTPClient`，定义在 [widget-utils.go:26](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L26-L32)，超时 5 秒，每 Host 最多 10 个空闲连接

---

## 二、缓存协作机制

### 2.1 缓存数据结构

每个 Widget 实例维护一个内存缓存 `cachedFeeds`，见 [widget-rss.go:45](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L45-L47)：

```go
type rssWidget struct {
    cachedFeedsMutex sync.Mutex
    cachedFeeds      map[string]*cachedRSSFeed
}

type cachedRSSFeed struct {
    etag         string      // HTTP ETag
    lastModified string      // HTTP Last-Modified
    items        []rssFeedItem // 解析后的条目
}
```

缓存以 `URL → cachedRSSFeed` 为键值对存储。

### 2.2 请求阶段：发送缓存协商 Header

请求前先查缓存并附带条件请求 Header，见 [widget-rss.go:200](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L200-L210)：

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

**工作原理：
- `If-None-Match` + ETag：服务器如果内容未变则返回 304
- `If-Modified-Since` + Last-Modified：基于时间戳的缓存验证

### 2.3 响应阶段：命中 304 Not Modified

收到响应后检查状态码判断，见 [widget-rss.go:222](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L222-L224)：

```go
if resp.StatusCode == http.StatusNotModified && isCached {
    return cache.items, nil
}
```

如果服务器返回 304，直接使用本地缓存条目，无需重新解析。

### 2.4 写入缓存

成功获取新内容后，将 ETag/Last-Modified 回写缓存，见 [widget-rss.go:333](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L333-L341)：

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

**缓存协作完整流程：

```
第一次请求：
  无缓存 → 直接 GET → 200 OK → 解析 → 存储 ETag/Last-Modified + items → 返回

第二次及以后请求：
  有缓存 → 附带 If-None-Match / If-Modified-Since
        ├─ 304 Not Modified → 直接返回缓存 items
        └─ 200 OK → 重新解析 → 更新缓存 → 返回新 items
```

---

## 三、字符集处理

### 3.1 读取原始字节

见 [widget-rss.go:230](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L230-L238)：

```go
body, err := io.ReadAll(resp.Body)
feed, err := feedParser.ParseString(string(body))
```

### 3.2 gofeed 自动字符集检测

项目使用 `github.com/mmcdole/gofeed` 库，Parser 在包级别初始化一次，见 [widget-rss.go:29](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L29-L29)：

```go
var feedParser = gofeed.NewParser()
```

`gofeed.Parser.ParseString()` 内部会：
1. 检测 XML 声明中的 `encoding` 属性（如 `<?xml version="1.0" encoding="GB2312"?>`）
2. 若 XML 声明缺失时，使用 `golang.org/x/net/html/charset` 进行自动字符集转换
3. 支持 GBK、GB2312、ISO-8859-1、UTF-8 等多种编码

### 3.3 HTML 实体转义

在解析后还对标题和描述做二次 HTML Unescape，见 [widget-rss.go:277](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L276-L280) 和 [widget-rss.go:378](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L378-L390)：

```go
rssItem.Title = html.UnescapeString(item.Title)
description = html.UnescapeString(description)
```

描述清理流程 `sanitizeFeedDescription()`：
1. 换行 → 空格
2. 正则剥离所有 HTML 标签
3. 合并连续空白
4. HTML 实体反转义

---

## 四、去重机制

### 4.1 基于 Link 的去重

见 [widget-rss.go:163](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L163-L179)：

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

**设计要点：
- 使用 `map[string]struct{}` 作为集合，零值空结构体不占内存
- 以条目 URL (Link) 作为唯一标识
- 不同 RSS 源若出现相同 URL，仅保留首次出现的条目
- 去重发生在所有源抓取完成之后、合并结果阶段

### 4.2 去重与顺序

- 去重保留首次出现：在遍历 feeds 数组顺序即源配置顺序
- 之后可选择按时间重新排序（`PreserveOrder: false` 时），见 [widget-rss.go:87](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L87-L89)

---

## 五、并发限制

### 5.1 Worker Pool 实现

Worker Pool 定义在 [widget-utils.go:141](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L141-L242)。

RSS Widget 使用 30 个 Worker，见 [widget-rss.go:155](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L155-L156)：

```go
job := newJob(widget.fetchItemsFromFeedTask, requests).withWorkers(30)
```

### 5.2 Worker Pool 数据结构

```go
type workerPoolJob[I any, O any] struct {
    data    []I           // 输入任务列表
    workers int            // Worker 数量
    task    func(I) (O, error) // 任务函数
    ctx     context.Context
}
```

### 5.3 Worker Pool 工作流程

```
                      ┌─────────────────────────────────────────────────┐
                      │            主 goroutine              │
                      │  tasksQueue (chan *Task)          │
                      │  resultsQueue (chan *Task)        │
                      └──────────────┬─────────────────────┘
                                     │
              ┌──────────────────────┬────┴─────┬──────────────────┐
              ↓                  ↓          ↓                  ↓
         Worker 1           Worker 2    Worker 3   ...    Worker N
              │                  │          │                  │
              └──────────────┴──────────┴──────────────────┘
                                     │
                                     ↓
                           结果收集回主 goroutine
```

核心实现见 [widget-utils.go:184](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L184-L242)：

```go
func workerPoolDo[I any, O any](job *workerPoolJob[I, O]) ([]O, []error, error) {
    // 1. 初始化结果和错误切片
    results := make([]O, len(job.data))
    errs := make([]error, len(job.data))

    // 2. 单任务优化：直接执行，避免 channel 开销
    if len(job.data) == 1 {
        results[0], errs[0] = job.task(job.data[0])
        return results, errs, nil
    }

    // 3. 创建任务和结果通道
    tasksQueue := make(chan *workerPoolTask[I, O])
    resultsQueue := make(chan *workerPoolTask[I, O])

    var wg sync.WaitGroup

    // 4. 启动 N 个 Worker goroutine
    for range job.workers {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for t := range tasksQueue {
                t.output, t.err = job.task(t.input)
                resultsQueue <- t
            }
        }()
    }

    // 5. 发送任务到 tasksQueue（独立 goroutine）
    go func() {
        for i := range job.data {
            tasksQueue <- &workerPoolTask[I, O]{index: i, input: job.data[i]}
        }
        close(tasksQueue)
        wg.Wait()
        close(resultsQueue)
    }()

    // 6. 从 resultsQueue 收集所有结果
    for task := range resultsQueue {
        errs[task.index] = task.err
        results[task.index] = task.output
    }

    return results, errs, err
}
```

### 5.4 并发控制要点

| 维度 | 说明 |
|-------|------|
| Worker 数 | RSS 固定 30，实际取 `min(30, len(feeds))`，见 [widget-utils.go:161](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L157-L165) |
| 单任务优化 | 只有 1 个源时不启动 Pool |
| 保持顺序 | 结果按 `index` 回填，与输入顺序一致 |
| 错误隔离 | 单个源失败不影响其他源，见 [widget-rss.go:166](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L166-L170) |
| HTTP 层限流 | `defaultHTTPClient` 每 Host 10 个空闲连接，全局无全局并发 |

### 5.5 错误分级

```go
failed == len(requests) // 全部失败 → errNoContent
failed > 0          // 部分失败 → errPartialContent
failed == 0          // 全部成功 → nil
```

---

## 六、完整数据流时序

```
Widget.update()
    │
    ├─ fetchItemsFromFeeds()
    │      │
    │      ├─ newJob(...).withWorkers(30)
    │      └─ workerPoolDo(job)
    │             │
    │             ├─ [Worker 1] fetchItemsFromFeedTask(url1)
    │             │      ├─ 查 cachedFeeds[url1]
    │             │      ├─ 附加 If-None-Match / If-Modified-Since
    │             │      ├─ HTTP GET
    │             │      ├─ 304? → 返回缓存 items
    │             │      └─ 200? → io.ReadAll → feedParser.ParseString
    │             │           → 字段映射/链接处理/缩略图查找
    │             │           → 写入 cachedFeeds[url1]
    │             │           → 返回 items
    │             │
    │             ├─ [Worker 2] fetchItemsFromFeedTask(url2)
    │             ├─ ...
    │             │
    │             ├─ 合并所有 feeds[i]
    │             ├─ seen map 去重 (Link)
    │             └─ 返回 entries
    │
    ├─ sortByNewest() （可选）
    ├─ 裁剪 Limit 条
    └─ widget.Items = items
```

---

## 七、关键文件索引

| 功能模块 | 文件与位置 |
|----------|-----------|
| RSS Widget 主体 | [widget-rss.go](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go) |
| Worker Pool 实现 | [widget-utils.go:141-242](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L141-L242) |
| HTTP Client 配置 | [widget-utils.go:26-32](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-utils.go#L26-L32) |
| 缓存数据结构 | [widget-rss.go:114-118](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L114-L118) |
| 缓存协商读写 | [widget-rss.go:200-210, 333-341](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L200-L210) |
| 去重逻辑 | [widget-rss.go:163-179](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L163-L179) |
| 字符集/HTML 清理 | [widget-rss.go:230-238, 378-402](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L230-L238) |
| 并发 30 Worker | [widget-rss.go:155-156](file:///d:/fz/0601/solo-dogfeeding/code/136-glance/internal/glance/widget-rss.go#L155-L156) |
