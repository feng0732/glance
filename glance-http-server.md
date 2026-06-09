# Glance HTTP Server 路由与中间件执行顺序分析

## 一、Server 启动路径（三条独立路径）

Glance 实际存在三条独立的 Server 启动路径，按优先级从高到低：

### 路径 A：v0.7 升级通知临时 Server（仅 Docker 环境触发）

```
main.go:main()
  └─ Main()
       └─ cliIntentServe 分支
            └─ serveUpdateNoticeIfConfigLocationNotMigrated()
                 ├─ 触发条件：isRunningInsideDockerContainer()
                 │       && 指定路径无 glance.yml
                 │       && 当前目录有 glance.yml 文件（非目录）
                 ├─ 构造独立 mux：
                 │    ├─ /static/  → 内嵌 FS（无 Cache-Control header）
                 │    └─ /        → 503 + HTML 升级提示页
                 └─ server.ListenAndServe()  ← 同步阻塞，永不返回
                      返回 true → Main() return 1 退出
```

代码位置：[serveUpdateNoticeIfConfigLocationNotMigrated()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/main.go#L183-L219)

> 此路径是一次性降级 Server，**不会继续进入 serveApp()**。

---

### 路径 B：配置文件监听器成功启动（正常热重载模式）

```
serveApp(configPath)
  ├─ parseYAMLIncludes(configPath)            # 同步解析配置（含 include）
  │
  ├─ configFilesWatcher(..., onChange, onErr)
  │    ├─ 创建 fsnotify.Watcher
  │    ├─ 注册所有 include 文件
  │    ├─ 启动后台 goroutine 监听 fsnotify 事件
  │    │
  │    └─ ↓ configFilesWatcher 返回前 ↓
  │         同步调用 onChange(lastContents)  ← 首次启动同步执行
  │
  └─ onChange(newContents) 回调执行：
       ├─ newConfigFromYAML()                 # 解析 YAML
       ├─ newApplication(config)              # 构造 application 实例
       │
       ├─ if stopServer != nil:
       │    stopServer()                       # ← 同步阻塞调用 server.Close()
       │                                         关闭 listener，强制断开连接
       │
       └─ go func() {                          # ← 新 goroutine 中启动新 Server
            startServer, stopServer = app.server()
            startServer() → ListenAndServe() 阻塞
          }()

  主线程：<-exitChannel 永久阻塞
```

代码位置：
- [serveApp()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/main.go#L93-L181)
- [configFilesWatcher() 末尾同步触发 onChange](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/config.go#L436)

---

### 路径 C：配置文件监听器失败（降级为无热重载模式）

当 `configFilesWatcher()` 返回 error 时走这条路径：

```
serveApp(configPath)
  ├─ parseYAMLIncludes(configPath)
  ├─ configFilesWatcher() 返回 error
  │
  ├─ log "Error starting file watcher, config file changes will require a manual restart"
  │
  ├─ newConfigFromYAML(configContents)    # 同步
  ├─ newApplication(config)               # 同步
  ├─ startServer, _ = app.server()        # stopServer 被 _ 丢弃
  └─ startServer()                         # ← 主线程同步阻塞 ListenAndServe()
```

> 此路径下 `stopServer` 被丢弃，配置变更不会触发热重载。

---

### 路径 B vs 路径 C 关键差异

| 维度 | 路径 B（watcher 成功） | 路径 C（watcher 失败） |
|:---|:---|:---|
| 首次启动时机 | `configFilesWatcher` 返回前同步调用 `onChange`，内部 goroutine 异步启动 Server | 主线程直接同步启动 Server |
| 主线程阻塞点 | `<-exitChannel` | `startServer()` 本身（`ListenAndServe` 阻塞） |
| 热重载 | 支持（500ms debounce） | 不支持 |
| `stopServer` 生命周期 | 闭包变量持续持有，reload 时可调用 | 被 `_` 丢弃 |
| 首次启动配置解析失败 | `close(exitChannel)` → 主线程解除阻塞 → `serveApp` return → 进程退出 | 直接 `return err` → `Main()` 返回 1 → 进程退出 |
| 运行中配置解析失败 | 仅 log，旧 Server 继续运行 | N/A（无热重载） |

---

## 二、热重载停启时序（路径 B）

```
时间轴 →

文件写入 → fsnotify.Write 事件
     │
     ├─ 500ms debounce（防抖：连续写入合并为一次）
     │
     ▼
onChange(newContents) 在 watcher goroutine 同步执行：
  │
  ├─ ① newConfigFromYAML() 失败？
  │    ├─ hadValidConfigOnStartup == false → close(exitChannel) → 进程退出
  │    └─ hadValidConfigOnStartup == true  → 仅 log，旧 Server 继续跑
  │
  ├─ ② newApplication() 失败？同上
  │
  ├─ ③ hadValidConfigOnStartup = true
  │
  ├─ ④ stopServer() — 同步阻塞：
  │    调用 server.Close()
  │    （立即关闭 listener，正在处理的连接被强制断开；
  │     不使用 Shutdown()，无优雅关闭）
  │
  └─ ⑤ go func() 启动新 Server：
       startServer, stopServer = app.server()    // 路由重新注册
       startServer() → ListenAndServe 阻塞

主线程始终在 <-exitChannel 等待

⚠️  竞态窗口：④ stopServer 关闭 listener 到 ⑤ 新 listener 绑定之间
     存在短暂端口真空，期间新连接会被拒绝。
```

关键代码：
- [stopServer 同步调用](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/main.go#L132-L136)
- [新 goroutine 启动 startServer](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/main.go#L138-L145)
- [server.Close()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L511-L513)

注意事项：
- 用 `server.Close()` 而非 `server.Shutdown(ctx)`，不等待正在进行的请求
- `startServer` 在新 goroutine 执行，旧 listener 关闭和新 listener 绑定之间没有同步屏障
- `hadValidConfigOnStartup` 标志用于区分"首次启动"和"运行中 reload"：首次启动失败会让进程退出，运行中 reload 失败则保留旧 Server

---

## 三、路由注册顺序（代码路径）

路由注册全部集中在 [glance.go:server()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L436-L516)，按如下顺序依次写入 `http.NewServeMux()`：

| 序号 | 方法限定 | 路径模式 | Handler | 注册条件 |
|:---:|:---|:---|:---|:---|
| 1 | GET | `/{$}` | [handlePageRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L306-L332)（首页 page=""） | 始终 |
| 2 | GET | `/{page}` | [handlePageRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L306-L332) | 始终 |
| 3 | GET | `/api/pages/{page}/content/{$}` | [handlePageContentRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L334-L367) | 始终 |
| 4 | POST | `/api/set-theme/{key}` | [handleThemeChangeRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/theme.go#L15-L39) | `!Theme.DisablePicker` |
| 5 | **不限** | `/api/widgets/{widget}/{path...}` | [handleWidgetRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L404-L425)（返回 501） | 始终 |
| 6 | GET | `/api/healthz` | 内联 200 OK | 始终 |
| 7 | GET | `/login` | [handleLoginPageRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L323-L343) | `RequiresAuth` |
| 8 | GET | `/logout` | [handleLogoutRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L306-L309) | `RequiresAuth` |
| 9 | POST | `/api/authenticate` | [handleAuthenticationAttempt](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L134-L248) | `RequiresAuth` |
| 10 | GET | `/static/{hash}/{path...}` | 内联静态文件服务（带 24h 缓存） | 始终 |
| 11 | GET | `/static/{hash}/css/bundle.css` | 内联直接返回 `bundledCSSContents` | 始终 |
| 12 | GET | `/manifest.json` | 内联返回 pre-parsed manifest（24h 缓存） | 始终 |
| 13 | **不限** | `/assets/{path...}` | 用户目录静态文件服务（带 2h 缓存） | `Server.AssetsPath != ""` |

> **Go 1.22+ ServeMux 匹配优先级**：
> - 精确路径 > 前缀路径（因此序号 11 的 `/static/{hash}/css/bundle.css` 精确匹配，优先于序号 10 的前缀匹配）
> - 查询字符串 `?v=xxx` 不参与路径匹配（`/manifest.json?v=123` 仍命中 `GET /manifest.json`）
> - 带方法限定的模式 > 不限方法的模式

---

## 四、中间件/装饰器链

Glance **没有使用通用的中间件包装器模式**（如 `func(h http.Handler) http.Handler`），所有横切关注点以 **装饰器函数内联** 或 **Handler 内部显式调用** 的方式散落。

---

### 4.1 异常恢复（Panic Recovery）

三层恢复机制，从外到内：

**层次一：标准库 `net/http` 层**

Go `http.Server` 在每个连接 goroutine 内内置 `recover()`。Handler 内部 panic 不会让进程崩溃，仅终止当前请求并输出 log。

**层次二：`html/template` 层**

Go `html/template.Execute` 执行期间，模板调用的函数（如 `.Subrequest(key)`）发生 panic，会被模板引擎捕获并转为 `Execute` 的 error 返回值，不会向上冒泡到 `http.Server`。

**层次三：应用层**

应用层**没有额外 recover 中间件**。显式 panic 场景：

| 位置 | 时机 | 行为 |
|:---|:---|:---|
| [widget-custom-api.go:Subrequest()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/widget-custom-api.go#L217-L229) | 请求期模板渲染 | 模板中调用 `.Subrequest(key)` 不存在时 panic，依赖 template 层捕获为 error |
| [templates.go:mustParseTemplate()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/templates.go#L61-L71) | 进程启动期 | 模板解析失败 panic → 进程退出（fail-fast） |
| [embed.go:bundledCSSContents](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/embed.go#L87-L151) | 进程启动期 | CSS 运行时打包失败 panic → 进程退出（fail-fast） |

**请求期模板 error 处理**（Handler 显式判断）：

- [handlePageRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L324-L329)：`err != nil` → 500 + error 文本
- [handlePageContentRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L357-L363)：`err != nil` → 500 + error 文本
- [handleLoginPageRequest](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L335-L340)：`err != nil` → 500 + error 文本
- [widgetBase.renderTemplate()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/widget.go#L217-L241)：渲染失败 → 标记 `ContentAvailable=false` 并 slog，**立即再次 Execute** 尝试渲染错误状态，避免 HTML 标签未闭合破坏整个页面

---

### 4.2 反向代理头处理

无全局中间件，Handler 内部按需读取：

| 头部 | 使用位置 | 生效条件 | 行为 |
|:---|:---|:---|:---|
| `X-Forwarded-For` | [addressOfRequest()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L369-L396) | `Server.Proxied == true` | 取逗号分隔第一个 IP；空或未配置则回退 `r.RemoteAddr` |
| `X-Forwarded-Proto` | [setAuthSessionCookie()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L311-L321) | 始终读取 | 值为 `https`（不区分大小写）时 Cookie 设置 `Secure=true` |

`addressOfRequest()` 目前**仅**在 [handleAuthenticationAttempt()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L134-L248) 中用于登录失败限速：5 分钟窗口内最多 5 次失败。

> 代码注释："This should probably be configurable or look for multiple headers, not just this one" — 当前仅读 `X-Forwarded-For`，不支持 `X-Real-IP`、RFC 7239 `Forwarded` 等头部。

---

### 4.3 静态资源缓存边界

静态资源存在**五条**独立的缓存/服务路径，各自策略不同：

#### (1) 内嵌静态资源 `/static/{hash}/...`（24h 强缓存）

```go
mux.Handle(
    "GET /static/"+staticFSHash+"/{path...}",
    http.StripPrefix(
        "/static/"+staticFSHash,
        fileServerWithCache(http.FS(staticFS), 24*time.Hour),
    ),
)
```

- `staticFS` 来自 [embed.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/embed.go#L20-L27)：`//go:embed static` 嵌入二进制。
- `staticFSHash`：包级 var，**进程启动时计算一次**（[embed.go:42-50](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/embed.go#L42-L50)），热重载不会变，进程重启才重新计算 MD5（取前 10 位十六进制）。
- `fileServerWithCache()` 装饰器实现：

```go
func fileServerWithCache(fs http.FileSystem, cacheDuration time.Duration) http.Handler {
    server := http.FileServer(fs)
    cacheControlValue := fmt.Sprintf("public, max-age=%d", int(cacheDuration.Seconds()))
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // TODO: fix always setting cache control even if the file doesn't exist
        w.Header().Set("Cache-Control", cacheControlValue)
        server.ServeHTTP(w, r)
    })
}
```

> ⚠️ **缓存边界问题：**无论文件是否存在（即使返回 404），都会先写入 `Cache-Control` header。

代码位置：[utils.go:fileServerWithCache()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/utils.go#L149-L158)

#### (2) bundle.css 特殊路径（覆盖序号 10）

`GET /static/{hash}/css/bundle.css` 精确匹配，优先级高于前缀路径，直接返回内存中的 `bundledCSSContents`：

```go
mux.HandleFunc("GET /static/{hash}/css/bundle.css", func(w http.ResponseWriter, r *http.Request) {
    w.Header().Add("Cache-Control", "public, max-age=86400")
    w.Header().Add("Content-Type", "text/css; charset=utf-8")
    w.Write(bundledCSSContents)
})
```

`bundledCSSContents` 是包级 var，启动期运行时 CSS 打包：递归解析 `@import`、去单行注释、去行首空白、去换行（约 20% 体积减少）。热重载不重新计算。

代码位置：[embed.go:bundledCSSContents](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/embed.go#L87-L151)

#### (3) PWA Manifest `/manifest.json`（24h 缓存 + 查询参数版本化）

```go
mux.HandleFunc("GET /manifest.json", func(w http.ResponseWriter, r *http.Request) {
    w.Header().Add("Cache-Control", "public, max-age=86400")
    w.Header().Add("Content-Type", "application/json")
    w.Write(a.parsedManifest)
})
```

- `a.parsedManifest`：每次 `newApplication()` 时用 `executeTemplateToString(manifestTemplate, templateData{App: app})` 重新渲染并存为 `[]byte`（热重载会重新渲染）。模板变量依赖 `AppName`、`AppBackgroundColor`、`AppIconURL` 等配置项，只有这些配置变化时 manifest 内容才实际变化。
- 模板中通过 `VersionedAssetPath("manifest.json")` 生成 URL：`/manifest.json?v=<CreatedAt.Unix()>`。

> ServeMux 忽略查询字符串，`/manifest.json?v=1717900000` 仍命中 `GET /manifest.json`。`?v=` 参数用于 cache busting：由于 `a.CreatedAt = time.Now()` 在每次 `newApplication()` 时赋值，**每次热重载 `?v=` 都会变化**，即使 manifest 内容未变浏览器也会被强制重新请求。

关键代码：
- `CreatedAt` 赋值：[newApplication()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L47-L54)
- manifest 渲染：[newApplication()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L224-L228)
- `VersionedAssetPath` URL 生成：[glance.go:431-434](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L431-L434)

#### (4) 用户自定义资源 `/assets/...`（2h 缓存，不限 HTTP 方法）

```go
mux.Handle(
    "/assets/{path...}",          // ← 不限 HTTP 方法，所有方法均可访问
    http.StripPrefix(
        "/assets/",
        fileServerWithCache(http.Dir(AssetsPath), 2*time.Hour),
    ),
)
```

- 读本地文件系统 `http.Dir`。
- 配置中 `/assets/xxx` 形式的 URL 会通过 [resolveUserDefinedAssetPath()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L272-L278) 自动拼接 `BaseURL` 前缀。

#### (5) 用户自定义 CSS 文件（模板层版本化 + resolveUserDefinedAssetPath）

`custom-css-file` 在 `newApplication()` 中先经过 [resolveUserDefinedAssetPath()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L272-L278) 处理：若路径以 `/assets/` 开头则自动拼接 `BaseURL` 前缀；外部 URL（`http(s)://`）和其他路径原样保留。

然后通过 [document.html:27](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/templates/document.html#L27) 引用：

```html
<link rel="stylesheet" href="{{ .App.Config.Theme.CustomCSSFile }}?v={{ .App.CreatedAt.Unix }}">
```

- `?v=` 时间戳 = `a.CreatedAt.Unix()`，**每次热重载都会变化**（与 CSS 文件本身是否修改无关）。
- 如果 CSS 部署在 `/assets/...` 下，实际 HTTP 请求会经过 Glance 的 `/assets/{path...}` 路由，命中 2h `Cache-Control`；`?v=` 参数用于在热重载时绕过浏览器缓存强制刷新。
- 如果 CSS 是外部 URL，则由外部服务器负责缓存策略，Glance 仅追加查询参数。

关键代码：
- resolveUserDefinedAssetPath 处理：[newApplication()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L197)
- 模板引用：[document.html:27](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/templates/document.html#L27)

#### 静态资源 URL 生成函数对比

| 函数 | URL 形式 | 用途 | URL 变化时机 |
|:---|:---|:---|:---|
| [StaticAssetPath(asset)](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L427-L429) | `/static/{hash}/{asset}` | 内嵌 JS / CSS / 字体 / 图标 | 进程重启（`staticFSHash` 是包级 var，热重载不变） |
| [VersionedAssetPath(asset)](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L431-L434) | `{asset}?v={CreatedAt}` | manifest.json 等 | 每次热重载（`a.CreatedAt = time.Now()` 在 newApplication() 中赋值） |
| `Theme.CustomCSSFile`（模板层） | `{CustomCSSFile}?v={CreatedAt}` | 用户自定义 CSS | 每次热重载（同上 `CreatedAt` 变化） |

---

### 资源版本号变化时机汇总（热重载 vs 重启）

| 资源 | 版本号来源 | 热重载时 URL 变化？ | 进程重启时 URL 变化？ | 内容实际变化？ |
|:---|:---|:---:|:---:|:---|
| `/static/{hash}/...`（内嵌） | `staticFSHash`（包级 var，MD5 内嵌 FS） | ❌ | ✅ | 仅代码修改重新编译后变化；热重载时内嵌 FS 不变 |
| `/static/{hash}/css/bundle.css` | `bundledCSSContents`（包级 var） | ❌ | ✅ | 同上 |
| `/manifest.json?v=xxx` | `a.CreatedAt.Unix()`（每次 newApplication 赋值） | ✅ | ✅ | 仅当 Branding 配置（AppName/BackgroundColor/AppIconURL）变化时内容变 |
| `custom-css-file?v=xxx` | `a.CreatedAt.Unix()` | ✅ | ✅ | 与 CSS 文件是否修改**无关**；URL 变强制浏览器重取 |
| `Branding.LogoURL`（用户自定义 `/assets/...`） | 无版本化，依赖 `/assets/` 的 2h Cache-Control | ❌ | ❌ | 文件系统实时读；浏览器缓存 2h；热重载后需手动清缓存或等 2h 过期 |
| `Branding.FaviconURL`（用户自定义 `/assets/...`） | 同上 | ❌ | ❌ | 同上 |
| `Branding.AppIconURL`（用户自定义 `/assets/...`） | 同上 | ❌ | ❌ | 同上 |
| `/assets/...` 通用 | 同上 | ❌ | ❌ | 同上 |

> **版本化策略不一致说明：**
> - 内嵌静态资源（`/static/{hash}/`）通过内容 hash 做 URL 指纹，热重载不变
> - manifest.json 和 custom-css-file 通过 `?v=<CreatedAt>` 做 cache busting，每次热重载 URL 都变（"过度失效"）
> - LogoURL / FaviconURL / AppIconURL 如果用户自定义为 `/assets/...`，则**完全没有 URL 版本化**，仅依赖 `/assets/` 的 2h Cache-Control — 修改这些资源后用户可能需要等 2 小时或手动清缓存才能看到新内容
> - 模板引用位置：LogoURL [page.html:22](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/templates/page.html#L22)、AppIconURL [document.html:22](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/templates/document.html#L22) + [manifest.json:10](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/templates/manifest.json#L10)、FaviconURL [document.html:24](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/templates/document.html#L24) — 均无 `?v=` 参数
>
> **⚠️ BaseURL 处理差异（用户自定义 `/assets/...` 场景）：**
> - **已处理**（自动加 BaseURL 前缀）：`CustomCSSFile` [glance.go:197](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L197)、`LogoURL` [glance.go:198](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L198)、`FaviconURL` [glance.go:200-204](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L200-L204)
> - **未处理**（用户自定义时原样保留，不加 BaseURL）：`AppIconURL` [glance.go:216-218](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L216-L218)
> - 子路径部署时，用户自定义 `app-icon-url: /assets/my-icon.png` 会被浏览器解析为站点根路径 `/assets/my-icon.png`，绕过反向代理的前缀剥离，导致 404。

---

### 4.4 响应压缩

**结论：应用层没有任何 gzip / brotli / deflate 压缩中间件。**

`go.mod` 中的 `klauspost/compress` 和 `andybalholm/brotli` 是 `golang.org/x/net` → `gofeed` → `goquery` 链的间接依赖，HTTP 响应路径未调用任何压缩 API。

生产环境应在反向代理层（nginx / traefik / caddy）开启压缩。

---

### 4.5 BaseURL / 前缀处理

`Server.BaseURL` 在 [newApplication()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L196) 中被 `TrimRight("/")`。

**不参与路由匹配**（路由永远注册在根 `/`），仅用于：

| 场景 | 代码位置 |
|:---|:---|
| 静态资源 URL 拼接 | [StaticAssetPath()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L427-L429)、[VersionedAssetPath()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L431-L434) |
| 登录/登出 303 重定向 | [auth.go:296](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L296)、[auth.go:308](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L308)、[auth.go:325](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L325) |
| Cookie `Path` | [auth.go:317](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L317)、[theme.go:31](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/theme.go#L31) |
| 用户资产 `/assets/` 前缀 | [resolveUserDefinedAssetPath()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L272-L278) |

子路径部署要求：反向代理层剥离前缀 `https://domain/glance/xxx` → `http://backend/xxx`，同时配置 `base-url: /glance`。

#### 用户自定义 Branding 资源的 BaseURL 处理差异（字段级）

`resolveUserDefinedAssetPath(path)` 逻辑：若 `path` 以 `/assets/` 开头则返回 `BaseURL + path`；否则原样返回（外部 URL 如 `https://...` 或其他相对路径）。

但四个 Branding/Theme 字段在 `newApplication()` 中的处理并不一致：

| 配置字段 | 空时默认值 | 用户自定义时是否调用 resolveUserDefinedAssetPath？ | 代码位置 |
|:---|:---|:---:|:---|
| `theme.custom-css-file` | 空 → 模板中不渲染 `<link>` 标签 | ✅ 是 | [glance.go:197](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L197) |
| `branding.logo-url` | 空 → 降级用 `logo-text`，再降级用内置 SVG | ✅ 是 | [glance.go:198](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L198) |
| `branding.favicon-url` | 空 → `StaticAssetPath("favicon.svg")`（自带 BaseURL） | ✅ 是 | [glance.go:200-204](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L200-L204) |
| `branding.app-icon-url` | 空 → `StaticAssetPath("app-icon.png")`（自带 BaseURL） | ❌ **否** | [glance.go:216-218](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L216-L218) |

> **⚠️ `app-icon-url` 子路径部署陷阱：**
> 用户在 YAML 中写 `app-icon-url: /assets/my-icon.png` 时，浏览器会把它解析为站点根路径 `/assets/my-icon.png`，反向代理的前缀剥离（例如从 `/glance/` 剥离）无法生效，最终请求到 `https://domain/assets/my-icon.png` 而非 `https://domain/glance/assets/my-icon.png`，导致 404。
>
> 临时绕过方案：用户在配置中手动写全路径 `app-icon-url: /glance/assets/my-icon.png`，或使用外部绝对 URL。
>
> `app-icon-url` 同时出现在 HTML head 的 `<link rel="apple-touch-icon">` [document.html:22](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/templates/document.html#L22) 和 PWA manifest 的 `icons[].src` [manifest.json:10](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/templates/manifest.json#L10)，两处都会受此影响。

---

### 4.6 鉴权检查

非全局中间件，每个需要保护的 Handler 入口显式调用：

```go
if a.handleUnauthorizedResponse(w, r, redirectToLogin) {
    return
}
```

| Handler | 未授权行为 | 代码位置 |
|:---|:---|:---|
| handlePageRequest | 303 → `BaseURL/login` | [glance.go:313-L315](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L313-L315) |
| handlePageContentRequest | 401 + JSON `{"error": "Unauthorized"}` | [glance.go:341-L343](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go#L341-L343) |

**匿名可访问路由**：所有静态资源（`/static/*`、`/assets/*`、`/manifest.json`）、`/api/healthz`、`/login`、`/api/authenticate`。

[isAuthorized()](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go#L250-L286) 内部流程：
1. `RequiresAuth == false` 直接放行
2. 读取 Cookie `session_token` → base64 解码 → HMAC-SHA256 验签
3. 检查 14 天有效期；剩余不足 7 天时自动重新签发新 Cookie

---

## 五、请求流时序图（以 `GET /{page}` 为例）

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
  │         ├─ verifySessionToken (HMAC 验签 + 过期检查)
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
Response（无应用层压缩；由反向代理层负责压缩）
```

---

## 六、关键文件索引

| 文件 | 作用 |
|:---|:---|
| [main.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/main.go) | 二进制入口 |
| [internal/glance/main.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/main.go) | CLI 分发、serveApp（三条启动路径、热重载停启）、v0.7 临时 Server |
| [internal/glance/glance.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/glance.go) | application 结构、路由注册、页面 Handler、X-Forwarded-For 解析、Static/VersionedAssetPath |
| [internal/glance/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/auth.go) | 登录/登出、会话 Token、Cookie（X-Forwarded-Proto）、鉴权检查 |
| [internal/glance/theme.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/theme.go) | 主题切换 Cookie、主题 CSS 输出 |
| [internal/glance/utils.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/utils.go) | fileServerWithCache 装饰器（404 也写 Cache-Control） |
| [internal/glance/embed.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/embed.go) | 内嵌 FS、staticFSHash（包级 var）、CSS 运行时打包（包级 var） |
| [internal/glance/templates.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/templates.go) | mustParseTemplate（启动期 panic fail-fast） |
| [internal/glance/widget.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/widget.go) | widget 模板渲染的错误恢复（二次渲染） |
| [internal/glance/config.go](file:///d:/fz/0601/solo-dogfeeding/code/145-glance/internal/glance/config.go) | config 结构、configFilesWatcher（末尾同步触发 onChange）、500ms debounce |
