# Glance 视频源聚合拉取路径全解析

本文档顺着代码调用链，逐一讲清四个核心环节：频道识别、元数据组装、封面缓存、直播状态预留。

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

## 四、直播状态预留：当前缺失与参照实现

### 4.1 Videos widget 的现状

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

**没有任何直播相关字段**（如 `IsLive`、`LiveSince`、`ViewersCount`）。对应的前端模板 [video-card-contents.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/video-card-contents.html) 也没有 LIVE 徽章、观看人数等 UI。

原因：Videos widget 当前只消费 YouTube 的 RSS feed，而 RSS feed 只包含"已发布的上传视频"，不含直播调度信息。要拿到直播状态需要额外走 YouTube Data API / GQL。

### 4.2 参照：Twitch Channels widget 的完整实现

项目中 Twitch 频道 widget（[widget-twitch-channels.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-twitch-channels.go)）是一份完整的"直播状态怎么接"的参考实现。

数据结构 [twitchChannel](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-twitch-channels.go#L62-L73)：

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

判定逻辑在 [fetchChannelFromTwitchTask()](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-twitch-channels.go#L133-L206)：

1. 一次 POST 请求携带两个 GraphQL Operation：
   - `ChannelShell` — 拿基础信息 + 是否有 stream（粗略判断直播状态）
   - `StreamMetadata` — 拿开播时间、分类、上次直播标题
2. 如果 `channelShell.UserOrError.Stream != nil` 则 `IsLive = true`
3. 离线时把 `ViewersCount` 设为 `-1`，这样排序时不会排在 0 观众的真直播频道前面

排序也做了两种模式 [sortByViewers / sortByLive](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-twitch-channels.go#L77-L87)。

### 4.3 预留接入点

如果未来 Videos widget 要加 YouTube 直播状态，可以从以下几点切入：

1. 在 `video` 结构体上新增 `IsLive bool`、`LiveViewers int`、`LiveScheduledFor time.Time` 等字段
2. `fetchYoutubeChannelUploads()` 之外，对检测到的直播频道额外调一次 YouTube Data API `liveBroadcasts` 或 GQL
3. 把拿到的直播数据 merge 进 `videos` 切片，并在 `sortByNewest()` 中把正在直播的条目置顶（参考 Twitch `sortByLive()` 用 `sort.SliceStable` 保证同状态顺序不抖）
4. 前端模板新增 LIVE badge 样式，参考 [widget-twitch-channels.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/twitch-channels.html)

---

## 五、调用链总览

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
                    ├─ 每条 entry → video 结构体（含缩略图 URL、标题、作者…）
                    ├─ 跨频道 sortByNewest()
                    └─ 截断到 Limit 条
    ↓ Render()
        ├─ videos.html / videos-grid.html / videos-vertical-list.html
        └─ video-card-contents.html → <img loading="lazy" src="YouTube CDN">
```

整个流程的核心文件：

- [widget-videos.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-videos.go) — 视频源聚合主逻辑
- [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget.go) — widget 基类（缓存调度、错误处理）
- [widget-utils.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-utils.go) — 并发 worker pool、XML/JSON 解码
- [widget-twitch-channels.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-twitch-channels.go) — 直播状态参考实现
- [widget-rss.go](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/widget-rss.go) — ETag/Last-Modified 条件缓存参考实现
- [video-card-contents.html](file:///d:/fz/0601/solo-dogfeeding/code/141-glance/internal/glance/templates/video-card-contents.html) — 前端渲染模板
