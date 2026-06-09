# Glance Social Feeds 数据源接入分析

## 一、整体架构

所有论坛类 widget（Reddit、HackerNews、Lobsters）共享统一的数据模型 `forumPost` 和通用的调度/重试/降级机制（由 `widgetBase` 提供）。数据源拉取发生在 `update(ctx)` 方法中，通过页面请求触发 `updateOutdatedWidgets()` 并发执行。

```
Page Request
    └─> updateOutdatedWidgets() [glance.go#L233-L270]
            └─> widget.update(ctx)  // 并发 goroutine
                    └─> widget.canContinueUpdateAfterHandlingErr(err)  // 统一错误处理
```

## 二、请求构造

### 2.1 HackerNews

HackerNews 使用官方 Firebase API，采用 **两阶段拉取**：先取帖子 ID 列表，再并发拉取详情。

**第一阶段：拉取帖子 ID 列表**

[fetchHackerNewsPostIds](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-hacker-news.go#L78-L86)

```
GET https://hacker-news.firebaseio.com/v0/{sort}stories.json
```

- `sort` 取值：`top` / `new` / `best`（在 [initialize](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-hacker-news.go#L25-L44) 中校验，默认 `top`）
- 默认拉取 40 个 ID（硬编码在 [update](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-hacker-news.go#L47) 中），再按 `limit` 截断

**第二阶段：并发拉取帖子详情**

[fetchHackerNewsPostsFromIds](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-hacker-news.go#L88-L139)

```
GET https://hacker-news.firebaseio.com/v0/item/{id}.json
```

- 使用 `workerPoolDo` 工作池，30 个 worker 并发（[widget-hacker-news.go#L97](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-hacker-news.go#L97)）
- HTTP Client：`defaultHTTPClient`，超时 5s，连接池 10/host

### 2.2 Reddit

Reddit 的请求构造更复杂，支持 **匿名模式** 和 **OAuth App 模式**，并需要解决 Cloudflare/Reddit 的 JS 挑战获取 `loid` cookie。

**基础 URL 选择**

[fetchSubredditPosts](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-reddit.go#L167-L295)

| 模式 | Base URL | 认证方式 |
|------|----------|----------|
| 匿名 | `https://www.reddit.com` | 浏览器 UA + `loid` cookie |
| OAuth App | `https://oauth.reddit.com` | `Bearer {accessToken}` |

OAuth Token 获取：[fetchNewAppAccessToken](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-reddit.go#L297-L324)

```
POST https://www.reddit.com/api/v1/access_token
Body: grant_type=client_credentials
Auth: Basic Auth (client_id:client_secret)
Header: User-Agent: {appName}/1.0
```

Token 缓存：在内存中保存至 `tokenExpiresAt`，过期前 1 分钟触发刷新（[widget-reddit.go#L183](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-reddit.go#L183)）。

**端点构造**

搜索模式：
```
{baseURL}/search.json?q={search} subreddit:{subreddit}&sort={sortBy}&limit={limit}
```

普通模式：
```
{baseURL}/r/{subreddit}/{sortBy}.json?t={topPeriod}&limit={limit}
```

- `sortBy`：`hot`（默认）/ `new` / `top` / `rising`
- `topPeriod`：`day`（默认）/ `hour` / `week` / `month` / `year` / `all`
- `limit`：仅当 >25 时显式传参（Reddit 默认返回 25 条）

**TLS 指纹伪装（uTLS）**

[redditHTTPClient](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-reddit.go#L329-L352)

Linux 下标准 `net/http` 的 TLS ClientHello 会被 Reddit/Cloudflare 识别并拦截。代码使用 `utls.HelloFirefox_Auto` 模仿 Firefox 的 TLS 握手指纹：

```go
uconn := utls.UClient(tcpConn, &utls.Config{ServerName: host}, utls.HelloFirefox_Auto)
```

**loid Cookie 获取（JS 挑战）**

[fetchRedditLoidCookie](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-reddit.go#L389-L456)

Reddit 的 `.json` 端点需要 `loid` cookie，流程：

1. `GET https://www.reddit.com/` 获取挑战页面
2. 正则提取 JS challenge 字符串和 CSRF token
3. 挑战解：`solution = challengeStr + challengeStr`（对应 JS 代码 `e + e`）
4. `GET https://www.reddit.com/?solution=...&js_challenge=1&token=...` 提交解答
5. 从响应 cookies 中提取 `loid`

全局共享 + 缓存（6 小时）+ Singleflight 合并请求：[getRedditLoidCookie](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-reddit.go#L362-L387)

**请求路由代理**

- `RequestURLTemplate`：将最终请求 URL 通过 `{REQUEST-URL}` 占位符包裹，走用户自定义代理
- `Proxy` 字段：配置独立 HTTP Proxy Client（[proxyOptionsField](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/config-fields.go#L190-L235)）

## 三、字段映射

统一目标结构 [forumPost](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-shared.go#L14-L26)：

| 字段 | 类型 | 说明 |
|------|------|------|
| Title | string | 帖子标题 |
| DiscussionUrl | string | 讨论区链接 |
| TargetUrl | string | 外部目标链接 |
| TargetUrlDomain | string | 目标链接域名（去 www.） |
| ThumbnailUrl | string | 缩略图 URL |
| CommentCount | int | 评论数 |
| Score | int | 分数/赞数 |
| Engagement | float64 | 计算出的互动度 |
| TimePosted | time.Time | 发布时间 |
| Tags | []string | 标签/flair |
| IsCrosspost | bool | 是否为转帖 |

### 3.1 HackerNews 字段映射

[widget-hacker-news.go#L119-L127](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-hacker-news.go#L119-L127)

| HN API 字段 | forumPost 字段 | 处理 |
|-------------|----------------|------|
| `title` | Title | 直接映射 |
| `id` | DiscussionUrl | `https://news.ycombinator.com/item?id={id}`，支持 `{POST-ID}` 模板替换 |
| `url` | TargetUrl | 直接映射（可为空，Ask HN 无 url） |
| `url` | TargetUrlDomain | `extractDomainFromUrl()` 解析 |
| `descendants` | CommentCount | 直接映射 |
| `score` | Score | 直接映射 |
| `time` | TimePosted | `time.Unix(seconds, 0)` |

### 3.2 Reddit 字段映射

[widget-reddit.go#L255-L291](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-reddit.go#L255-L291)

| Reddit API 字段 | forumPost 字段 | 处理 |
|-----------------|----------------|------|
| `title` | Title | `html.UnescapeString()` 反转义 |
| `permalink` | DiscussionUrl | `https://www.reddit.com{permalink}`，支持 `{SUBREDDIT}`/`{POST-ID}`/`{POST-PATH}` 模板 |
| `url` | TargetUrl | 仅当 `is_self=false` 时设置 |
| `domain` | TargetUrlDomain | 直接映射 |
| `thumbnail` | ThumbnailUrl | 过滤 `self`/`default`/`nsfw`/空值，HTML 反转义 |
| `num_comments` | CommentCount | 直接映射 |
| `ups` | Score | 直接映射 |
| `created` | TimePosted | `time.Unix(int64(created), 0)` |
| `link_flair_text` | Tags | 仅当 `show-flairs=true` 时追加 |
| `stickied`/`pinned` | - | 过滤跳过置顶/固定帖子 |
| `crosspost_parent_list[0]` | IsCrosspost, TargetUrl, TargetUrlDomain | 转帖时指向源帖子 |

### 3.3 Lobsters 字段映射（参考）

[widget-lobsters.go#L93-L102](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-lobsters.go#L93-L102)

| Lobsters 字段 | forumPost 字段 |
|---------------|----------------|
| `title` | Title |
| `comments_url` | DiscussionUrl |
| `url` | TargetUrl |
| `url` | TargetUrlDomain |
| `comment_count` | CommentCount |
| `score` | Score |
| `created_at` | TimePosted (RFC3339 解析) |
| `tags` | Tags |

## 四、频控（Rate Limiting）

### 4.1 缓存层 —— 最主要的频控手段

所有论坛 widget 默认缓存 **30 分钟**（Lobsters 为 1 小时）：

```go
// Reddit & HN
widget.withCacheDuration(30 * time.Minute)
// Lobsters
widget.withCacheDuration(time.Hour)
```

调度检查：[requiresUpdate](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget.go#L173-L183) —— 仅当 `now.After(nextUpdate)` 才触发实际请求。用户可通过 `cache:` YAML 字段自定义。

### 4.2 Reddit loid cookie 全局共享

通过闭包缓存 + Singleflight 避免频繁触发 JS 挑战：

- 缓存时长：6 小时（[widget-reddit.go#L370](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-reddit.go#L370)）
- 所有 Reddit widget 实例共享同一个 `loid`
- 并发请求合并（[Singleflight](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/singleflight.go)）：同一时刻只允许一个实际请求

### 4.3 HN 并发池限制

工作池固定 30 worker（[widget-hacker-news.go#L97](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-hacker-news.go#L97)），默认 10 worker，最多不超过请求数。

### 4.4 HTTP Client 连接池

`defaultHTTPClient` 的 `MaxIdleConnsPerHost: 10`，防止对同一主机建立过多连接。

## 五、重试机制

### 5.1 指数退避早期重试

[scheduleEarlyUpdate](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget.go#L350-L367)

```
重试间隔 = n² 分钟  (n 为已重试次数，上限 5)
```

即：1min → 4min → 9min → 16min → 25min。与正常缓存到期时间取较小者。

触发条件：`canContinueUpdateAfterHandlingErr(err)` 中任何 `err != nil`。

### 5.2 Reddit loid cookie 优雅降级

[getRedditLoidCookie](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-reddit.go#L374-L380)

获取新 cookie 失败时，若有缓存值则返回缓存并打印警告，而非直接报错：

```go
if cachedLoid != "" {
    fmt.Printf("Error fetching new reddit loid cookie, using cached value: %v\n", err)
    return cachedLoid, nil
}
```

### 5.3 Reddit OAuth Token 预刷新

提前 1 分钟检查 token 是否即将过期，避免请求时才发现失效（[widget-reddit.go#L183](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-reddit.go#L183)）。

## 六、降级路径（Error Handling & Fallback）

### 6.1 统一错误处理入口

[canContinueUpdateAfterHandlingErr](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget.go#L293-L325)

```
err != nil
├─> scheduleEarlyUpdate()            // 指数退避重试调度
├─> 错误是 errPartialContent ?
│       ├─ 是：withNotice(err)       // 显示为"警告"级别提示，保留已有内容，继续更新
│       └─ 否：withError(err)        // 显示为"错误"级别，widget 进入错误态，终止本次更新
└─> err == nil
    └─> scheduleNextUpdate()         // 正常缓存调度，清除错误/提示
```

关键错误定义在 [widget-utils.go#L20-L22](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-utils.go#L20-L22)：
- `errNoContent`：完全没拿到内容 → 显示错误，终止
- `errPartialContent`：部分内容获取失败 → 显示通知，保留已有内容

### 6.2 HackerNews 单条帖子失败降级

[fetchHackerNewsPostsFromIds](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-hacker-news.go#L105-L109)

```go
if errs[i] != nil {
    slog.Error("Failed to fetch or parse hacker news post", ...)
    continue  // 跳过失败帖子，不中断整体
}
```

- 全部失败：返回 `errNoContent` → widget 错误态
- 部分失败：返回 `errPartialContent` → widget 显示通知，保留成功帖子
- 全部成功：无错误

### 6.3 Reddit 匿名模式失败链路

1. HTTP 请求失败 / 非 200 状态 → `decodeJsonFromRequest` 返回错误
2. loid cookie 获取失败 → 无缓存时直接返回 `could not solve reddit challenge`
3. 返回空帖子列表 → `no posts found` 错误
4. 以上任一错误 → `canContinueUpdateAfterHandlingErr` 调度重试 + 显示错误

### 6.4 内容可用性（ContentAvailable）

[withError](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget.go#L283-L291) 中：
```go
if err == nil && !w.ContentAvailable {
    w.ContentAvailable = true
}
```
首次成功获取内容后标记为可用，后续即使临时失败也可复用上次渲染结果（取决于模板层逻辑）。

### 6.5 渲染失败兜底

[renderTemplate](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget.go#L217-L241)

模板渲染失败时立即重新执行一次渲染（尝试渲染错误信息），避免页面因半开标签崩溃。两次均失败则留空 buffer。

## 七、互动度二次排序（Extra Sort）

三个论坛 widget 均支持 `extra-sort-by: engagement`：

[calculateEngagement](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-shared.go#L34-L58)

```
Engagement = (CommentCount / AvgComments + Score / AvgScore) / 2
```

时间衰减：发布超过 7 小时的帖子按线性折旧，24 小时后最大折旧 90%。

## 八、关键文件索引

| 文件 | 职责 |
|------|------|
| [widget-reddit.go](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-reddit.go) | Reddit 数据源：OAuth、uTLS 伪装、loid 挑战、字段映射 |
| [widget-hacker-news.go](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-hacker-news.go) | HN 数据源：两阶段拉取、30-worker 并发池 |
| [widget-lobsters.go](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-lobsters.go) | Lobsters 数据源：最简实现参考 |
| [widget-shared.go](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-shared.go) | `forumPost` 共享模型 + Engagement 排序 |
| [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget.go) | `widgetBase`：缓存调度、指数退避重试、错误/通知分级 |
| [widget-utils.go](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/widget-utils.go) | HTTP 编解码、工作池、UA 生成 |
| [singleflight.go](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/singleflight.go) | 并发请求合并（用于 loid cookie） |
| [config-fields.go](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/config-fields.go) | Proxy 配置解析 |
| [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/137-glance/internal/glance/glance.go) | 页面级并发 widget 更新触发 |
