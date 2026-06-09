# Glance 页面布局渲染流程分析

## 一、整体流程概览

从 YAML 配置到最终 HTML 输出的完整链路：

```
YAML 配置文件
    ↓ (config.go)
parseYAMLIncludes() → parseConfigVariables() → yaml.Unmarshal()
    ↓
config 结构体（含 Pages、Columns、Widgets）
    ↓ (glance.go: newApplication)
newApplication() - 初始化页面 slug、主列索引、widget provider
    ↓ (glance.go: handlePageRequest)
HTTP 请求 → 查找 page → 构造 templateData → pageTemplate.Execute()
    ↓ (template 渲染)
document.html → page.html → page-content.html → widget-base.html → 各 widget 模板
```

---

## 二、页面组装流程详解

### 2.1 配置数据结构

核心数据结构定义在 [config.go](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/config.go#L77-L92)：

```go
type page struct {
    Title                  string  `yaml:"name"`
    Slug                   string  `yaml:"slug"`
    Width                  string  `yaml:"width"`           // "wide" | "slim" | "default"
    DesktopNavigationWidth string  `yaml:"desktop-navigation-width"`
    ShowMobileHeader       bool    `yaml:"show-mobile-header"`
    HideDesktopNavigation  bool    `yaml:"hide-desktop-navigation"`
    CenterVertically       bool    `yaml:"center-vertically"`
    HeadWidgets            widgets `yaml:"head-widgets"`     // 顶部 widgets
    Columns                []struct {
        Size    string  `yaml:"size"`     // "small" | "full"
        Widgets widgets `yaml:"widgets"`
    } `yaml:"columns"`
    PrimaryColumnIndex int8       `yaml:"-"`  // 运行时计算：第一个 full 列的索引
}
```

### 2.2 应用初始化阶段（页面预处理）

在 [glance.go: newApplication](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/glance.go#L47-L231) 中完成以下预处理：

1. **Slug 生成**：若未配置 slug，通过 `titleToSlug()` 将标题转为 URL 友好格式
2. **宽度标准化**：`page.Width == "default"` → 置空字符串（使用 CSS 默认）
3. **导航宽度继承**：`DesktopNavigationWidth` 未设置时继承 `page.Width`
4. **主列索引计算**：遍历 columns，将第一个 `size: full` 的列索引存入 `PrimaryColumnIndex`（用于移动端默认选中）
5. **Widget 注册**：所有 head-widgets 和 columns 中的 widget 注册到 `app.widgetByID`，并注入 `assetResolver` provider

### 2.3 HTTP 请求处理

页面完整渲染入口：[glance.go: handlePageRequest](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/glance.go#L306-L332)

```go
func (a *application) handlePageRequest(w http.ResponseWriter, r *http.Request) {
    page, exists := a.slugToPage[r.PathValue("page")]
    // ... 鉴权检查
    data := templateData{Page: page, App: a}
    a.populateTemplateRequestData(&data.Request, r)  // 从 cookie 读取主题
    pageTemplate.Execute(&responseBytes, data)        // 渲染完整 HTML 文档
}
```

页面内容局部刷新（AJAX）入口：[glance.go: handlePageContentRequest](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/glance.go#L334-L367)

```go
func (a *application) handlePageContentRequest(w http.ResponseWriter, r *http.Request) {
    // ...
    page.mu.Lock()
    page.updateOutdatedWidgets()       // 并发更新需要刷新的 widget
    pageContentTemplate.Execute(...)   // 只渲染 page-content 部分
}
```

---

## 三、模板继承关系

### 3.1 模板文件依赖图

```
mustParseTemplate() 函数用于构建模板依赖树：

pageTemplate
├── page.html              (主模板)
│   ├── document.html      (基础 HTML 骨架)
│   │   └── footer.html    (页脚)
│   └── (通过 block 覆盖注入内容)

pageContentTemplate
└── page-content.html      (页面主体内容)

各 Widget 模板（以 clock 为例）
clockWidgetTemplate
├── clock.html             (widget 具体内容)
│   └── widget-base.html   (所有 widget 的通用外壳)
```

模板初始化代码在 [glance.go:20-24](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/glance.go#L20-L24)：
```go
var (
    pageTemplate        = mustParseTemplate("page.html", "document.html", "footer.html")
    pageContentTemplate = mustParseTemplate("page-content.html")
)
```

### 3.2 模板继承机制详解

#### document.html - 基础骨架

[document.html](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/templates/document.html) 定义了完整 HTML 文档结构，暴露多个可覆盖 block：

| Block 名称 | 作用 |
|-----------|------|
| `document-head-before` | `<head>` 起始位置注入 |
| `document-title` | `<title>` 内容 |
| `document-head-after` | `<head>` 末尾注入（CSS/JS） |
| `document-body` | `<body>` 全部内容 |

#### page.html - 页面主模板

[page.html](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/templates/page.html) 通过 `{{ template "document.html" . }}` 引入基础模板，并覆盖各 block：

```html
{{ template "document.html" . }}

{{ define "document-title" }}{{ .Page.Title }}{{ end }}

{{ define "document-head-after" }}
    <script type="module" src='{{ .App.StaticAssetPath "js/page.js" }}'></script>
{{ end }}

{{ define "navigation-links" }}
    <!-- 渲染页面导航链接 -->
{{ end }}

{{ define "document-body" }}
    <!-- 桌面端导航栏 + 移动端导航 + 内容容器 + 页脚 -->
    <div class="content-bounds grow{{ if .Page.Width }} content-bounds-{{ .Page.Width }}{{ end }}">
        <main class="page" id="page">
            <div class="page-content" id="page-content"></div>  <!-- JS 动态填充 -->
            <div class="page-loading-container">...</div>
        </main>
    </div>
{{ end }}
```

**关键点**：初始页面渲染时 `page-content` 区域为空，由前端 JS 请求 `/api/pages/{page}/content/` 获取 `pageContentTemplate` 的输出后填充。

#### page-content.html - 动态内容区

[page-content.html](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/templates/page-content.html) 负责实际的页面内容布局：

```html
{{ if .Page.ShowMobileHeader }}
    <div class="mobile-reachability-header">{{ .Page.Title }}</div>
{{ end }}

{{ if .Page.HeadWidgets }}
    <div class="head-widgets">
        {{- range .Page.HeadWidgets }}
        {{- .Render }}    <!-- 调用 widget 的 Render() 方法 -->
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

#### widget-base.html - Widget 通用外壳

[widget-base.html](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/templates/widget-base.html) 定义了所有 widget 的统一结构：

```html
<div class="widget widget-type-{{ .GetType }}{{ if .CSSClass }} {{ .CSSClass }}{{ end }}">
    {{- if not .HideHeader }}
    <div class="widget-header">
        <!-- 标题、WIP 标识、错误/通知图标 -->
    </div>
    {{- end }}
    <div class="widget-content{{ if .ContentAvailable }} {{ block "widget-content-classes" . }}{{ end }}{{ end }}">
        {{- if .ContentAvailable }}
        {{ block "widget-content" . }}{{ end }}    <!-- 子模板覆盖 -->
        {{- else }}
        <!-- 错误显示 -->
        {{- end}}
    </div>
</div>
```

Widget 模板通过暴露 `widget-content-classes` 和 `widget-content` 两个 block 实现扩展。

#### 具体 Widget 模板示例（split-column）

[split-column.html](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/templates/split-column.html)：

```html
{{ template "widget-base.html" . }}

{{ define "widget-content-classes" }}widget-content-frameless{{ end }}

{{ define "widget-content" }}
<div class="masonry" data-max-columns="{{ .MaxColumns }}">
{{ range .Widgets }}
    {{ .Render }}
{{ end }}
</div>
{{ end }}
```

---

## 四、列宽处理机制

### 4.1 页面级宽度

CSS 定义在 [site.css:226-239](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/static/css/site.css#L226-L239)：

| 类名 | max-width | 触发条件 |
|------|-----------|---------|
| `.content-bounds` | 1600px | 默认（page.Width 为空） |
| `.content-bounds-wide` | 1920px | `page.Width == "wide"` |
| `.content-bounds-slim` | 1100px | `page.Width == "slim"` |

模板中动态应用：`class="content-bounds grow{{ if .Page.Width }} content-bounds-{{ .Page.Width }}{{ end }}"`

### 4.2 列级宽度

CSS 定义在 [site.css:135-148](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/static/css/site.css#L135-L148)：

| 类名 | 宽度值 | 说明 |
|------|--------|------|
| `.page-column-small` | `width: 300px; flex-shrink: 0` | 固定 300px，不收缩 |
| `.page-column-full` | `width: 100%; min-width: 0` | 弹性填充剩余空间 |
| `.page-columns` | `display: flex; gap: var(--widget-gap)` | flex 容器，列间有间距 |

模板中动态应用：`<div class="page-column page-column-{{ .Size }}">`

### 4.3 列宽组合约束

在 [config.go: isConfigStateValid](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/config.go#L451-L537) 中校验：

- slim 页面最多 2 列，wide/default 页面最多 3 列
- 必须有 1 或 2 个 `size: full` 的列（不能全是 small，也不能超过 2 个 full）

典型布局示例：
- 2 列：`[full, small]` 或 `[small, full]`
- 3 列：`[small, full, small]` 或 `[full, full, small]`

### 4.4 Split-Column Widget 的瀑布流布局

[split-column.html](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/templates/split-column.html) 使用 CSS Masonry 布局：

```html
<div class="masonry" data-max-columns="{{ .MaxColumns }}">
```

JS 逻辑在 [masonry.js](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/static/js/masonry.js) 中实现，根据 `data-max-columns` 属性将子元素动态分配到多列中。

---

## 五、Widget 渲染流程

### 5.1 Widget 接口

[widget.go:126-139](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/widget.go#L126-L139) 定义了 widget 必须实现的接口：

```go
type widget interface {
    Render() template.HTML
    GetType() string
    GetID() uint64
    initialize() error
    requiresUpdate(*time.Time) bool
    setProviders(*widgetProviders)
    update(context.Context)
    setID(uint64)
    handleRequest(w http.ResponseWriter, r *http.Request)
    setHideHeader(bool)
}
```

### 5.2 Render 方法实现

以 [clock widget](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/widget-clock.go#L46-L48) 为例：

```go
func (widget *clockWidget) Render() template.HTML {
    return widget.cachedHTML  // initialize() 时已预渲染
}
```

以 [split-column widget](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/widget-split-column.go#L43-L45) 为例：

```go
func (widget *splitColumnWidget) Render() template.HTML {
    return widget.renderTemplate(widget, splitColumnWidgetTemplate)
}
```

基础渲染方法在 [widget.go:217-241](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/widget.go#L217-L241)：

```go
func (w *widgetBase) renderTemplate(data any, t *template.Template) template.HTML {
    w.templateBuffer.Reset()
    err := t.Execute(&w.templateBuffer, data)
    if err != nil {
        // 错误处理：如果渲染失败，尝试再次渲染以显示错误信息
        // 避免半渲染的 HTML 破坏整个页面结构
    }
    return template.HTML(w.templateBuffer.String())
}
```

---

## 六、移动端适配

### 6.1 移动端列切换

在 [page.html](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/templates/page.html#L57-L104) 的移动端导航中，使用 radio input + CSS `:has()` 选择器实现列切换：

```html
{{ range $i, $column := .Page.Columns }}
<label class="mobile-navigation-label">
    <input type="radio" class="mobile-navigation-input" 
           name="column" value="{{ $i }}"
           {{ if eq $i $.Page.PrimaryColumnIndex }} checked{{ end }}>
    <div class="mobile-navigation-pill"></div>
</label>
{{ end }}
```

CSS 在 [mobile.css:89-91](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/static/css/mobile.css#L89-L91)：
```css
body:has(.mobile-navigation-input[value="0"]:checked) .page-columns > :nth-child(1),
body:has(.mobile-navigation-input[value="1"]:checked) .page-columns > :nth-child(2),
body:has(.mobile-navigation-input[value="2"]:checked) .page-columns > :nth-child(3) {
    /* 显示对应列 */
}
```

---

## 七、页面壳与内容接口的异步协作机制

### 7.1 架构设计：壳（Shell）与内容分离

Glance 采用"首屏骨架 + 异步内容填充"的两段式渲染架构：

```
┌───────────────────────────────────────────────┐
│  HTTP GET /{page}                              │
│  → pageTemplate 渲染（document.html + page.html）│
│  → 返回包含空 page-content 的完整 HTML 骨架      │
│  → 浏览器加载 page.js                          │
└────────────────────┬──────────────────────────┘
                     │ page.js 执行 setupPage()
                     ▼
┌───────────────────────────────────────────────┐
│  HTTP GET /api/pages/{page}/content/            │
│  → handlePageContentRequest                    │
│  → 更新过期 widget → pageContentTemplate 渲染   │
│  → 返回纯 HTML 片段（head-widgets + columns）   │
└────────────────────┬──────────────────────────┘
                     │ 注入 DOM + 初始化交互组件
                     ▼
┌───────────────────────────────────────────────┐
│  页面可交互                                    │
└───────────────────────────────────────────────┘
```

### 7.2 服务端：双接口协作

#### 接口一：完整页面壳

[glance.go: handlePageRequest](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/glance.go#L306-L332) 负责输出页面骨架：

- **渲染模板**：`pageTemplate`（page.html + document.html + footer.html）
- **包含内容**：HTML doctype、head（CSS/主题/meta）、导航栏、`#page` 容器、空的 `#page-content`、loading 动画、footer
- **不做的事**：不更新 widget，不渲染任何具体 widget 内容
- **注入 JS 数据**：通过内联脚本将 `pageData`（slug、baseURL、当前主题）暴露给前端

模板中注入的数据（[page.html:5-12](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/templates/document.html#L5-L12)）：
```html
<script>
const pageData = {
    slug: "{{ .Page.Slug }}",
    baseURL: "{{ .App.Config.Server.BaseURL }}",
    theme: "{{ .Request.Theme.Key }}",
};
</script>
```

#### 接口二：页面内容片段

[glance.go: handlePageContentRequest](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/glance.go#L334-L367) 负责输出动态内容：

```go
func (a *application) handlePageContentRequest(w http.ResponseWriter, r *http.Request) {
    // ... 鉴权

    func() {
        page.mu.Lock()           // 加互斥锁，防止并发更新同一页面
        defer page.mu.Unlock()

        page.updateOutdatedWidgets()       // 第1步：并发刷新所有过期 widget
        err = pageContentTemplate.Execute(...)  // 第2步：渲染内容模板
    }()
    // ...
}
```

**关键特性**：
- **加锁执行**：`page.mu` 互斥锁确保 widget 更新和模板渲染的原子性，避免并发请求触发重复更新或数据竞争
- **惰性更新**：`updateOutdatedWidgets()` 只刷新 `requiresUpdate()` 返回 true 的 widget（基于缓存时长）
- **纯内容输出**：只返回 `page-content.html` 渲染的 HTML 片段，不含 html/head/body/导航等骨架

### 7.3 客户端：page.js 异步编排

[page.js: setupPage](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/static/js/page.js#L746-L786) 是前端主控函数：

```javascript
async function setupPage() {
    initThemePicker();                                // 先初始化主题选择器

    const pageElement = document.getElementById("page");
    const pageContentElement = document.getElementById("page-content");
    const pageContent = await fetchPageContent(pageData);  // 异步拉取内容

    pageContentElement.innerHTML = pageContent;            // 注入 DOM

    try {
        setupPopovers();              // 气泡弹出
        setupClocks();                // 时钟（实时更新）
        await setupCalendars();       // 日历组件
        await setupTodos();           // TODO 组件
        setupCarousels();             // 轮播
        setupSearchBoxes();           // 搜索框
        setupCollapsibleLists();      // 可折叠列表
        setupCollapsibleGrids();      // 可折叠网格
        setupGroups();                // group widget tab 切换
        setupMasonries();             // split-column 瀑布流布局 ← 关键！
        setupDynamicRelativeTime();   // 动态相对时间（"5m ago"）
        setupLazyImages();            // 懒加载图片淡入
    } finally {
        // 标记内容就绪
        pageElement.classList.add("content-ready");
        pageElement.setAttribute("aria-busy", "false");

        // 触发所有 afterContentReady 回调（如 Masonry、懒加载图片等注册的延迟初始化）
        for (let i = 0; i < contentReadyCallbacks.length; i++) {
            contentReadyCallbacks[i]();
        }

        setTimeout(() => setupTruncatedElementTitles(), 50);
        setTimeout(() => {
            document.body.classList.add("page-columns-transitioned");
        }, 300);   // 300ms 后启用列动画，避免首屏性能抖动
    }
}
```

### 7.4 协作时序与状态机

```
浏览器                                服务端
  │                                     │
  │── GET /{page} ─────────────────────▶│
  │                                     │── pageTemplate.Execute()
  │◀──────── HTML 骨架 + loading ───────│
  │                                     │
  │─ page.js 加载并执行 ─               │
  │   ├─ initThemePicker()              │
  │   └─ fetchPageContent() ──          │
  │                                     │
  │            ┌────────────────────────│
  │            │ GET /api/pages/{slug}/content/
  │            │                        │── page.mu.Lock()
  │            │                        │── updateOutdatedWidgets()
  │            │                        │── pageContentTemplate.Execute()
  │            │◀── HTML 片段 ──────────│
  │            │
  │─ pageContentElement.innerHTML = ...
  │─ setupMasonries() ← split-column 布局（依赖 DOM 尺寸）
  │─ ... 其他交互初始化
  │─ pageElement.classList.add("content-ready")
  │     ↓ 隐藏 loading，显示内容
```

### 7.5 afterContentReady 延迟回调机制

对于需要等内容 DOM 完全插入后才能计算（如依赖元素尺寸的布局），提供了回调注册：

```javascript
const contentReadyCallbacks = [];

function afterContentReady(callback) {
    contentReadyCallbacks.push(callback);
}
```

例如 `setupLazyImages()` 和 `setupCollapsibleGrids()` 都通过 `afterContentReady()` 延迟 ResizeObserver 的 attach，确保能正确读取元素尺寸。

---

## 八、Split-Column 的列分配策略详解

### 8.1 配置与模板

后端配置在 [widget-split-column.go](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/widget-split-column.go#L11-L15)：

```go
type splitColumnWidget struct {
    widgetBase          `yaml:",inline"`
    containerWidgetBase `yaml:",inline"`
    MaxColumns          int `yaml:"max-columns"`
}
```

模板输出在 [split-column.html](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/templates/split-column.html#L5-L10)：

```html
{{ define "widget-content" }}
<div class="masonry" data-max-columns="{{ .MaxColumns }}">
{{ range .Widgets }}
    {{ .Render }}      <!-- 所有子 widget 作为 masonry 的直接子元素顺序输出 -->
{{ end }}
</div>
{{ end }}
```

**注意**：后端只负责将子 widget 按配置顺序平铺渲染为 masonry 容器的直接子节点，**列分配完全由前端 JS 完成**。

### 8.2 Masonry 核心算法

[masonry.js](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/static/js/masonry.js) 实现的是一种**简化版轮询（Round-Robin）分配策略**，而非真正的最短列优先瀑布流：

```javascript
export function setupMasonries() {
    const masonryContainers = document.getElementsByClassName("masonry");

    for (let i = 0; i < masonryContainers.length; i++) {
        const container = masonryContainers[i];

        const options = {
            minColumnWidth: container.dataset.minColumnWidth || 330,  // 默认 330px
            maxColumns: container.dataset.maxColumns || 6,            // 默认 6 列
        };

        const items = Array.from(container.children);  // 缓存所有子元素（引用，不会被 DOM 移除影响）
        let previousColumnsCount = 0;

        const render = function() {
            // ── 步骤 1：根据容器宽度计算实际列数 ──
            const columnsCount = clamp(
                Math.floor(container.offsetWidth / options.minColumnWidth),
                1,
                Math.min(options.maxColumns, items.length)
            );
            // 列数 = clamp( floor(容器宽 / 最小列宽), 1, min(最大列数, 元素个数) )

            // ── 步骤 2：列数未变化则跳过（性能优化） ──
            if (columnsCount === previousColumnsCount) {
                return;
            } else {
                container.textContent = "";       // 清空所有内容
                previousColumnsCount = columnsCount;
            }

            // ── 步骤 3：创建 N 个列容器 ──
            const columnsFragment = document.createDocumentFragment();
            for (let i = 0; i < columnsCount; i++) {
                const column = document.createElement("div");
                column.className = "masonry-column";
                columnsFragment.append(column);
            }

            // ── 步骤 4：核心分配策略 ── Round-Robin 轮询
            for (let i = 0; i < items.length; i++) {
                columnsFragment.children[i % columnsCount].appendChild(items[i]);
            }
            // 第 0 个 → 第 0 列
            // 第 1 个 → 第 1 列
            // ...
            // 第 N 个 → 第 (N % columnsCount) 列
            // 第 N+1 个 → 回到第 0 列

            container.append(columnsFragment);
        };

        // ── 步骤 5：监听容器尺寸变化，重新布局 ──
        const observer = new ResizeObserver(() => requestAnimationFrame(render));
        observer.observe(container);
    }
}
```

### 8.3 分配策略特性分析

| 特性 | 说明 |
|------|------|
| **分配算法** | 纯 Round-Robin 轮询，`i % columnsCount` 按顺序循环分配 |
| **不考虑高度** | 不测量子元素高度，不做"最短列优先"。如果前几个元素特别高，会出现列高度不均 |
| **列数计算** | 基于容器宽度与 `minColumnWidth`（默认 330px）自动计算，上限为 `maxColumns` 和元素数量 |
| **列数变化检测** | `previousColumnsCount` 对比，只有列数真正变化才重建 DOM，避免无效重排 |
| **性能优化** | 使用 `DocumentFragment` 批量插入，`ResizeObserver` + `requestAnimationFrame` 确保在下一帧渲染 |
| **响应式** | 窗口 resize 导致容器宽度变化时自动重新计算列数并重新分配 |

### 8.4 列分配示例

假设 `maxColumns=3`，容器宽度能容纳 3 列（≥ 990px），配置了 7 个子 widget：

```
原始顺序:  [W0, W1, W2, W3, W4, W5, W6]

分配结果:
  Column 0   Column 1   Column 2
   ┌───┐     ┌───┐     ┌───┐
   │ W0│     │ W1│     │ W2│
   ├───┤     ├───┤     ├───┤
   │ W3│     │ W4│     │ W5│
   ├───┤     └───┘     └───┘
   │ W6│
   └───┘

i=0 → 0%3=0 → Col0
i=1 → 1%3=1 → Col1
i=2 → 2%3=2 → Col2
i=3 → 3%3=0 → Col0
i=4 → 4%3=1 → Col1
i=5 → 5%3=2 → Col2
i=6 → 6%3=0 → Col0
```

如果用户缩小窗口使容器宽度只能容纳 2 列：

```
重新分配 (columnsCount=2):
  Column 0   Column 1
   ┌───┐     ┌───┐
   │ W0│     │ W1│
   ├───┤     ├───┤
   │ W2│     │ W3│
   ├───┤     ├───┤
   │ W4│     │ W5│
   ├───┤     └───┘
   │ W6│
   └───┘
```

### 8.5 后端 MaxColumns 与前端的协作

- 后端 widget 的 `MaxColumns` 配置通过 `data-max-columns` 属性传递给前端
- 若用户未配置 `max-columns`，[widget-split-column.go:24-26](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/widget-split-column.go#L24-L26) 将默认值设为 2
- 前端 `clamp()` 确保实际列数不会超过此值，也不会超过子元素数量
