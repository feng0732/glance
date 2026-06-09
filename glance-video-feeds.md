# Glance 视频源聚合拉取路径全解析

本文档顺着代码调用链，讲清频道识别、元数据组装、封面缓存、直播状态预留，以及纵向列表的独立渲染分支、抓取失败后的重试机制、发布时间解析失败对排序的影响、直播状态接入与刷新频率/排序/展示样式的关联。

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

## 五、抓取失败后的重试机制

重试逻辑不在 Videos widget 自身，而是继承自 `widgetBase`，通过 `canContinueUpdateAfterHandlingErr()` + `scheduleEarlyUpdate()` 两段组合实现。

### 5.1 错误分级与调度入口

[widgetBase.canContinueUpdateAfterHandlingErr()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget.go#L293-L325)：

```go
func (w *widgetBase) canContinueUpdateAfterHandlingErr(err error) bool {
    if err != nil {
        w.scheduleEarlyUpdate()          // 只要有错误，一定走早重试

        if !errors.Is(err, errPartialContent) {
            w.withError(err)             // 严重错误：显示红色 ERROR 面板
            w.withNotice(nil)
            return false                 // 不保留旧数据
        }

        w.withError(nil)
        w.withNotice(err)                // 部分错误：右上角黄色小图标
        return true                      // 保留已拿到的部分数据
    }

    w.withNotice(nil)
    w.withError(nil)
    w.scheduleNextUpdate()              // 无错误：按正常间隔排期
    return true
}
```

对应 Videos widget 的两种失败场景：

| 场景 | fetchYoutubeChannelUploads 返回值 | 行为 |
|------|----------------------------------|------|
| 全部频道请求失败 | `errNoContent` | 显示 ERROR 面板，下次按指数退避早重试 |
| 部分频道请求失败 | `errPartialContent` + 已有视频 | 右上角黄色感叹号，展示成功的视频，同时早重试 |
| 全部成功 | `nil` | 正常排 1 小时后的下一次刷新 |

`withError` / `withNotice` 的 UI 表现见 [widget-base.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/widget-base.html#L21-L25)：严重错误会把整个 widget 内容替换成 ERROR 卡片，轻微错误只是 header 上出现 `notice-icon-minor` 小圆点，鼠标悬停显示错误文字。

### 5.2 指数退避的早重试

[scheduleEarlyUpdate()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget.go#L350-L367)：

```go
func (w *widgetBase) scheduleEarlyUpdate() *widgetBase {
    w.updateRetriedTimes++
    if w.updateRetriedTimes > 5 {
        w.updateRetriedTimes = 5       // 最大退避次数封顶
    }

    // 第 n 次失败，等 n² 分钟后再试
    nextEarlyUpdate := time.Now().Add(time.Duration(math.Pow(float64(w.updateRetriedTimes), 2)) * time.Minute)
    nextUsualUpdate := w.getNextUpdateTime()

    // 取较早的那个，防止退避时间超过正常刷新周期
    if nextEarlyUpdate.After(nextUsualUpdate) {
        w.nextUpdate = nextUsualUpdate
    } else {
        w.nextUpdate = nextEarlyUpdate
    }
    return w
}
```

退避时间表（Videos widget 正常周期 1h）：

| 失败次数 | 早重试间隔 | 实际下次刷新 |
|---------|-----------|-------------|
| 1 | 1 分钟后 | 1 分钟后 |
| 2 | 4 分钟后 | 4 分钟后 |
| 3 | 9 分钟后 | 9 分钟后 |
| 4 | 16 分钟后 | 16 分钟后 |
| 5 | 25 分钟后 | 25 分钟后 |
| 6+ | 仍按 25 分钟 | 25 分钟后封顶 |

一旦某次请求成功，[scheduleNextUpdate()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget.go#L343-L348) 会把 `updateRetriedTimes` 归零，下次又回到 1 小时正常周期。

### 5.3 worker 池层面的容错

并发请求在 [workerPoolDo()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-utils.go#L184-L242) 里不会因为单个任务失败就中止整个批次。每个 worker 的结果和错误分别放到独立数组，Videos widget 在拿到 `errs[i]` 后只是 `slog.Error` 记一条日志并 `continue`，不影响其他频道的视频入库。

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

## 七、直播状态接入：刷新频率、排序与展示样式的关联

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

**没有任何直播相关字段**（如 `IsLive`、`LiveSince`、`ViewersCount`）。对应的前端模板 [video-card-contents.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/video-card-contents.html) 和 [videos-vertical-list.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/videos-vertical-list.html) 也没有 LIVE 徽章、观看人数等 UI。

原因：Videos widget 当前只消费 YouTube 的 RSS feed，而 RSS feed 只包含"已发布的上传视频"，不含直播调度信息。要拿到直播状态需要额外走 YouTube Data API / GQL。

### 7.2 参照：Twitch Channels widget 的完整实现

项目中 Twitch 频道 widget（[widget-twitch-channels.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-twitch-channels.go)）是一份完整的"直播状态怎么接"的参考实现，也展示了刷新频率、排序、样式三者的绑定关系。

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

#### 刷新频率对比

| Widget | withCacheDuration | 原因 |
|--------|-------------------|------|
| videos | **1 小时** | 已上传视频变化慢，RSS 够了 |
| twitch-channels | **10 分钟** | 直播状态变化快，下播/开播都需要及时反映 |
| twitch-top-games | **10 分钟** | 同上 |

如果 Videos widget 接入 YouTube 直播，1 小时刷新肯定不够——主播开播 1 小时后用户才看到就失去了意义。合理的做法是**双轨刷新**：

- RSS 拉上传视频：仍然 1 小时
- 直播 API 拉在播状态：5~10 分钟（与 Twitch 同量级）

也可以只给配置了特定频道的用户启用直播轮询，避免所有 Videos widget 都加 API 调用开销。

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

### 7.3 接入影响总览

| 维度 | 当前 Videos | 接入直播后需要变更 |
|------|------------|------------------|
| 刷新频率 | 1 小时 | 拆成双轨：视频 RSS 1h + 直播 API 5~10min |
| video 结构体 | 6 字段 | 加 `IsLive`、`LiveSince`、`LiveViewers`、`StreamTitle`、`Category` |
| 排序 | `sort.Slice` 按时间 | 加 `sort.SliceStable` 直播置顶模式，离线用哨兵值避免 |
| 默认/网格样式 | `video-card-contents.html` | 加 LIVE 徽章、观众数、预览 popover |
| 纵向列表样式 | `videos-vertical-list.html` 独立分支 | 单独加 LIVE 标签、直播时长，或先重构复用子模板 |
| 错误图标 | `notice-icon-minor`（黄色） | 直播 API 单独超时不要影响已上传视频展示 |

---

## 八、调用链总览

```
配置加载 (config.go)
    ↓ newWidget("videos") → videosWidget{}
    ↓ initialize()
        ├─ Playlists 加前缀 → 并入 Channels
        └─ withCacheDuration(time.Hour)   // 应用层缓存 1h
    ↓
页面请求 (glance.go handlePageContentRequest)
    ↓ page.updateOutdatedWidgets()
        ↓ widget.requiresUpdate() 判断是否到刷新时间
            ↓ widget.update(ctx)
                ↓ fetchYoutubeChannelUploads(Channels, ...)
                    ├─ 按前缀/UC 判定 → 构造 playlist_id 或 channel_id URL
                    ├─ 30 worker 并发 GET YouTube RSS
                    ├─ XML → youtubeFeedResponseXml
                    ├─ 每条 entry → video 结构体（TimePosted 解析失败→time.Now→置顶）
                    ├─ 跨频道 sortByNewest()
                    └─ 截断到 Limit 条
                ↓ 错误处理（widgetBase.canContinueUpdateAfterHandlingErr）
                    ├─ errNoContent → 红色 ERROR + 指数退避重试（1m,4m,9m,16m,25m）
                    ├─ errPartialContent → 黄色感叹号 + 保留部分数据 + 退避重试
                    └─ nil → scheduleNextUpdate()，1h 后再见
    ↓ Render()
        ├─ style=="grid-cards" → videos-grid.html → 复用 video-card-contents.html
        ├─ style=="vertical-list" → videos-vertical-list.html（完全独立分支）
        └─ default → videos.html → 横向轮播，复用 video-card-contents.html
            └─ <img loading="lazy" src="YouTube CDN">  ← 浏览器懒加载+CDN缓存
```

整个流程的核心文件：

- [widget-videos.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-videos.go) — 视频源聚合主逻辑
- [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget.go) — widget 基类（缓存调度、错误处理、指数退避重试）
- [widget-utils.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-utils.go) — 并发 worker pool、XML/JSON 解码
- [widget-twitch-channels.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-twitch-channels.go) — 直播状态参考实现（刷新频率、稳定排序、LIVE UI）
- [widget-rss.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-rss.go) — ETag/Last-Modified 条件缓存参考实现
- [video-card-contents.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/video-card-contents.html) — 默认/网格样式卡片子模板
- [videos-vertical-list.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/videos-vertical-list.html) — 纵向列表独立渲染模板
- [widget-base.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/widget-base.html) — ERROR 面板、notice 图标
- [page.js](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/static/js/page.js) — 轮播、折叠、懒加载图片、相对时间更新
