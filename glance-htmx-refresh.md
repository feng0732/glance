# Glance 增量刷新机制详解（从请求到片段替换的边界分析）

Glance 项目并未直接使用 HTMX 库，而是实现了一套**自定义的类 HTMX 增量刷新机制**。本文档从四个维度对照代码剖析其完整流程：触发方式、片段路由、swap 策略、缓存头处理。

---

## 一、触发方式（Trigger）

### 1.1 页面级增量刷新：页面加载后的内容拉取

**入口点**：[page.js](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/page.js#L746-L784)

页面初次加载时，`page.html` 模板中的 `<main id="page">` 只渲染一个空的占位容器 `#page-content` 和加载动画：

```html
<!-- page.html L107-L114 -->
<main class="page" id="page" aria-live="polite" aria-busy="true">
    <h1 class="visually-hidden">{{ .Page.Title }}</h1>
    <div class="page-content" id="page-content"></div>
    <div class="page-loading-container">
        <div class="visually-hidden">Loading</div>
        <div class="loading-icon" aria-hidden="true"></div>
    </div>
</main>
```

随后 `page.js` 中的 `setupPage()` 自动触发：

```javascript
// page.js L746-L784
async function setupPage() {
    initThemePicker();
    const pageElement = document.getElementById("page");
    const pageContentElement = document.getElementById("page-content");
    const pageContent = await fetchPageContent(pageData);  // 触发增量请求

    pageContentElement.innerHTML = pageContent;  // 片段替换

    // 替换完成后重新绑定所有交互
    setupPopovers();
    setupClocks();
    await setupCalendars();
    await setupTodos();
    // ... 更多 widget 初始化
}
```

**关键数据注入**：`document.html` 模板在 `<head>` 中内联注入 `pageData` 全局变量，为前端提供路由参数：

```html
<!-- document.html L5-L12 -->
<script>
    const pageData = {
        slug: "{{ .Page.Slug }}",
        baseURL: "{{ .App.Config.Server.BaseURL }}",
        theme: "{{ .Request.Theme.Key }}",
    };
</script>
```

### 1.2 Widget 级触发：DOM 替换后渐进激活

部分 Widget（Calendar、Todo）在服务端只渲染占位骨架，前端 JS 加载后通过 `swapWith` 替换为完整功能组件：

- **Calendar**：[calendar.js](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/calendar.js#L32-L36)
- **Todo**：[todo.js](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/todo.js#L9-L13)

```javascript
// calendar.js L32-L36
export default function(element) {
    element.swapWith(Calendar(
        Number(element.dataset.firstDayOfWeek ?? 1)
    ));
}
```

### 1.3 交互触发：主题切换

[page.js L670-L692](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/page.js#L670-L692) 中的 `changeTheme()` 是典型的用户交互触发增量请求：

```javascript
async function changeTheme(key, onChanged) {
    const response = await fetch(`${pageData.baseURL}/api/set-theme/${key}`, {
        method: "POST",
    });
    const newThemeStyle = await response.text();
    themeStyleElem.html(newThemeStyle);  // 局部替换 <style> 内容
    document.documentElement.setAttribute("data-scheme", response.headers.get("X-Scheme"));
}
```

---

## 二、片段路由（Fragment Routing）

服务端通过 Go 标准库 `http.ServeMux` 注册了分层的路由体系，每个路由返回不同粒度的 HTML 片段。

### 2.1 路由注册总览

[glance.go L436-L516](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/glance.go#L436-L516)

```go
func (a *application) server() (func() error, func() error) {
    mux := http.NewServeMux()

    // 完整页面（包含 document.html 外壳）
    mux.HandleFunc("GET /{$}", a.handlePageRequest)
    mux.HandleFunc("GET /{page}", a.handlePageRequest)

    // ★ 增量刷新核心路由：仅返回页面内容片段
    mux.HandleFunc("GET /api/pages/{page}/content/{$}", a.handlePageContentRequest)

    // 主题切换：返回 CSS 片段
    mux.HandleFunc("POST /api/set-theme/{key}", a.handleThemeChangeRequest)

    // Widget 级路由（目前未实现）
    mux.HandleFunc("/api/widgets/{widget}/{path...}", a.handleWidgetRequest)

    // ... 静态资源、认证等路由
}
```

### 2.2 完整页面路由 vs 片段路由的边界

| 路由 | Handler | 模板 | 返回内容 |
|------|---------|------|----------|
| `GET /{page}` | `handlePageRequest` | `page.html` + `document.html` | 完整 HTML 文档（含 `<head>`、导航、外壳） |
| `GET /api/pages/{page}/content/` | `handlePageContentRequest` | `page-content.html` | 仅 widget 渲染结果（无外壳） |

#### 未登录时的分支响应差异

两个路由使用不同的未授权 fallback 策略，这是增量刷新机制的关键边界：

- **整页请求** `handlePageRequest` 使用 `redirectToLogin`：浏览器级 303 重定向到登录页
- **内容请求** `handlePageContentRequest` 使用 `showUnauthorizedJSON`：返回 401 状态码 + JSON 错误体

**代码对照**：

```go
// glance.go L313 — 整页请求未授权 → 303 重定向
if a.handleUnauthorizedResponse(w, r, redirectToLogin) {
    return
}

// glance.go L341 — 内容请求未授权 → 401 JSON（前端 fetch 目前未处理此分支）
if a.handleUnauthorizedResponse(w, r, showUnauthorizedJSON) {
    return
}
```

**fallback 实现**：[auth.go L289-L303](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/auth.go#L289-L303)

```go
func (a *application) handleUnauthorizedResponse(w http.ResponseWriter, r *http.Request, fallback doWhenUnauthorized) bool {
    if a.isAuthorized(w, r) {
        return false
    }
    switch fallback {
    case redirectToLogin:
        http.Redirect(w, r, a.Config.Server.BaseURL+"/login", http.StatusSeeOther)
    case showUnauthorizedJSON:
        w.WriteHeader(http.StatusUnauthorized)
        w.Write([]byte(`{"error": "Unauthorized"}`))
    }
    return true
}
```

**前端现状**：[page.js L6-L13](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/page.js#L6-L13) 中 `fetchPageContent` 有明确 TODO 注释说明未处理非 200 状态码，所以未登录时内容请求会走到请求失败分支（加载态永远停留）。

**关键差异代码**：

```go
// glance.go L306-L332 — 完整页面
func (a *application) handlePageRequest(w http.ResponseWriter, r *http.Request) {
    data := templateData{Page: page, App: a}
    a.populateTemplateRequestData(&data.Request, r)
    pageTemplate.Execute(&responseBytes, data)  // pageTemplate = page.html + document.html + footer.html
    w.Write(responseBytes.Bytes())
}

// glance.go L334-L367 — 增量片段
func (a *application) handlePageContentRequest(w http.ResponseWriter, r *http.Request) {
    pageData := templateData{Page: page}
    page.mu.Lock()
    defer page.mu.Unlock()
    page.updateOutdatedWidgets()  // ★ 路由边界前：先刷新过期 widget 数据
    pageContentTemplate.Execute(&responseBytes, pageData)  // pageContentTemplate = page-content.html
    w.Write(responseBytes.Bytes())
}
```

### 2.3 片段模板的结构

[page-content.html](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/templates/page-content.html) 仅包含两部分：

```html
{{ if .Page.HeadWidgets }}
<div class="head-widgets">
    {{- range .Page.HeadWidgets }}
    {{- .Render }}   <!-- 调用每个 widget 的 Render() 方法 -->
    {{- end }}
</div>
{{ end }}

<div class="page-columns">
{{- range .Page.Columns }}
    <div class="page-column page-column-{{ .Size }}">
        {{- range .Widgets }}
        {{- .Render }}
        {{- end }}
    </div>
{{- end }}
</div>
```

### 2.4 移动端页头返回边界处理

移动端页头是 `page-content.html` 中条件渲染的第一块内容，其显示-隐藏受三重边界控制：

**第一边界：服务端模板条件渲染**

[page-content.html L1-L3](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/templates/page-content.html#L1-L3)

```html
{{ if .Page.ShowMobileHeader }}
<div class="mobile-reachability-header">{{ .Page.Title }}</div>
{{ end }}
```

配置项来源：[config.go L82](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/config.go#L82) — `ShowMobileHeader bool`

**第二边界：CSS 媒体查询边界**

[site.css L296-L298](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/css/site.css#L296-L298) — 桌面端默认隐藏：

```css
.mobile-navigation, .mobile-reachability-header {
    display: none;
}
```

[mobile.css L218-L225](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/css/mobile.css#L218-L225) — `@media (max-width: 550px)` 下启用：

```css
@media (max-width: 550px) {
    .mobile-reachability-header {
        display: block;
        /* ... 10vh 垂直内边距，居中显示页面标题 */
        animation: pageColumnsEntrance .3s cubic-bezier(0.25, 1, 0.5, 1) backwards;
    }
}
```

**第三边界：content-ready 激活边界**

虽然模板渲染了 DOM 内，但其父容器 `#page-content` 默认 `display: none`（见 [site.css L10-L12](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/css/site.css#L10-L12)），只有当 `#page` 加上 `.content-ready` 类后才显示：

```css
.page.content-ready > .page-content {
    display: block;
}
```

因此移动端页头的完整显示链路是：**配置开启 → 视口宽度 ≤ 550px → 增量内容请求成功 → `content-ready` 类被添加。

**补充：移动端底部导航栏的返回占位

为避免内容被底部导航栏遮挡，`page.html L118 渲染了 `.mobile-navigation-offset` 占位元素，高度与导航栏高度精确匹配：

- 普通移动端：[mobile.css L31-L34](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/css/mobile.css#L31-L34) — `height: var(--mobile-navigation-height)
- 全屏模式（PWA）：[mobile.css L184-L186](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/css/mobile.css#L184-L186) — 额外加上 `--safe-area-inset-bottom

---

### 2.5 Widget 级路由（预留接口）

[glance.go L404-L425](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/glance.go#L404-L425) 中 `handleWidgetRequest` 目前返回 `501 Not Implemented`，但 `widget` 接口已定义了 `handleRequest` 方法，为未来单 widget 增量刷新预留了扩展点。

---

## 三、Swap 策略（内容替换策略）

Glance 实现了三级 swap 策略，对应不同粒度的内容更新。

### 3.1 Level 1: 整页内容替换（innerHTML）

最粗粒度，用于初始页面加载后的内容填充。

**代码位置**：[page.js L751-L753](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/page.js#L751-L753)

```javascript
const pageContent = await fetchPageContent(pageData);
pageContentElement.innerHTML = pageContent;
```

**边界**：
- 替换目标：`#page-content` 容器
- 替换前：显示 loading 动画，`aria-busy="true"`
- 替换后：执行 `contentReadyCallbacks` 队列，重新绑定所有 widget 的 JS 交互
- 副作用：所有 DOM 节点被销毁重建，需要 `setupPopovers/setupClocks/setupTodos` 等重新初始化

#### 请求失败时加载态永远停留的边界

**触发失败的场景**：

1. `fetchPageContent` 有明确 TODO 注释说明未处理异常情况 [page.js L6-L13](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/page.js#L6-L13)：

```javascript
async function fetchPageContent(pageData) {
    // TODO: handle non 200 status codes/time outs
    // TODO: add retries
    const response = await fetch(`${pageData.baseURL}/api/pages/${pageData.slug}/content/`);
    const content = await response.text();
    return content;
}
```

2. `setupPage` 中 `try/finally` 的位置关键：`fetchPageContent` 和 `innerHTML` **在 try 块之外**，只有 widget 初始化在 try 内 [page.js L746-L784](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/page.js#L746-L784)：

```javascript
async function setupPage() {
    const pageContent = await fetchPageContent(pageData);  // ← try 外，失败则崩溃
    pageContentElement.innerHTML = pageContent;            // ← try 外，失败则崩溃
    try {
        setupPopovers();  // ← try 内，单个 widget 初始化失败不影响 finally
        // ...
    } finally {
        pageElement.classList.add("content-ready");   // ← 只有成功到达这里才隐藏 loading
        pageElement.setAttribute("aria-busy", "false");
    }
}
```

**失败后的 CSS 停留机制**：

[site.css L10-L17](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/css/site.css#L10-L17)

```css
.page-content, .page.content-ready .page-loading-container {
    display: none;      /* 默认：内容隐藏；.content-ready 时 loading 隐藏 */
}

.page.content-ready > .page-content {
    display: block;     /* .content-ready 时：内容显示 */
}
```

逻辑真值表：

| 状态 | `.content-ready` | `#page-content` 显示 | `.page-loading-container` 显示 |
|------|------------------|----------------------|--------------------------------|
| 初始加载 | ❌ | ❌ | ✅（loading 转圈） |
| 请求成功 | ✅ | ✅ | ❌ |
| **请求失败/未登录** | **❌** | **❌** | **✅（永远转圈）** |

**加载动画的精确位置**：

- [site.css L157-L169](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/css/site.css#L157-L169) — 容器 100% 高度，`display: flex; align-items: center; justify-content: center`，视觉上垂直水平居中
- loading 图标本身额外 `translate: 0 -250%`，即**在容器中心基础上再向上偏移 2.5 倍自身高度**
- `.loading-icon` 为 `800ms infinite linear` 的旋转边框动画

### 3.2 Level 2: Widget 骨架替换（swapWith）

中等粒度，Calendar、Todo 等复杂交互 Widget 使用服务端渲染的占位元素 + 客户端完整替换。

**代码位置**：[templating.js L122-L125](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/templating.js#L122-L125)

```javascript
ep.swapWith = function(element) {
    this.replaceWith(element);
    return element;
}
```

**典型应用**：

```javascript
// calendar.js L32-L36
export default function(element) {
    element.swapWith(Calendar(Number(element.dataset.firstDayOfWeek ?? 1)));
}
```

**边界**：
- 替换目标：服务端渲染的 `<div class="calendar">` / `<div class="todo">` 骨架
- 替换物：客户端用 `elem()` 构建的完整 DOM 树（含事件绑定、动画、状态管理）
- 优势：避免 innerHTML 后丢失事件监听，替换即激活

### 3.3 Level 3: 细粒度组件内更新（animateUpdate / 直接操作）

最细粒度，用于组件内部状态变化时的局部更新。

**代码位置**：
- [templating.js L182-L189](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/templating.js#L182-L189) — `animateUpdate`
- [animations.js](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/animations.js) — 动画定义

```javascript
// templating.js L182-L189
ep.animateUpdate = function(update, exit, entrance) {
    this.animate(exit, () => {
        update(this);       // 1. 执行退出动画
        this.animate(entrance);  // 2. 更新内容  3. 执行进入动画
    });
    return this;
}
```

**Calendar 月份切换示例**：[calendar.js L168-L180](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/calendar.js#L168-L180)

```javascript
const update = function(now, newDate) {
    if (datesWithinSameMonth(newDate, lastRenderedDate)) {
        updateFullMonth(now, newDate);  // 同月：直接更新文本，无动画
        return;
    }
    // 跨月：先左滑退出，更新内容，再右滑进入
    dates.animateUpdate(
        () => updateFullMonth(now, newDate),
        next ? datesExitLeft : datesExitRight,
        next ? datesEntranceRight : datesEntranceLeft,
    );
}
```

### 3.4 Swap 策略层级总结

| 层级 | 方法 | 适用场景 | DOM 影响 | 动画支持 |
|------|------|----------|----------|----------|
| L1 | `innerHTML` | 整页内容初始加载 | 全量重建 | Loading 指示器 |
| L2 | `swapWith` (replaceWith) | Widget 骨架→完整组件 | 单 Widget 重建 | 无（首次渲染） |
| L3 | `animateUpdate` + 直接操作 | 组件内状态变化 | 局部文本/属性更新 | slideFade 等自定义动画 |

---

## 四、缓存头处理（Cache Headers）

### 4.1 静态资源缓存：长期缓存 + 内容哈希

**代码位置**：
- [glance.go L26](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/glance.go#L26) — 缓存时长常量
- [glance.go L459-L482](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/glance.go#L459-L482) — 静态资源路由
- [utils.go L149-L158](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/utils.go#L149-L158) — `fileServerWithCache` 包装器

```go
const STATIC_ASSETS_CACHE_DURATION = 24 * time.Hour

func fileServerWithCache(fs http.FileSystem, cacheDuration time.Duration) http.Handler {
    cacheControlValue := fmt.Sprintf("public, max-age=%d", int(cacheDuration.Seconds()))
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Cache-Control", cacheControlValue)
        server.ServeHTTP(w, r)
    })
}
```

**URL 缓存爆破机制**：静态资源路径中嵌入 `staticFSHash`（构建时生成的内容哈希）：

```go
// glance.go L427-L429
func (a *application) StaticAssetPath(asset string) string {
    return a.Config.Server.BaseURL + "/static/" + staticFSHash + "/" + asset
}
```

最终 URL 形如：`/static/abc123def456/js/page.js`，内容变化时哈希变化，天然避免缓存污染。

### 4.2 动态内容：无缓存头 → 每次请求必达服务端

**增量刷新核心路由** `handlePageContentRequest` 和 **完整页面路由** `handlePageRequest` 均**未设置任何 HTTP 缓存相关响应头**，具体缺失清单：

| 响应头 | 状态 | 影响 |
|--------|------|------|
| `Cache-Control` | ❌ 未设置 | 浏览器无法获知缓存策略 |
| `Expires` | ❌ 未设置 | 无明确过期时间 |
| `ETag` | ❌ 未设置 | 无法做 `If-None-Match` 条件请求 |
| `Last-Modified` | ❌ 未设置 | 无法做 `If-Modified-Since` 条件请求 |
| `Pragma` | ❌ 未设置 | 无 HTTP/1.0 兼容缓存指令 |

**请求必达服务端的多层依据**：

1. **响应侧缺失**：Go `http.ResponseWriter` 默认不写任何缓存头，`w.Write()` 输出的响应只有 `Date`、`Content-Type`、`Content-Length` 等基础头
2. **请求侧 fetch 默认行为**：`fetch()` API 的 `cache` 参数默认为 `'default'`，对于没有缓存头的响应，浏览器不会存入 HTTP 缓存（RFC 7234 规定无缓存指令的响应可做启发式缓存，但现代浏览器对 `/api/` 路径的 GET 请求通常采取保守策略即不缓存）
3. **页面生命周期强制刷新**：每次用户刷新页面或导航到新页面 → 重新加载 `page.js` → 重新执行 `setupPage()` → 必然发起新的 `fetchPageContent()` 调用
4. **无缓存键爆破机制**：与静态资源不同，动态 API URL 中没有版本哈希或时间戳参数，但由于上述三点，实际上每次请求都穿透到服务端

这意味着每次页面加载或刷新，`/api/pages/{page}/content/` 请求必然到达服务端，触发 `page.updateOutdatedWidgets()` 检查并按需更新 widget 数据。

### 4.3 Widget 级内存缓存（服务端内缓存，非 HTTP 缓存）

虽然没有 HTTP 缓存头，但 Glance 在服务端实现了 Widget 粒度的内存缓存体系：

**代码位置**：[widget.go L141-L367](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/widget.go#L141-L367)

```go
type cacheType int
const (
    cacheTypeInfinite cacheType = iota  // 永不更新（如静态书签）
    cacheTypeDuration                    // 定时更新（如天气 30 分钟）
    cacheTypeOnTheHour                   // 整点更新（如市场数据）
)

type widgetBase struct {
    cacheDuration      time.Duration
    cacheType          cacheType
    nextUpdate         time.Time    // 下次更新时间点
    updateRetriedTimes int          // 失败重试计数（指数退避）
    templateBuffer     bytes.Buffer // 渲染结果缓存
}
```

**更新判定逻辑**：[widget.go L173-L183](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/widget.go#L173-L183)

```go
func (w *widgetBase) requiresUpdate(now *time.Time) bool {
    if w.cacheType == cacheTypeInfinite {
        return false
    }
    if w.nextUpdate.IsZero() {
        return true  // 首次访问必须更新
    }
    return now.After(w.nextUpdate)
}
```

**失败重试的指数退避**：[widget.go L350-L367](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/widget.go#L350-L367)

```go
func (w *widgetBase) scheduleEarlyUpdate() *widgetBase {
    w.updateRetriedTimes++
    if w.updateRetriedTimes > 5 { w.updateRetriedTimes = 5 }
    // 重试间隔：1²=1min, 2²=4min, 3²=9min, 4²=16min, 5²=25min
    nextEarlyUpdate := time.Now().Add(time.Duration(math.Pow(float64(w.updateRetriedTimes), 2)) * time.Minute)
    // 取"提前重试时间"和"正常更新时间"中较早的一个
    w.nextUpdate = minTime(nextEarlyUpdate, w.getNextUpdateTime())
    return w
}
```

### 4.4 自定义响应头：X-Scheme（主题元数据）

主题切换接口除了返回 CSS 文本，还通过自定义 Header 传递配色方案信息，避免前端解析 CSS：

**服务端**：[theme.go L36-L38](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/theme.go#L36-L38)

```go
w.Header().Set("Content-Type", "text/css")
w.Header().Set("X-Scheme", ternary(properties.Light, "light", "dark"))
w.Write([]byte(properties.CSS))
```

**客户端**：[page.js L689](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/page.js#L689)

```javascript
document.documentElement.setAttribute("data-scheme", response.headers.get("X-Scheme"));
```

### 4.5 认证相关头处理

- `Retry-After`：登录限流时返回，告知客户端下次尝试时间 [auth.go L168-L169](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/auth.go#L168-L169)
- `Set-Cookie`：会话 Cookie（HttpOnly + SameSite=Lax）[auth.go L311-L321](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/auth.go#L311-L321)
- `Set-Cookie`：主题偏好 Cookie（2 年有效期）[theme.go L28-L34](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/theme.go#L28-L34)

---

## 五、完整请求-替换链路时序图

```
浏览器                           Glance 服务端
  │                                  │
  │ GET /home                        │
  │─────────────────────────────────▶│ handlePageRequest
  │  ┌─ 未登录分支 ─┐                │   → isAuthorized() = false
  │  │ 303 → /login │                │   → handleUnauthorizedResponse(redirectToLogin)
  │  └──────────────┘                │
  │                                  │   → 渲染 page.html + document.html
  │ 200 OK (完整 HTML)               │     (仅空壳 #page-content + loading)
  │◀─────────────────────────────────│
  │                                  │
  │ 执行 setupPage()                 │
  │   ├─ #page aria-busy="true"      │
  │   ├─ .page-loading-container 显示│
  │   └─ GET /api/pages/home/content/│
  │─────────────────────────────────▶│ handlePageContentRequest
  │  ┌─ 未登录分支 ─┐                │   → isAuthorized() = false
  │  │ 401 Unauthorized  │           │   → handleUnauthorizedResponse(showUnauthorizedJSON)
  │  │ {"error":"Unauthorized"}│     │
  │  │ ↓                          │  │
  │  │ fetchPageContent() 抛异常   │  │
  │  │ finally 永不执行           │  │
  │  │ loading 永远停留           │  │
  │  └────────────────────────────┘  │
  │                                  │   ├─ page.mu.Lock()
  │                                  │   ├─ page.updateOutdatedWidgets()
  │                                  │   │   └─ 并发更新所有过期 widget
  │                                  │   ├─ 渲染 page-content.html
  │                                  │   │   ├─ {{if ShowMobileHeader}} 移动端页头
  │                                  │   │   └─ 每个 widget.Render()
  │                                  │   └─ page.mu.Unlock()
  │ 200 OK (HTML 片段，无任何缓存头) │
  │◀─────────────────────────────────│
  │                                  │
  │ pageContentElement.innerHTML =   │
  │   pageContent                    │
  │ try {                            │
  │   setupCalendars()               │
  │     → .swapWith(Calendar())      │
  │   setupTodos()                   │
  │     → .swapWith(Todo())          │
  │   setupPopovers() ...            │
  │ } finally {                      │
  │   ├─ #page.content-ready ← 加上  │
  │   │   ├─ .page-content 显示      │
  │   │   ├─ .page-loading-container │
  │   │   │   隐藏                    │
  │   │   └─ ≤550px 时移动端页头显示 │
  │   ├─ aria-busy = "false"         │
  │   └─ 300ms 后 body +             │
  │      .page-columns-transitioned  │
  │ }                                │
  │                                  │
  │ 用户点击主题切换                  │
  │ POST /api/set-theme/dark         │
  │─────────────────────────────────▶│ handleThemeChangeRequest
  │                                  │   → Set-Cookie: theme=dark
  │                                  │   → X-Scheme: dark
  │ 200 OK (CSS 文本)                │
  │◀─────────────────────────────────│
  │ themeStyleElem.html(newCSS)      │
```

---

## 六、关键边界总结

| 边界点 | 位置 | 说明 |
|--------|------|------|
| **触发入口** | [page.js:setupPage()](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/page.js#L746-L784) | 页面加载后自动触发 fetch 增量请求 |
| **整页 vs 片段路由分界** | [glance.go:server()](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/glance.go#L436-L516) | `GET /{page}` 返回完整文档，`GET /api/pages/*/content/` 返回片段 |
| **未登录分支差异** | [auth.go:handleUnauthorizedResponse()](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/auth.go#L289-L303) | 整页请求 → 303 重定向登录页；内容请求 → 401 JSON（前端未处理） |
| **数据刷新边界** | [glance.go:handlePageContentRequest()](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/glance.go#L334-L367) | 请求到达后先 `updateOutdatedWidgets()` 再渲染模板 |
| **移动端页头三重边界** | 配置 `ShowMobileHeader` + [mobile.css @media ≤550px](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/css/mobile.css#L218-L225) + `.content-ready` 类 | 配置开启 + 窄视口 + 请求成功 三者同时满足才显示 |
| **DOM 替换边界** | [page.js L753](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/page.js#L753) | `innerHTML` 是 L1 替换，之后必须重新绑定所有交互 |
| **请求失败-loading 停留边界** | [page.js L746-L784](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/page.js#L746-L784) try/finally 位置 | fetch/innerHTML 在 try 外，失败则 finally 永不执行，loading 永远转圈 |
| **loading 精确位置** | [site.css L157-L169](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/css/site.css#L157-L169) | flex 居中基础上再 `translateY(-250%)`，视觉上偏上 |
| **Widget 激活边界** | [calendar.js L32](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/calendar.js#L32) / [todo.js L9](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/static/js/todo.js#L9) | `swapWith` 将服务端骨架替换为客户端完整组件 |
| **动态请求必达服务端依据** | Go `http.ResponseWriter` 默认行为 + fetch 默认 cache:'default' | 无 Cache-Control/ETag/Last-Modified 任何缓存头，每次穿透 |
| **HTTP 缓存边界** | 静态资源 vs 动态 API | 静态 24h + 内容哈希爆破；动态 API 无任何缓存头 |
| **服务端缓存边界** | [widget.go:requiresUpdate()](file:///d:/fz/0601/solo-dogfeeding/code/144-glance/internal/glance/widget.go#L173-L183) | Widget 粒度内存缓存，按 cacheType 判定是否需要拉取新数据 |
