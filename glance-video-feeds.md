# Glance 视频源聚合拉取路径全解析

本文档顺着代码调用链，讲清频道识别、元数据组装、封面缓存、纵向列表的独立渲染分支、抓取失败后的重试机制、发布时间解析失败对排序的影响，以及直播状态接入与刷新频率/排序/展示样式/重试机制的关联。

---

## 一、频道识别：从配置到 Feed URL 的路由

核心代码位于 [widget-videos.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-videos.go)。

### 1.1 配置入口

`videosWidget` 结构体暴露了两个配置字段：

```go
type videosWidget struct {
    Channels  []string `yaml:"channels"`
    Playlists []string `yaml:"playlists"`
    // ...
}
```

用户在 YAML 中可以分开写 `channels` 和 `playlists`，从语义上做区分。

### 1.2 初始化阶段的归一化

[initialize()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-videos.go#L36-L64) 方法会把 `Playlists` 统一合并进 `Channels`，但给每个 playlist ID 加上 `videosWidgetPlaylistPrefix`（值为 `"playlist:"`）前缀：

```go
if len(widget.Playlists) > 0 {
    initialLen := len(widget.Channels)
    widget.Channels = append(widget.Channels, make([]string, len(widget.Playlists))...)

    for i := range widget.Playlists {
        widget.Channels[initialLen+i] = videosWidgetPlaylistPrefix + widget.Playlists[i]
    }
}
```

这样后续所有逻辑只需处理 `Channels` 一个切片，前缀承担了类型标签的作用。

### 1.3 拉取时的三路分支

真正构造 RSS 请求 URL 的逻辑在 [fetchYoutubeChannelUploads()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-videos.go#L141-L216)：

```go
for i := range channelOrPlaylistIDs {
    var feedUrl string
    if strings.HasPrefix(channelOrPlaylistIDs[i], videosWidgetPlaylistPrefix) {
        // 分支 A: 显式 playlist（playlist: 前缀）
        feedUrl = "https://www.youtube.com/feeds/videos.xml?playlist_id=" +
            strings.TrimPrefix(channelOrPlaylistIDs[i], videosWidgetPlaylistPrefix)
    } else if !includeShorts && strings.HasPrefix(channelOrPlaylistIDs[i], "UC") {
        // 分支 B: UC 频道，且用户要求排除 Shorts
        // 将 UC 前缀替换为 UULF，即"该频道上传的长视频"系统播放列表
        playlistId := strings.Replace(channelOrPlaylistIDs[i], "UC", "UULF", 1)
        feedUrl = "https://www.youtube.com/feeds/videos.xml?playlist_id=" + playlistId
    } else {
        // 分支 C: 普通 channel_id，包含 Shorts
        feedUrl = "https://www.youtube.com/feeds/videos.xml?channel_id=" + channelOrPlaylistIDs[i]
    }
}
```

关键点：

| 输入形式 | includeShorts | 最终请求 | 说明 |
|----------|--------------|----------|------|
| `playlist:PLxxx` | 任意 | `playlist_id=PLxxx` | 用户显式指定播放列表 |
| `UCxxx` | false | `playlist_id=UULFxxx` | 替换前缀，拿系统生成的"只含长视频"列表 |
| `UCxxx` | true | `channel_id=UCxxx` | 完整频道 feed，含 Shorts |
| 其他 ID | 任意 | `channel_id=xxx` | 兜底走 channel feed |

并发方面使用 `workerPoolDo(job)`，最多 30 个 worker 同时请求，批量拉取。

---

## 二、元数据组装：从 XML 到 video 结构体

### 2.1 XML 解析结构

[youtubeFeedResponseXml](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-videos.go#L95-L111) 定义了解析模板：

```go
type youtubeFeedResponseXml struct {
    Channel     string `xml:"author>name"`
    ChannelLink string `xml:"author>uri"`
    Videos      []struct {
        Title     string `xml:"title"`
        Published string `xml:"published"`
        Link      struct {
            Href string `xml:"href,attr"`
        } `xml:"link"`
        Group struct {
            Thumbnail struct {
                Url string `xml:"url,attr"`
            } `xml:"http://search.yahoo.com/mrss/ thumbnail"`
        } `xml:"http://search.yahoo.com/mrss/ group"`
    } `xml:"entry"`
}
```

注意缩略图走的是 `media:group/media:thumbnail` 的 mrss 命名空间。

### 2.2 组装循环

在 `fetchYoutubeChannelUploads()` 内部，每条视频做如下处理：

```go
for j := range response.Videos {
    v := &response.Videos[j]
    var videoUrl string

    if videoUrlTemplate == "" {
        // 默认直接用 YouTube 原始链接
        videoUrl = v.Link.Href
    } else {
        // 支持用户自定义 URL 模板，替换 {VIDEO-ID}
        parsedUrl, err := url.Parse(v.Link.Href)
        if err == nil {
            videoUrl = strings.ReplaceAll(videoUrlTemplate, "{VIDEO-ID}", parsedUrl.Query().Get("v"))
        } else {
            videoUrl = "#"
        }
    }

    videos = append(videos, video{
        ThumbnailUrl: v.Group.Thumbnail.Url,
        Title:        v.Title,
        Url:          videoUrl,
        Author:       response.Channel,       // 频道名
        AuthorUrl:    response.ChannelLink + "/videos", // 频道主页
        TimePosted:   parseYoutubeFeedTime(v.Published),
    })
}
```

[video](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-videos.go#L122-L129) 结构体一共 6 个字段，就是前端渲染需要的最小元数据集合。

### 2.3 后处理

所有频道视频汇总后：

1. `videos.sortByNewest()` — 按 `TimePosted` 降序，跨频道混合排序
2. 超过 `Limit`（默认 25）则截断
3. 如果有部分频道失败，返回 `errPartialContent`，不会丢弃已拿到的视频

---

## 三、封面缓存：三层缓存机制

封面（缩略图）的缓存不是单一动作，而是由三层叠加完成的。

### 3.1 应用层：Widget 级定时缓存

在 [initialize()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-videos.go#L37)：

```go
widget.withTitle("Videos").withCacheDuration(time.Hour)
```

Videos widget 默认 1 小时才重新拉一次 feed。也就是说，1 小时内即使 YouTube 有新视频、新缩略图，前端也看不到。这一层由 [widgetBase](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget.go#L259-L269) 的 `cacheTypeDuration` 机制控制，配合 `requiresUpdate()` + `scheduleNextUpdate()` 实现。

> 对比：RSS widget 的缓存更精细，见下一节。

### 3.2 HTTP 层：Videos widget 缺失的条件请求

这里值得注意的是 **Videos widget 并没有做 ETag / Last-Modified 条件请求**。也就是说每次刷新 feed 都是完整下载 XML。

作为参照，[widget-rss.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-rss.go#L114-L118) 的 RSS widget 有完整实现：

```go
type cachedRSSFeed struct {
    etag         string
    lastModified string
    items        []rssFeedItem
}
```

请求时带上 `If-None-Match` / `If-Modified-Since`，服务端返回 `304 Not Modified` 就复用内存中的 `cache.items`。

Videos widget 如果要省带宽，可以借鉴这套机制。

### 3.3 浏览器层：前端渲染的懒加载 + CDN

前端模板 [video-card-contents.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/video-card-contents.html#L1-L12)：

```html
<img class="video-thumbnail thumbnail" loading="lazy" src="{{ .ThumbnailUrl }}" alt="">
```

两点：

- `loading="lazy"`：图片进入视口才加载，减少初始请求数
- `src` 直接指向 YouTube CDN（`i.ytimg.com`），不经过 Glance 后端代理，所以封面本身的 HTTP 缓存完全由 YouTube 的 Cache-Control 头 + 浏览器本地缓存决定

也就是说，Glance 后端并不存储、代理、缓存任何封面图片二进制数据，它只存 URL 字符串，实际取图交给浏览器。

---

## 四、纵向列表的独立渲染分支

Videos widget 一共有三种展示样式，在 [Render()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-videos.go#L80-L93) 里按 `Style` 字段分派到三个模板：

```go
func (widget *videosWidget) Render() template.HTML {
    var template *template.Template
    switch widget.Style {
    case "grid-cards":
        template = videosWidgetGridTemplate
    case "vertical-list":
        template = videosWidgetVerticalListTemplate
    default:
        template = videosWidgetTemplate
    }
    return widget.renderTemplate(widget, template)
}
```

三种样式的差异不止布局，**纵向列表是完全独立的渲染分支**，没有复用卡片子模板。

### 4.1 默认样式 videos.html（横向卡片轮播）

[videos.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/videos.html#L1-L15)：

```html
<div class="carousel-container">
    <div class="cards-horizontal carousel-items-container">
        {{ range .Videos }}
        <div class="card widget-content-frame thumbnail-parent">
            {{ template "video-card-contents" . }}
        </div>
        {{ end }}
    </div>
</div>
```

- 外层是 JS 轮播容器 `carousel-container`，由 [page.js 的 setupCarousels()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/static/js/page.js#L15-L48) 接管滚动条和左右截断遮罩
- 每个视频包一层 `card widget-content-frame`（带背景色和边框的卡片样式）
- 复用 [video-card-contents.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/video-card-contents.html) 子模板：大图（16:8.9 宽高比）+ 标题两行截断 + 时间与频道名

### 4.2 网格样式 videos-grid.html（网格卡片）

[videos-grid.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/videos-grid.html#L1-L13)：

```html
<div class="cards-grid collapsible-container" data-collapse-after-rows="{{ .CollapseAfterRows }}">
    {{ range .Videos }}
    <div class="card widget-content-frame thumbnail-parent">
        {{ template "video-card-contents" . }}
    </div>
    {{ end }}
</div>
```

- 与默认样式**共享** `video-card-contents.html` 子模板，只是外层容器换成 CSS Grid
- `data-collapse-after-rows` 配合 [setupCollapsibleGrids()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/static/js/page.js#L420-L493) 根据每行卡片数动态折叠，响应式

### 4.3 纵向列表 videos-vertical-list.html（完全独立分支）

[videos-vertical-list.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/videos-vertical-list.html#L1-L20) 不引用 `video-card-contents.html`，从头写了一套结构：

```html
<ul class="list list-gap-14 collapsible-container" data-collapse-after="{{ .CollapseAfter }}">
    {{- range .Videos }}
    <li class="flex thumbnail-parent gap-10 items-center">
        <img class="video-horizontal-list-thumbnail thumbnail" loading="lazy" src="{{ .ThumbnailUrl }}" alt="">
        <div class="min-width-0">
            <a class="block text-truncate color-primary-if-not-visited" href="{{ .Url | safeURL }}" ...>{{ .Title }}</a>
            <ul class="list-horizontal-text flex-nowrap">
                <li class="shrink-0" {{ dynamicRelativeTimeAttrs .TimePosted }}></li>
                <li class="min-width-0">
                    <a class="block text-truncate" href="{{ .AuthorUrl }}" ...>{{ .Author }}</a>
                </li>
            </ul>
        </div>
    </li>
    {{- end }}
</ul>
```

与前两个样式的关键差异：

| 维度 | 默认 / 网格 | 纵向列表 |
|------|------------|---------|
| 子模板 | 共享 `video-card-contents.html` | 完全独立，自己写 HTML |
| 缩略图 CSS 类 | `video-thumbnail`（宽 100%，16:8.9） | `video-horizontal-list-thumbnail`（高 4rem，16:8.9） |
| 标题截断 | `text-truncate-2-lines`（两行） | `text-truncate`（单行） |
| 外层容器 | `div` + `carousel-container` / `cards-grid` | `<ul>` 语义化列表 |
| 折叠机制 | `data-collapse-after-rows`（按行） | `data-collapse-after`（按条数） |
| 折叠 JS | `setupCollapsibleGrids()`（带 ResizeObserver） | `setupCollapsibleLists()`（简单计数） |
| 卡片框架 | `card widget-content-frame`（有背景和边框） | 无卡片背景，纯列表项 |

这意味着如果以后给视频加直播徽章、观看人数等 UI，**纵向列表需要单独改**，改 `video-card-contents.html` 不会自动同步到纵向列表样式。

---

## 五、抓取失败与重试：notice/error 状态机与内容保留

重试机制不是孤立的"等 N 分钟再试"，而是由 `withError` → `withNotice` → `canContinueUpdateAfterHandlingErr` → `scheduleEarlyUpdate` / `scheduleNextUpdate` 四段代码共同组成的状态机，直接决定了 UI 上显示什么、是否保留旧内容、以及下次什么时候刷新。

### 5.1 三个关键标志位

[widgetBase 结构体](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget.go#L149-L167) 中有三个字段决定一切：

```go
type widgetBase struct {
    ContentAvailable    bool   // 是否展示过成功内容
    Error               error  // 严重错误
    Notice              error  // 轻微提示
    nextUpdate          time.Time // 下次允许刷新的时间点
    updateRetriedTimes  int    // 连续失败次数
    // ...
}
```

三者的语义绑定：
- `ContentAvailable == false`：从未成功过，渲染时显示全屏 ERROR 面板
- `ContentAvailable == true && Error != nil`：曾成功过，现在有严重错误，渲染内容+右上角红色 major 图标
- `ContentAvailable == true && Notice != nil`：曾成功过，现在有轻微提示，渲染内容+右上角黄色 minor 图标
- `ContentAvailable == true && Error == nil && Notice == nil`：一切正常，渲染干净的内容

### 5.2 withError 的隐含副作用：首次成功解锁 ContentAvailable

[withError()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget.go#L283-L291) 是整个状态机的关键开关：

```go
func (w *widgetBase) withError(err error) *widgetBase {
    if err == nil && !w.ContentAvailable {
        w.ContentAvailable = true   // 第一次无错误调用 → 永久解锁
    }
    w.Error = err
    return w
}
```

`ContentAvailable` 是**单向锁**：一旦被设为 `true`，代码中没有任何路径能把它重新设回 `false`（除了 `renderTemplate()` 在模板渲染失败时会设为 false，但这跟网络请求无关）。意味着：

- 首次加载失败 → `ContentAvailable` 仍为 false → 显示全屏 ERROR
- 哪怕只有一次成功 → `ContentAvailable = true` → 之后再失败也不会回到全屏 ERROR，只会在内容上方显示红色图标并保留上次的旧数据

这是一个"渐强可信度"设计：widget 只要成功过一次，用户就永远不会再看到空 ERROR 页，最差情况是看到稍旧的数据。

### 5.3 canContinueUpdateAfterHandlingErr 的完整分支

[canContinueUpdateAfterHandlingErr()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget.go#L293-L325) 是 Videos widget 的 `update()` 中唯一的出口判断：

```go
func (w *widgetBase) canContinueUpdateAfterHandlingErr(err error) bool {
    if err != nil {
        w.scheduleEarlyUpdate()           // 任何错误都触发指数退避重试

        if !errors.Is(err, errPartialContent) {
            // 分支 1：严重错误（全部失败）
            w.withError(err)              // Error = err，Notice = nil
            w.withNotice(nil)
            return false                  // ⚠ 返回 false → widget.Videos 不会被赋值
        }

        // 分支 2：部分错误（只挂了部分频道）
        w.withError(nil)                  // Error = nil；若首次成功则解锁 ContentAvailable
        w.withNotice(err)                 // Notice = err
        return true                       // ✓ 返回 true → widget.Videos 会被赋值

    }

    // 分支 3：完全成功
    w.withNotice(nil)
    w.withError(nil)                      // Error = nil；若首次成功则解锁 ContentAvailable
    w.scheduleNextUpdate()                // 重试计数归零，正常排期
    return true
}
```

把三个分支和 `update()` 的后续代码拼起来看：

```go
// videosWidget.update()
videos, err := fetchYoutubeChannelUploads(...)

if !widget.canContinueUpdateAfterHandlingErr(err) {
    return   // 严重错误时直接 return，Videos 不更新
}

// 只有 return true 才会走到下面
if len(videos) > widget.Limit {
    videos = videos[:widget.Limit]
}
widget.Videos = videos   // 用新数据覆盖旧数据
```

因此三种情况下的完整状态变化是：

| 场景 | ContentAvailable 之前 | ContentAvailable 之后 | Error | Notice | 数据是否更新 | UI 表现 |
|------|----------------------|----------------------|-------|--------|------------|---------|
| **首次加载全挂** | false | false | errNoContent | nil | ❌ 不更新（保留空数组） | 全屏红色 ERROR 面板 |
| **后续全挂（曾成功过）** | true | true | errNoContent | nil | ❌ 不更新（保留旧 Videos） | 旧内容 + 右上角红色 major 图标，title 显示错误信息 |
| **部分失败** | false/true | true | nil | errPartialContent | ✅ 用拿到的部分数据覆盖 | 内容 + 右上角黄色 minor 图标 |
| **全部成功** | false/true | true | nil | nil | ✅ 新数据覆盖 | 干净内容，无图标 |

关键点：**严重错误时 `update()` 在赋值 `widget.Videos` 之前就 return 了**，所以旧数据得以保留（如果之前有的话）。这是 Glance 的 "graceful degradation" 策略：宁旧勿空。

### 5.4 更新触发节奏：只有 HTTP 请求才会驱动刷新

整个项目**没有后台定时 goroutine** 主动刷新 widget。更新链路完全由 HTTP 请求驱动：

1. 浏览器加载页面 → `setupPage()` 调用 `fetchPageContent()`（[page.js L746-L784](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/static/js/page.js#L746-L784)）
2. 请求打到 `/api/pages/{page}/content/` → [handlePageContentRequest()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/glance.go#L334-L367)
3. 该 handler 先拿 `page.mu` 互斥锁 → 调用 [updateOutdatedWidgets()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/glance.go#L233-L270)
4. `updateOutdatedWidgets()` 遍历所有 widget，调 [requiresUpdate()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget.go#L173-L183) 判断：

```go
func (w *widgetBase) requiresUpdate(now *time.Time) bool {
    if w.cacheType == cacheTypeInfinite {
        return false
    }
    if w.nextUpdate.IsZero() {
        return true   // 从未刷新过 → 一定刷
    }
    return now.After(w.nextUpdate)   // 到了 nextUpdate 时间点才刷
}
```

5. 需要刷新的 widget 被并发 goroutine 执行 `widget.update(ctx)`，`wg.Wait()` 等全部完成才释放锁、渲染模板、返回响应。

**前端 JS 也不会周期轮询**：`setupDynamicRelativeTime()` 每 60 秒只刷新"5 分钟前"这样的相对时间文本，不重新拉 `/api/pages/.../content/`。也就是说如果用户把页面开着挂 8 小时不手动刷新，widget 数据就是 8 小时前的快照。

### 5.5 scheduleEarlyUpdate 指数退避的细节

[scheduleEarlyUpdate()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget.go#L350-L367)：

```go
func (w *widgetBase) scheduleEarlyUpdate() *widgetBase {
    w.updateRetriedTimes++
    if w.updateRetriedTimes > 5 {
        w.updateRetriedTimes = 5       // 最大 5 次，封顶
    }

    nextEarlyUpdate := time.Now().Add(
        time.Duration(math.Pow(float64(w.updateRetriedTimes), 2)) * time.Minute)
    nextUsualUpdate := w.getNextUpdateTime()

    // 取两者中更早的那个
    if nextEarlyUpdate.After(nextUsualUpdate) {
        w.nextUpdate = nextUsualUpdate
    } else {
        w.nextUpdate = nextEarlyUpdate
    }
    return w
}
```

Videos widget 正常周期 1 小时，所以实际退避时间表是：

| 连续失败次数 | 退避间隔 | 与 1h 比较 | 实际 nextUpdate |
|------------|---------|-----------|----------------|
| 1 | 1 分钟 | < 1h | 1 分钟后 |
| 2 | 4 分钟 | < 1h | 4 分钟后 |
| 3 | 9 分钟 | < 1h | 9 分钟后 |
| 4 | 16 分钟 | < 1h | 16 分钟后 |
| 5 | 25 分钟 | < 1h | 25 分钟后 |
| 6+ | 仍按 25 分钟（封顶） | < 1h | 25 分钟后 |

但这个"25 分钟后"只意味着 `requiresUpdate()` 会返回 true，**真正触发刷新还要等下一次用户访问页面**。如果用户在退避时间窗口内根本没来访问，退避就毫无意义——下一次访问时直接判断 `now.After(nextUpdate)` 为 true，立刻刷新。

一旦某次刷新成功（分支 3），`scheduleNextUpdate()` 会把 `updateRetriedTimes = 0`，退避计数器完全归零。

### 5.6 worker 池层面的容错：单 channel 失败不传染

[fetchYoutubeChannelUploads()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-videos.go#L160-L215) 中：

```go
responses, errs, err := workerPoolDo(job)

for i := range responses {
    if errs[i] != nil {
        failed++
        slog.Error("Failed to fetch youtube feed", "channel", channelOrPlaylistIDs[i], "error", errs[i])
        continue   // 只记日志，跳过这个频道
    }
    // ... 处理成功频道的视频
}
```

单个 HTTP 请求失败（网络超时、YouTube 500、404 等）只会让对应频道的视频丢失，不会让整个批次失败。只有 `workerPoolDo` 本身返回 `err`（context 取消等非常罕见的情况）或者所有 `len(videos) == 0` 才会走到 `errNoContent` 分支。

---

## 六、发布时间解析失败对排序的影响

### 6.1 解析函数的兜底行为

[parseYoutubeFeedTime()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-videos.go#L113-L120)：

```go
func parseYoutubeFeedTime(t string) time.Time {
    parsedTime, err := time.Parse("2006-01-02T15:04:05-07:00", t)
    if err != nil {
        return time.Now()   // 解析失败→返回当前时间
    }
    return parsedTime
}
```

这是一个"宁新勿旧"的策略：时间字符串解析不出来时，**直接回退到 `time.Now()`**，而不是零值或某个很早的时间。

对比 [utils.go 里的 parseRFC3339Time()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/utils.go#L122-L129)，也是同样的处理模式——失败一律 `time.Now()`。这在项目里是统一约定。

### 6.2 对排序的具体影响

排序函数 [videoList.sortByNewest()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-videos.go#L133-L139) 用的是 `After` 比较：

```go
func (v videoList) sortByNewest() videoList {
    sort.Slice(v, func(i, j int) bool {
        return v[i].TimePosted.After(v[j].TimePosted)
    })
    return v
}
```

因此，解析失败的视频会被推到**列表最顶端**，因为 `time.Now()` 永远比任何真实的视频发布时间新。

几种后果：

1. **单条视频时间格式异常**：该视频会出现在顶部，看起来像"刚刚发布"。前端相对时间由 [dynamicRelativeTimeAttrs()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates.go#L89-L91) 写入 `data-dynamic-relative-time` 属性，JS 侧 [timestampToRelativeTime()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/static/js/page.js#L56-L82) 会把 `time.Now().Unix()` 渲染成"1m"或"in 1m"，用户会看到"刚刚发布"的假象。

2. **整个 feed 的时间字段格式变化**（例如 YouTube 改了 RSS 格式）：所有视频的 `TimePosted` 都是 `Now` 附近，排序退化为"按 XML 中出现的顺序"，因为 `sort.Slice` 不是稳定排序，同一毫秒的条目之间顺序可能抖。

3. **截断影响**：因为默认按 `Limit`（25）截断，解析失败的"伪最新"视频会挤占掉真正的新视频名额，导致最新真实内容不显示。

如果要改进，更好的做法是：解析失败时跳过该条视频（记一条 warn 日志），或者给一个比 Unix 0 还早的哨兵值让它沉底，而不是置顶。

---

## 七、直播状态接入：机制限制与改造方向

接入直播状态不是简单加个 `IsLive` 字段就完事。现有 widget 框架在刷新触发、缓存粒度、错误通道等方面都有硬性约束，会直接限制直播功能的设计空间。

### 7.1 Videos widget 现状

Videos widget 的 [video](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-videos.go#L122-L129) 结构体：

```go
type video struct {
    ThumbnailUrl string
    Title        string
    Url          string
    Author       string
    AuthorUrl    string
    TimePosted   time.Time
}
```

**没有任何直播相关字段**（如 `IsLive`、`LiveSince`、`ViewersCount`）。对应的前端模板也没有 LIVE 徽章、观看人数等 UI。

原因：Videos widget 当前只消费 YouTube 的 RSS feed，而 RSS feed 只包含"已发布的上传视频"，不含直播调度信息。要拿到直播状态需要额外走 YouTube Data API / GQL。

### 7.2 框架限制一：单一 cacheDuration，无法双轨刷新

每个 widget 只有一套缓存参数，在 `widgetBase` 里：

```go
type widgetBase struct {
    cacheDuration  time.Duration
    cacheType      cacheType
    nextUpdate     time.Time
    // ...
}
```

`initialize()` 里一次性设定后就不能拆开了：

```go
widget.withTitle("Videos").withCacheDuration(time.Hour)
```

但直播和上传视频对新鲜度的要求完全不同：

| 数据 | 合理刷新周期 | 原因 |
|------|------------|------|
| 已上传视频列表 | 1 小时 | 新视频发布不会很频繁，RSS 本身也有延迟 |
| 直播在播状态 | 5~10 分钟 | 开播/下播随时可能发生，1 小时延迟完全不可用 |
| 实时观众数 | 1 分钟以内 | 观众数是秒级变化的 |

Twitch Channels widget 用的是 10 分钟缓存，那是因为它**只有直播状态**这一种数据，可以做折中。但 Videos widget 要同时承载"历史视频 + 直播状态"两种生命周期差异巨大的数据，单一 `cacheDuration` 就成了瓶颈：

- 设为 1 小时：直播状态完全不可用
- 设为 10 分钟：上传视频的 RSS 被过度请求，浪费 YouTube 带宽和自己的请求配额

**可行的改造方向**：
1. 在 `videosWidget` 内部维护第二套 `liveNextUpdate` 时间戳，`update()` 里分别判断 RSS 和直播 API 是否该刷，但这需要绕过 `widgetBase.requiresUpdate()` 的单一路径
2. 或者把直播状态拆成独立的 `twitch-channels` 式 widget，但用户体验就割裂了——视频和直播状态不在同一个卡片里

### 7.3 框架限制二：刷新完全由页面请求驱动，无后台轮询

如 5.4 节所述，更新只在用户访问 `/api/pages/.../content/` 时触发，前端 JS 也不会周期轮询。这意味着：

- 用户打开页面看了一眼后最小化 3 小时 → 3 小时内直播状态不会变
- 用户一直在看页面但不手动 F5 → `setupDynamicRelativeTime()` 只刷新"X 分钟前"的文字，直播状态不会更新
- 10 分钟的 `cacheDuration` 只保证"用户第 11 分钟访问时会触发刷新"，不保证"第 10 分钟准时刷新"

如果做直播功能，用户合理的预期是"正在直播的频道旁边，观众数和 LIVE 徽章是实时变化的"。但现有架构连 10 分钟级的准实时更新都做不到，只能在每次页面加载时刷新一次。

**可行的改造方向**：
1. 前端加 `setInterval` 周期调用 `/api/pages/.../content/` 或新增 `/api/widgets/{id}` 接口（目前 [handleWidgetRequest()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/glance.go#L404-L425) 还是 501 Not Implemented）
2. 后端加后台 goroutine ticker 定时刷新指定 widget，但这会打破"只在请求时才计算"的简单模型，需要处理并发安全和 goroutine 生命周期

### 7.4 框架限制三：单一 Error / Notice 通道

`widgetBase` 只有一个 `Error` 槽和一个 `Notice` 槽。接入直播 API 后，一个 Videos widget 可能同时遇到多种问题：

- RSS 拉 10 个频道，2 个失败 → 想显示"部分视频可能缺失"
- 直播 API 单独超时 → 想显示"直播状态暂不可用"
- 直播 API 遇到 rate limit → 想显示"直播数据已延迟"

但现在 Notice 只能存一个 error，后面的会覆盖前面的。直播 API 和 RSS 其中一个挂了，到底算严重错误（显示红色图标、保留旧数据）还是轻微提示（黄色图标），也没有明确优先级。

**可行的改造方向**：
- 自定义 `multiError` 类型把多个错误拼起来放进 Notice
- 或者在 `videosWidget` 里新增 `LiveError`、`FeedError` 两个独立字段，渲染时分别显示

### 7.5 框架限制四：nextUpdate 被严重错误和部分错误共享

`scheduleEarlyUpdate()` 是"任何错误都触发"，不区分是 RSS 挂了还是直播 API 挂了：

- 如果直播 API 超时（5 秒内的临时故障）→ 触发退避，1 分钟后重试，这合理
- 但如果直播 API 返回 403 Forbidden（API Key 配置错误，永久故障）→ 仍然按 1/4/9/16/25 分钟退避重试，白白消耗资源

代码作者在 TODO 注释里已经意识到这个问题：

```go
// TODO: needs covering more edge cases.
// ... need some kind of mechanism that tells us whether we should update early
// or not depending on the number of things that failed during the initial
// and subsequent update and how they failed - ie whether it was server
// error (like gateway timeout, do retry early) or client error (like
// hitting a rate limit, don't retry early).
```

接入直播后这个问题会被放大：直播 API 和 RSS 有不同的失败模式（403/429/5xx/超时），应该有不同的退避策略，但现在是一锅端。

### 7.6 参照：Twitch Channels widget 的实现

项目中 Twitch 频道 widget（[widget-twitch-channels.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-twitch-channels.go)）是一份完整的参考实现，也展示了在当前框架内能做到的上限。

#### 数据结构

[twitchChannel](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-twitch-channels.go#L62-L73)：

```go
type twitchChannel struct {
    Login        string
    Exists       bool
    Name         string
    StreamTitle  string
    AvatarUrl    string
    IsLive       bool          // 是否在播
    LiveSince    time.Time     // 开播时间
    Category     string        // 分类（游戏名）
    CategorySlug string
    ViewersCount int           // 观众数
}
```

#### 排序：两种模式的差异

Twitch 提供了两种排序方式，由 `SortBy` 配置项控制：

[sortByViewers() / sortByLive()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-twitch-channels.go#L77-L87)：

```go
func (channels twitchChannelList) sortByViewers() {
    sort.Slice(channels, func(i, j int) bool {
        return channels[i].ViewersCount > channels[j].ViewersCount
    })
}

func (channels twitchChannelList) sortByLive() {
    sort.SliceStable(channels, func(i, j int) bool {
        return channels[i].IsLive && !channels[j].IsLive
    })
}
```

注意 `sortByLive()` 用的是 **`sort.SliceStable` 而不是 `sort.Slice`**——这样可以保证"同样在播"或"同样离线"的频道之间相对顺序不抖，避免每次刷新 UI 跳动。

另外 Twitch 对离线频道做了一个细节：[fetchChannelFromTwitchTask()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-twitch-channels.go#L199-L203) 中离线频道 `ViewersCount = -1`，这样即便用观众数排序，离线频道也不会排在 0 观众的真直播频道前面。

如果 Videos widget 要接入直播，排序至少需要两种模式：

- 默认：`sortByNewest()` 仍按时间排序，但直播视频优先（类似 Twitch 的 `sortByLive`）
- 可选：直播视频按观众数降序

同时必须用 `sort.SliceStable`，否则每 10 分钟刷新一次排序都会让同样状态的视频相互换位。

#### 展示样式：三种 Videos 样式需要分别改造

当前三种 Videos 样式中，默认样式和网格样式复用了 `video-card-contents.html`，纵向列表是独立分支。如果接入直播状态，三种样式需要不同的 UI 元素：

**默认横向卡片 + 网格卡片**（共用 `video-card-contents.html`）：

参考 Twitch 模板 [twitch-channels.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/twitch-channels.html#L7-L24) 的做法，卡片需要：
- 缩略图角上加 `.twitch-channel-live` 类似的 LIVE 红点徽章
- 标题下方加直播分类（如果有）和观众数（`formatApproxNumber` 格式化，如 "12.3k viewers"）
- 直播时标题换成 `StreamTitle`，而不是视频标题
- 鼠标悬停弹出直播预览图（Twitch 用了 popover 机制）

**纵向列表**（独立分支 `videos-vertical-list.html`）：

纵向列表空间更紧凑，需要在已有结构上最小化加东西：
- 主播名旁边加 LIVE 文字标签（参考 Twitch 模板的 `{{ if .IsLive }} color-highlight{{ end }}` 高亮）
- 时间位换成"已直播 X 时间"（用 `LiveSince` 算相对时间，而不是视频发布时间）
- 可选：右侧加 `ViewersCount`

两种渲染分支意味着**所有直播 UI 改动都要做两遍**，除非先把纵向列表改造成也复用 `video-card-contents.html` 的子模板。

### 7.7 接入影响总览

| 维度 | 当前 Videos | 接入直播后需要变更 | 限制级别 |
|------|------------|------------------|---------|
| 刷新频率 | 1 小时，单一值 | 需要双轨：RSS 1h + 直播 API 5~10min | 🔴 框架限制，需改 widgetBase 或绕过 |
| 刷新触发 | 仅页面请求时 | 前端加周期轮询或后台 ticker | 🔴 架构限制 |
| 轮询并发性 | N/A（无轮询） | page.mu 全局互斥锁阻止高频并发刷新 | 🔴 架构限制（见 8.2 节） |
| 接口可靠性 | N/A（很少调用） | fetch 无超时、无状态码检查、无重试 | 🔴 前端完全缺失（见 8.3 节） |
| video 结构体 | 6 字段 | 加 `IsLive`、`LiveSince`、`LiveViewers`、`StreamTitle`、`Category` | 🟢 简单改动 |
| 排序 | `sort.Slice` 按时间 | 加 `sort.SliceStable` 直播置顶模式，离线用哨兵值 | 🟡 需改动排序函数 |
| 错误通道 | 单一 Error/Notice | 需要区分 RSS 错误和直播错误 | 🟡 中等改动 |
| 退避策略 | 不区分错误类型 | 区分超时/限流/权限错误的退避 | 🟡 作者已标 TODO |
| 默认/网格样式 | `video-card-contents.html` | 加 LIVE 徽章、观众数、预览 popover | 🟢 模板改动 |
| 纵向列表样式 | 独立分支 | 单独加 LIVE 标签、直播时长，或先重构复用子模板 | 🟡 中等改动 |
| ContentAvailable | 单向锁 true | 保留现状即可（旧数据+红色图标的 graceful degradation 对直播是合理的） | ✅ 无需改动 |

---

## 八、整页刷新路径与可靠性分析

直播状态要做到"准实时刷新"，不是简单在前端加个 `setInterval` 就能完事。整条刷新链路在并发控制、超时处理、错误响应等环节都有缺口，会直接影响直播刷新的稳定性和用户体验。

### 8.1 整页刷新的完整路径

刷新不是"某个 widget 自己刷新"，而是**整页所有 widget 一起判断过期、一起更新、一起渲染、一起返回**。完整前后端链路如下：

**前端触发（page.js）**：

[setupPage()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/static/js/page.js#L746-L786) 在页面加载时执行一次：

```js
async function setupPage() {
    initThemePicker();
    const pageElement = document.getElementById("page");
    const pageContentElement = document.getElementById("page-content");
    const pageContent = await fetchPageContent(pageData);   // ← 仅此一次

    pageContentElement.innerHTML = pageContent;

    try {
        setupPopovers();
        setupClocks();
        await setupCalendars();
        // ... 十几个 setup 函数
        setupDynamicRelativeTime();   // 只更新相对时间文字，不重新拉内容
        setupLazyImages();
    } finally {
        pageElement.classList.add("content-ready");
        // ...
    }
}
```

[fetchPageContent()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/static/js/page.js#L6-L13) 极简：

```js
async function fetchPageContent(pageData) {
    // TODO: handle non 200 status codes/time outs
    // TODO: add retries
    const response = await fetch(`${pageData.baseURL}/api/pages/${pageData.slug}/content/`);
    const content = await response.text();
    return content;
}
```

两个 TODO 是开发者自己埋下的可靠性伏笔，详见 8.3 节。

**后端处理（glance.go）**：

请求进入 [handlePageContentRequest()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/glance.go#L334-L367)：

```go
func (a *application) handlePageContentRequest(w http.ResponseWriter, r *http.Request) {
    page, exists := a.slugToPage[r.PathValue("page")]
    // ... 鉴权

    pageData := templateData{Page: page}

    func() {
        page.mu.Lock()           // ← 1. 拿整页全局互斥锁
        defer page.mu.Unlock()

        page.updateOutdatedWidgets()   // ← 2. 同步更新所有过期 widget
        err = pageContentTemplate.Execute(&responseBytes, pageData)  // ← 3. 渲染整页模板
    }()

    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        w.Write([]byte(err.Error()))
        return
    }
    w.Write(responseBytes.Bytes())
}
```

[page 结构体](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/config.go#L77-L92) 定义了这个互斥锁：

```go
type page struct {
    Title    string
    Columns  []struct { ... Widgets widgets ... }
    mu       sync.Mutex `yaml:"-"`   // 每个 page 实例一把锁
    // ...
}
```

[updateOutdatedWidgets()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/glance.go#L233-L270) 并发更新所有过期 widget，但**等全部完成才返回**：

```go
func (p *page) updateOutdatedWidgets() {
    now := time.Now()
    var wg sync.WaitGroup
    context := context.Background()   // ← 根 context，无超时

    for c := range p.Columns {
        for w := range p.Columns[c].Widgets {
            widget := p.Columns[c].Widgets[w]
            if !widget.requiresUpdate(&now) {
                continue
            }
            wg.Add(1)
            go func() {
                defer wg.Done()
                widget.update(context)
            }()
        }
    }
    wg.Wait()   // ← 阻塞到所有 goroutine 完成
}
```

整条路径的关键特征：**锁 → 所有过期 widget 更新（wg.Wait）→ 整页渲染 → 解锁**，四步在同一个互斥锁临界区内串行完成。

### 8.2 为何高频轮询被硬性限制

如果前端加 `setInterval(fetchPageContent, 60000)`（每分钟刷一次）来追求直播状态新鲜度，会撞到三层瓶颈：

#### 瓶颈一：page.mu 全局互斥锁 —— 串行化所有请求

同一个 page slug 的所有请求都争抢 `page.mu` 同一把锁。场景：

1. 用户 A 在 0 秒请求 → 拿到锁，开始更新 20 个 widget
2. 用户 B 在 0.5 秒请求 → `Lock()` 阻塞等待
3. 用户 C 在 1 秒请求 → 同样阻塞排队
4. 所有并发请求被强制串行化，响应时间 = 排队数 × 单次处理时间

锁在整个 `updateOutdatedWidgets()` + `Execute()` 期间都持有，而不是只在写共享数据时持有。

#### 瓶颈二：wg.Wait() 木桶效应 —— 最慢 widget 决定整页延迟

`updateOutdatedWidgets()` 用 `wg.Wait()` 等所有过期 widget 全部完成，哪怕只有 1 个 widget 还在跑，锁就不会释放。

具体到 Videos widget + 直播 API 的场景：
- 30 个 YouTube RSS feed，单个请求超时 5 秒（`defaultClientTimeout`），并发 30 worker，最差 5 秒完成
- 如果直播 API 响应慢（比如 YouTube Data API 偶尔 3~4 秒），就被它拖慢整页
- 页面上其他慢 widget（天气、日历、DNS 统计等）也都会被等

哪怕你只想刷新直播状态，也必须等整页所有过期 widget 都更新完，**没有"单 widget 刷新"的接口**（`handleWidgetRequest()` 目前还是 501 Not Implemented）。

#### 瓶颈三：context.Background() 无请求级超时 —— 理论上可以永久挂死

`updateOutdatedWidgets()` 传入的是 `context.Background()`，没有任何 deadline。虽然单个 HTTP 请求有 `defaultHTTPClient.Timeout = 5 * time.Second` 保护，但：

- 如果某个 widget 的 `update()` 里做了多次串行 HTTP 请求（比如直播 API 先拿 token 再查状态再拿预览图），总时间可以远超 5 秒
- 如果 widget 里有 CPU 密集计算或死循环（理论 bug），没有任何机制打断它
- 锁会被一直持有，后续所有请求全部排队超时

Go `net/http` server 默认有 `ReadTimeout` / `WriteTimeout`，可以从外层切断连接，但 `updateOutdatedWidgets()` 的 goroutine 不会被取消，会一直在后台跑，锁也一直不释放。

三层瓶颈叠加的结论：**前端轮询间隔不能短于"最慢 widget 更新时间 × 并发用户数"**。假设 10 个用户同时访问，每个更新平均 3 秒，轮询间隔至少要 30 秒才不会导致请求堆积。要做 1 分钟级的直播刷新，单用户还行，多用户会直接被锁排队拖垮。

### 8.3 超时与非 200 响应的缺失应对措施

刷新链路在前后端都缺少可靠性保障，而且是开发者明知道的缺口（代码里直接写了 TODO）。

#### 8.3.1 前端 fetchPageContent：三项全缺

[fetchPageContent()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/static/js/page.js#L6-L13) 的两行 TODO 说明了一切：

```js
// TODO: handle non 200 status codes/time outs
// TODO: add retries
const response = await fetch(`${pageData.baseURL}/api/pages/${pageData.slug}/content/`);
const content = await response.text();
```

逐项拆解：

**缺 1：无超时（AbortController）**

`fetch()` 默认没有超时限制。如果后端因为锁排队迟迟不响应，浏览器会一直等到 TCP keepalive 超时（通常几分钟），期间页面一直处于 loading 状态。没有 `AbortController` + `signal` 来主动中断。

**缺 2：无状态码检查**

`response.text()` 不管 `response.ok` 与否都会执行。后端返回 500 Internal Server Error 时，响应体是 `err.Error()` 的纯文字：

```go
if err != nil {
    w.WriteHeader(http.StatusInternalServerError)
    w.Write([]byte(err.Error()))   // 比如 "template execution error: ..."
    return
}
```

这段错误文字会被直接塞进 `pageContentElement.innerHTML`，用户看到页面上铺满 Go 错误信息，同时 `setupPopovers()` 等后续 JS 因为 DOM 结构不对而抛异常，`finally` 里的 `content-ready` 类可能加上了，但页面已经处于不可用状态。

401 / 403 鉴权失败也一样，`showUnauthorizedJSON` 返回的是 JSON，被当 HTML 渲染出来就是 `{"error":"unauthorized"}`。

**缺 3：无重试逻辑**

网络波动导致的一次性失败没有重试。`setupPage()` 只在页面加载时调用一次，失败了就永远失败，用户必须手动 F5。

如果加了 `setInterval` 做轮询，一次失败不会自动重试，要等下一个周期，最坏情况下直播状态会停滞 2 个周期以上。

#### 8.3.2 后端 handler：缺少请求级超时和优雅降级

[handlePageContentRequest()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/glance.go#L334-L367) 本身的可靠性也不足：

**缺 1：无 handler 级 timeout**

没有 `context.WithTimeout(r.Context(), 10*time.Second)` 之类的保护。`r.Context()` 是 net/http 给的请求 context，客户端断连会取消它，但如果客户端不主动断（比如浏览器挂着），handler 可以一直跑。

**缺 2：非模板错误一律返回 200**

只有模板执行 `Execute()` 失败才会返回 500。而 widget 内部的失败（RSS 超时、直播 API 403 等）都被 `canContinueUpdateAfterHandlingErr` 吞掉，变成 widget header 上的小图标，HTTP 响应状态码仍然是 `200 OK`。前端无法通过状态码判断"这次刷新是否有效"，只能靠解析 HTML 找错误提示。

**缺 3：无快速失败 / 降级响应**

如果 `page.mu` 已经被持有了 5 秒以上，说明前面的请求很慢，当前请求大概率也会慢。没有 `TryLock()` 之类的机制来快速返回 503 Service Unavailable 或返回缓存的旧内容，而是一律阻塞等待。

#### 8.3.3 单个 HTTP 请求层面：有基本保护但不传播

作为对比，单个 widget 内部的 HTTP 请求是有保护的：

- [defaultHTTPClient](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-utils.go#L24-L32) 有 `Timeout: 5 * time.Second`
- [decodeJsonFromRequest()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-utils.go#L62-L93) 和 [decodeXmlFromRequest()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-utils.go#L102-L133) 都会检查 `response.StatusCode != http.StatusOK`，非 200 会返回格式化错误
- RSS widget 还额外处理了 `304 Not Modified`（[widget-rss.go L222-L224](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-rss.go#L222-L224)）

但这些错误只会在 widget 内部变成 `Error` / `Notice`，不会上升到 HTTP 响应状态码，前端无从感知。

### 8.4 对直播状态刷新的具体影响

| 限制 | 对直播刷新的实际后果 |
|------|-------------------|
| page.mu 全局锁 | 多用户同时轮询时请求排队，直播状态延迟从"1 分钟"变成"N 分钟"，N 取决于排队长度 |
| wg.Wait() 木桶效应 | 直播 API 本身快但其他慢 widget（天气、日历等）也会让锁不释放，拖慢直播刷新 |
| context.Background() 无超时 | 某个 widget 卡住时，整页直播刷新永久停滞，前端只能等浏览器层超时 |
| fetch 无 AbortController | 后端慢响应时前端无法中断，发起下一轮轮询会造成"请求堆积"（旧请求还在等，新请求又发了） |
| 无状态码检查 | 后端 500 时页面渲染错误文本，直播徽章消失，用户以为主播全下播了 |
| 无重试 | 直播 API 一次偶发 5xx，直播状态要等到下一个轮询周期才恢复，最坏滞后 2 个周期 |
| 无单 widget 刷新接口 | 哪怕只关心直播状态，每次都要拉整页 HTML，带宽和 CPU 都浪费 |
| widget 错误不传播到 HTTP 状态码 | 前端无法区分"刷新成功但没人直播"和"刷新失败"，都显示为没有 LIVE 徽章 |

对直播用户体验的连锁影响是：**LIVE 徽章的出现和消失都可能延迟数分钟甚至完全错乱**，而用户又得不到任何"数据可能已过期"的提示——跟 Twitch 直播间那种"直播中断了但页面几秒内就切换到离线画面"的体验完全不在一个量级。

### 8.5 改造优先级建议

如果要做直播功能，按影响面从小到大排序：

1. **P0 必须做**：`fetchPageContent()` 加 `AbortController` 超时 + `response.ok` 检查 + 失败重试（2~3 次指数退避），这三行 TODO 必须先填上
2. **P0 必须做**：新增 `/api/widgets/{id}` 单 widget 刷新接口（补全 `handleWidgetRequest`），直播状态只刷自己，不拖整页
3. **P1 应该做**：`handlePageContentRequest` 加 `context.WithTimeout(r.Context(), 10*time.Second)`，把超时从单个 HTTP 请求提升到整个 handler 级别
4. **P2 可以做**：`page.mu` 临界区缩小，`wg.Wait()` 完成后就解锁，模板渲染可以拿读锁或无锁执行（widget 数据已写入完毕）
5. **P2 可以做**：widget 错误汇总成 HTTP 响应头（如 `X-Glance-Widget-Errors: 2`），前端据此判断是否要提示用户

---

## 九、调用链总览

```
配置加载 (config.go)
    ↓ newWidget("videos") → videosWidget{}
    ↓ initialize()
        ├─ Playlists 加前缀 → 并入 Channels
        └─ withCacheDuration(time.Hour)   // 应用层缓存 1h
    ↓
用户浏览器加载页面 (page.js setupPage)
    ↓ fetchPageContent() 仅调用一次
        │  ⚠ 无 AbortController 超时  ⚠ 无 response.ok 检查  ⚠ 无重试（代码中标 TODO）
    ↓ HTTP GET /api/pages/{slug}/content/
        ↓ handlePageContentRequest()
            │  ⚠ 无 handler 级 context 超时
            │  ⚠ widget 内部错误一律返回 200，不体现在状态码
            ↓ page.mu.Lock()  ← 🔒 整页全局互斥锁，所有同 slug 请求在此串行排队
                ↓ page.updateOutdatedWidgets(context.Background())
                    │  ⚠ 根 context，无 deadline
                    ↓ 遍历所有 widget，调 requiresUpdate(now)
                        ├─ nextUpdate.IsZero() → 首次必刷
                        └─ now.After(nextUpdate) → 到期才刷
                            ↓ widget.update(ctx)  并发 goroutine 执行
                                │  wg.Add(1) / wg.Done()
                                ├─ ... 其他 widget（天气、日历等，木桶效应）
                                └─ videosWidget.update()
                                    ↓ fetchYoutubeChannelUploads(Channels, ...)
                                        ├─ 按前缀/UC 判定 → 构造 playlist_id 或 channel_id URL
                                        ├─ 30 worker 并发 GET YouTube RSS（单请求 5s 超时）
                                        ├─ XML → youtubeFeedResponseXml
                                        ├─ TimePosted 解析失败→time.Now→置顶
                                        ├─ 跨频道 sortByNewest()
                                        └─ 截断到 Limit 条
                                    ↓ canContinueUpdateAfterHandlingErr(err) 状态机：
                                        ├─ errNoContent
                                        │   ├─ ContentAvailable==false → 全屏 ERROR，下一次 1m/4m/9m... 后
                                        │   └─ ContentAvailable==true  → 旧内容 + 红色 major 图标，保留旧 Videos
                                        ├─ errPartialContent → 部分内容 + 黄色 minor 图标，Videos 被更新
                                        └─ nil → 干净内容，scheduleNextUpdate() 归零退避，1h 后再见
                ↓ wg.Wait()  ← ⏳ 阻塞到所有过期 widget 全部完成，最慢者决定锁释放时间
                ↓ pageContentTemplate.Execute()  ← 整页渲染，仍在锁内
            ↓ page.mu.Unlock()  ← 🔓 释放锁
            ↓ 返回 HTTP 响应（只有模板失败才 500，其余一律 200）
    ↓ response.text()  ← ⚠ 500 错误文本也会被当 HTML 塞进页面
    ↓ pageContentElement.innerHTML = content
    ↓ setupPopovers / setupCarousels / setupDynamicRelativeTime 等
        ⚠ setupDynamicRelativeTime 每分钟只刷新相对时间文字，不重新拉内容
    ↓ 页面进入 content-ready 状态，此后无自动内容刷新
        ↓ Render()
            ├─ style=="grid-cards" → videos-grid.html → 复用 video-card-contents.html
            ├─ style=="vertical-list" → videos-vertical-list.html（完全独立分支）
            └─ default → videos.html → 横向轮播，复用 video-card-contents.html
                └─ <img loading="lazy" src="YouTube CDN">  ← 浏览器懒加载+CDN缓存
```

整个流程的核心文件：

- [widget-videos.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-videos.go) — 视频源聚合主逻辑
- [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget.go) — widget 基类（ContentAvailable 单向锁、Error/Notice 状态机、指数退避重试）
- [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/glance.go) — 请求驱动刷新链路（handlePageContentRequest → page.mu 锁 → updateOutdatedWidgets → wg.Wait）
- [config.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/config.go#L77-L92) — page 结构体定义（含全局互斥锁 `mu sync.Mutex`）
- [widget-utils.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-utils.go) — defaultHTTPClient（5s 超时）、并发 worker pool、XML/JSON 解码
- [widget-twitch-channels.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-twitch-channels.go) — 直播状态参考实现（稳定排序、LIVE UI、10min 缓存）
- [widget-rss.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-rss.go) — ETag/Last-Modified 条件缓存、304 处理参考实现
- [video-card-contents.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/video-card-contents.html) — 默认/网格样式卡片子模板
- [videos-vertical-list.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/videos-vertical-list.html) — 纵向列表独立渲染模板
- [widget-base.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/widget-base.html) — ERROR 面板、major/minor notice 图标条件渲染
- [page.js](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/static/js/page.js) — fetchPageContent（无超时/状态码检查/重试，标了 TODO）、轮播、折叠、懒加载、相对时间更新（不做内容轮询）
