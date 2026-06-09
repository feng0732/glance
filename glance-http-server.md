# Glance HTTP Server 路由与中间件执行顺序分析

## 一、整体调用链路

```
main.go:main()
  └─ internal/glance/main.go:Main()
        ├─ parseCliOptions()                # 解析命令行
        └─ serveApp(configPath)             # 启动 HTTP 服务
              ├─ parseYAMLIncludes()        # 解析配置文件(含 include)
              ├─ configFilesWatcher()       # 配置文件热重载监听
              └─ onChange() callback:
                    ├─ newConfigFromYAML()  # 构造 config
                    ├─ newApplication(cfg)  # 构造 application 实例
                    └─ app.server()         # 注册路由 & 返回 start/stop
                          └─ http.Server{ Handler: mux }
                                └─ startServer() → server.ListenAndServe()
```

## 二、路由注册顺序（代码路径）

路由注册全部集中在 [glance.go:server()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L436-L516)，按如下顺序依次写入 `http.NewServeMux()`：

| 序号 | 方法 | 路径模式 | Handler | 条件 |
|:---:|:---|:---|:---|:---|
| 1 | GET | `/{$}` | [handlePageRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L306-L332)（首页 page=空） | 始终 |
| 2 | GET | `/{page}` | [handlePageRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L306-L332) | 始终 |
| 3 | GET | `/api/pages/{page}/content/{$}` | [handlePageContentRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L334-L367) | 始终 |
| 4 | POST | `/api/set-theme/{key}` | [handleThemeChangeRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/theme.go#L15-L39) | `!Theme.DisablePicker` |
| 5 | 任意 | `/api/widgets/{widget}/{path...}` | [handleWidgetRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L404-L425)（目前返回 501） | 始终 |
| 6 | GET | `/api/healthz` | 内联 200 OK | 始终 |
| 7 | GET | `/login` | [handleLoginPageRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L323-L343) | `RequiresAuth` |
| 8 | GET | `/logout` | [handleLogoutRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L306-L309) | `RequiresAuth` |
| 9 | POST | `/api/authenticate` | [handleAuthenticationAttempt](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L134-L248) | `RequiresAuth` |
| 10 | GET | `/static/{hash}/{path...}` | 内联静态文件服务（带缓存） | 始终 |
| 11 | GET | `/static/{hash}/css/bundle.css` | 内联直接返回 `bundledCSSContents`（覆盖序号10） | 始终 |
| 12 | GET | `/manifest.json` | 内联返回 pre-parsed manifest | 始终 |
| 13 | 任意 | `/assets/{path...}` | 用户目录静态文件服务（带缓存） | `Server.AssetsPath != ""` |

> **注意**：Go 1.22+ `http.ServeMux` 使用"最具体匹配优先"，因此序号 11 的精确路径会覆盖序号 10 的前缀匹配。

## 三、中间件/装饰器链

Glance **没有使用通用的中间件包装器模式**（如 `func(h http.Handler) http.Handler`），而是将各类横切关注点以**装饰器函数内联**或**在 Handler 内部显式调用**的方式散落在代码中。以下按请求流经的先后顺序整理。

### 3.1 异常恢复（Panic Recovery）

**位置**：Go 标准库 `net/http` 自身。

`http.Server` 在每个连接的 goroutine 内已经内置了 `recover()`，Handler 内部的 panic 不会让整个进程崩溃，只会终止当前请求并记录日志。

应用层**没有额外的 recover 中间件**，唯一显式依赖 panic-recovery 机制的场景在模板渲染：

- [widget-custom-api.go:Subrequest()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/widget-custom-api.go#L217-L229)：模板中调用 `.Subrequest(key)` 时，若 key 不存在直接 `panic()`，依赖 `html/template` 自身的 execute 流程捕获 panic，返回 error。
- [templates.go:mustParseTemplate()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/templates.go#L61-L71)：启动期模板解析失败会 `panic()`，让进程立刻退出（属于 fail-fast，非请求期恢复）。
- [embed.go:bundledCSSContents](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/embed.go#L87-L151)：启动期 CSS 打包失败同样 `panic()`。

请求期的模板 execute error 会被各 Handler 显式判断，返回 500：

- [handlePageRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L324-L329)
- [handlePageContentRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L357-L363)
- [handleLoginPageRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L335-L340)
- [widgetBase.renderTemplate](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/widget.go#L217-L241)：widget 模板渲染失败会标记 `ContentAvailable=false` 并记录 slog，**不会**让整个页面 500，而是尝试重新渲染错误状态。

### 3.2 反向代理头处理

**位置**：Handler 内部按需读取，无全局中间件。

| 头部 | 使用位置 | 行为 |
|:---|:---|:---|
| `X-Forwarded-For` | [addressOfRequest()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L369-L396) | 仅当 `Server.Proxied=true` 时生效；取逗号分隔的第一个 IP；空则回退到 `r.RemoteAddr`。 |
| `X-Forwarded-Proto` | [setAuthSessionCookie()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L311-L321) | 值为 `https`（不区分大小写）时，Cookie 才会设置 `Secure=true`。 |

> 代码备注明确写着："This should probably be configurable or look for multiple headers, not just this one" — 当前仅读 `X-Forwarded-For`，不读取 `X-Real-IP`、`Forwarded` (RFC 7239) 等。

`addressOfRequest()` 目前**仅**在 [handleAuthenticationAttempt](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L134-L248) 中用于登录失败限速（5 次 / 5 分钟窗口）。

### 3.3 静态资源服务与缓存

Glance 静态资源有三条独立路径：

#### (1) 内嵌静态资源（`/static/{hash}/...`）

```
mux.Handle(
    "GET /static/{hash}/{path...}",
    http.StripPrefix(
        "/static/"+staticFSHash,
        fileServerWithCache(http.FS(staticFS), 24h),
    ),
)
```

- `staticFS` 来自 [embed.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/embed.go#L20-L27)，通过 `//go:embed static` 嵌入二进制。
- `staticFSHash` 是启动时对整个 `static/` 目录做 MD5 取前 10 位（[computeFSHash()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/embed.go#L52-L81)），作为 URL 指纹实现强缓存。
- [fileServerWithCache()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/utils.go#L149-L158) 是一个轻量装饰器，在 `http.FileServer` 前统一加 `Cache-Control: public, max-age=86400`。
- 特殊路径 `/static/{hash}/css/bundle.css` 被单独拦截（序号 11），直接返回 [bundledCSSContents](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/embed.go#L87-L151) — 运行时把所有 `@import` 递归拼接、去注释、去行首空白、去换行，做了一个极简 CSS 打包。

#### (2) PWA Manifest（`/manifest.json`）

启动期用 [manifestTemplate](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L23) 渲染一次存到 `app.parsedManifest`，请求时直接写回，带同样的 24h `Cache-Control`。

#### (3) 用户自定义资源（`/assets/...`）

当 `Server.AssetsPath` 非空时：

```
mux.Handle(
    "/assets/{path...}",
    http.StripPrefix(
        "/assets/",
        fileServerWithCache(http.Dir(AssetsPath), 2h),
    ),
)
```

缓存时长 2 小时，读本地文件系统。配置中所有 `/assets/xxx` 的相对引用会通过 [resolveUserDefinedAssetPath()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L272-L278) 自动加上 `BaseURL` 前缀。

### 3.4 响应压缩

**结论：应用层没有实现任何 gzip / brotli / deflate 压缩中间件。**

虽然 `go.mod` 中 transitively 依赖了 `github.com/klauspost/compress` 和 `github.com/andybalholm/brotli`，但它们是 `golang.org/x/net` → `gofeed` → `goquery` 这条链带进来的，Glance 本身并未在 HTTP 响应路径上调用任何压缩 API。

生产环境下应在反向代理层（nginx / traefik / caddy）开启压缩。

### 3.5 BaseURL / 前缀处理

`Server.BaseURL` 在 [newApplication()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L196) 中被 `TrimRight("/")`。它不影响路由匹配（路由永远注册在根 `/`），但会出现在：

- 所有静态资源 URL 拼接：[StaticAssetPath()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L427-L429)、[VersionedAssetPath()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L431-L434)
- 登录/登出重定向：[auth.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L296)、[auth.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L308)、[auth.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L325)
- Cookie `Path`：[auth.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L317)、[theme.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/theme.go#L31)
- 用户资产解析：[resolveUserDefinedAssetPath()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L272-L278)

如果 Glance 部署在子路径下，需要在反向代理层把 `https://domain/glance/xxx` 反代为 `http://backend/xxx`，同时配置 `base-url: /glance`，否则前端资源路径和 Cookie 作用域会出错。

### 3.6 鉴权检查

鉴权同样**不是全局中间件**，而是在每个需要保护的 Handler 入口显式调用：

```go
if a.handleUnauthorizedResponse(w, r, redirectToLogin) {
    return
}
```

- [handlePageRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L313-L315) 使用 `redirectToLogin`（303 → `/login`）
- [handlePageContentRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L341-L343) 使用 `showUnauthorizedJSON`（401 + JSON）
- 静态资源、`/api/healthz`、`/manifest.json`、`/login`、`/api/authenticate` **没有**鉴权检查，匿名可访问。

[isAuthorized()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L250-L286) 内部流程：
1. 若 `RequiresAuth=false` 直接放行
2. 读取 Cookie `session_token` → base64 解码 → HMAC-SHA256 验签
3. 检查过期时间（14 天有效期）；剩余不足 7 天时自动重新签发新 Cookie

## 四、请求流时序图（以 GET /{page} 为例）

```
Client
  │  GET /dashboard
  ▼
http.Server (goroutine per conn, 内置 recover)
  │
  ▼
ServeMux 路由匹配 → 命中 GET /{page} → handlePageRequest
  │
  ├─ 从 slugToPage 查 page；不存在 → handleNotFound (404)
  │
  ├─ handleUnauthorizedResponse
  │    └─ isAuthorized
  │         ├─ 读 session_token Cookie
  │         ├─ verifySessionToken (验签 + 过期检查)
  │         └─ 临近过期 → generateSessionToken + Set-Cookie
  │    └─ 未授权 → 303 重定向到 BaseURL/login
  │
  ├─ populateTemplateRequestData （读 theme Cookie → 选主题）
  │
  └─ pageTemplate.Execute
       └─ widget.renderTemplate
            ├─ t.Execute 成功 → 返回 HTML
            └─ t.Execute 失败
                 ├─ ContentAvailable=false, Error=err, slog.Error
                 └─ 立即再次 Execute 渲染错误状态（避免 HTML 标签未闭合）
  │
  ▼
Response（无压缩；由反向代理负责压缩）
```

## 五、关键文件索引

| 文件 | 作用 |
|:---|:---|
| [main.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/main.go) | 二进制入口 |
| [internal/glance/main.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/main.go) | CLI 分发 + serveApp（配置热重载） |
| [internal/glance/glance.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go) | application 结构、路由注册、页面 Handler、X-Forwarded-For 解析 |
| [internal/glance/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go) | 登录/登出、会话 Token、Cookie（含 X-Forwarded-Proto）、鉴权检查 |
| [internal/glance/theme.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/theme.go) | 主题切换 Cookie、主题 CSS 输出 |
| [internal/glance/utils.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/utils.go) | fileServerWithCache 装饰器 |
| [internal/glance/embed.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/embed.go) | 内嵌 FS、静态资源 hash 计算、CSS 运行时打包 |
| [internal/glance/templates.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/templates.go) | mustParseTemplate（启动期 panic 机制） |
| [internal/glance/widget.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/widget.go) | widget 模板渲染的错误恢复 |
| [internal/glance/config.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/config.go) | config 结构（Server.Proxied / BaseURL / AssetsPath） |
