# Glance Widget 生命周期深度解析

本文档沿代码路径追踪 Widget 从 YAML 声明到屏幕渲染再到定时刷新的完整运转机制，重点剖析运行时注册表、交互刷新边界与容器递归链。

---

## 0. 核心接口与基类

所有 Widget 都实现了 `widget` 接口，定义于 [`internal/glance/widget.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L126-L139)：

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

具体 Widget（如 `clockWidget`、`weatherWidget`）通过嵌入 `widgetBase` 结构体获得默认实现。`widgetBase` 定义于 [`internal/glance/widget.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L149-L167)，持有 ID、缓存策略、错误状态、渲染缓冲区。

容器型 Widget（`groupWidget`、`splitColumnWidget`）额外嵌入 `containerWidgetBase`，定义于 [`internal/glance/widget-container.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-container.go#L9-L11)，持有 `Widgets widgets` 子节点列表。

---

## 1. 类型注册：从 YAML 到 Go 结构体

### 1.1 YAML 配置入口

用户在 `glance.yml` 中声明 Widget：

```yaml
pages:
  - name: Home
    columns:
      - size: full
        widgets:
          - type: weather
            location: "Beijing, CN"
          - type: group
            widgets:
              - type: clock
              - type: bookmarks
```

配置加载始于 [`internal/glance/config.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/config.go)。

### 1.2 反序列化触发点

`newConfigFromYAML` 调用 `yaml.Unmarshal`，当解析到 `widgets` 字段时，触发自定义的 `widgets.UnmarshalYAML` 方法。

### 1.3 `widgets.UnmarshalYAML`

方法定义于 [`internal/glance/widget.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L95-L124)。

执行流程：

1. 先读取每个节点的 `type` 字段
2. 调用 `newWidget(meta.Type)` 创建对应类型的 Widget 实例
3. 再用 `node.Decode(widget)` 将剩余 YAML 字段解码到具体结构体

```go
func (w *widgets) UnmarshalYAML(node *yaml.Node) error {
    for _, node := range nodes {
        meta := struct {
            Type string `yaml:"type"`
        }{}
        node.Decode(&meta)

        widget, err := newWidget(meta.Type)  // 类型注册 + 分配 ID
        node.Decode(widget)                   // 字段填充
        *w = append(*w, widget)
    }
}
```

**递归特性**：容器型 Widget（`group`、`split-column`）的 `Widgets widgets` 字段在 `node.Decode(widget)` 阶段会再次触发本方法，形成递归解析树。

### 1.4 `newWidget`：类型注册表

函数定义于 [`internal/glance/widget.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L20-L91)，本质是一个巨大的 switch 语句——所有 Widget 类型的硬编码注册中心：

```go
func newWidget(widgetType string) (widget, error) {
    switch widgetType {
    case "calendar":         w = &calendarWidget{}
    case "clock":            w = &clockWidget{}
    case "weather":          w = &weatherWidget{}
    case "group":            w = &groupWidget{}
    case "split-column":     w = &splitColumnWidget{}
    // ... 约 30+ 种类型
    }
    w.setID(widgetIDCounter.Add(1))  // 原子递增分配全局唯一 ID
    return w, nil
}
```

**关键点**：
- 全局 `widgetIDCounter` 是 `atomic.Uint64` 原子计数器，确保并发安全
- 每个 Widget（包括容器内的子 Widget）在创建时都获得全局唯一 ID
- 这是「类型映射注册表」模式：YAML 字符串 → 具体 Go 结构体的工厂函数

---

## 2. 初始化：`initialize()`

### 2.1 触发时机

配置反序列化完成后，`newConfigFromYAML` 在 [`internal/glance/config.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/config.go#L112-L126) 遍历所有 Page 的 HeadWidgets 和 Columns 中的**顶层** Widget，逐个调用 `initialize()`：

```go
for p := range config.Pages {
    for w := range config.Pages[p].HeadWidgets {
        config.Pages[p].HeadWidgets[w].initialize()
    }
    for c := range config.Pages[p].Columns {
        for w := range config.Pages[p].Columns[c].Widgets {
            config.Pages[p].Columns[c].Widgets[w].initialize()
        }
    }
}
```

**注意**：这里只遍历了**顶层** Widget。容器型 Widget 的子 Widget 初始化由容器自身递归完成。

### 2.2 典型 `initialize()` 实现

以 `weatherWidget.initialize()` 为例，定义于 [`internal/glance/widget-weather.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-weather.go#L35-L57)：

```go
func (widget *weatherWidget) initialize() error {
    widget.withTitle("Weather").withCacheOnTheHour()  // 设置标题和缓存策略
    if widget.Location == "" {
        return fmt.Errorf("location is required")
    }
    // 更多校验...
    return nil
}
```

以 `clockWidget.initialize()` 为例，定义于 [`internal/glance/widget-clock.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-clock.go#L22-L44)：

```go
func (widget *clockWidget) initialize() error {
    widget.withTitle("Clock").withError(nil)
    // 参数校验
    widget.cachedHTML = widget.renderTemplate(widget, clockWidgetTemplate)
    return nil
}
```

### 2.3 容器型 Widget 的初始化递归链

以 `groupWidget.initialize()` 为例，定义于 [`internal/glance/widget-group.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-group.go#L17-L36)：

```go
func (widget *groupWidget) initialize() error {
    widget.withError(nil)
    widget.HideHeader = true

    for i := range widget.Widgets {
        widget.Widgets[i].setHideHeader(true)
    }

    if err := widget.containerWidgetBase._initializeWidgets(); err != nil {
        return err
    }
    return nil
}
```

`_initializeWidgets()` 定义于 [`internal/glance/widget-container.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-container.go#L13-L21)：

```go
func (widget *containerWidgetBase) _initializeWidgets() error {
    for i := range widget.Widgets {
        if err := widget.Widgets[i].initialize(); err != nil {
            return formatWidgetInitError(err, widget.Widgets[i])
        }
    }
    return nil
}
```

`splitColumnWidget.initialize()` 同理，定义于 [`internal/glance/widget-split-column.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-split-column.go#L17-L29)，同样调用 `_initializeWidgets()`。

**递归初始化链**：
```
page level initialize() 顶层遍历
    └─► groupWidget.initialize()
           └─► _initializeWidgets()
                  ├─► clock.initialize()
                  └─► bookmarks.initialize()
```

### 2.4 初始化阶段的核心职责

| 职责 | 对应方法/字段 | 说明 |
|---|---|---|
| 设置标题 | `withTitle()` | 设置 Widget 标题 |
| 设置缓存策略 | `withCacheDuration()` / `withCacheOnTheHour()` | 决定刷新频率 |
| 参数校验 | — | 校验必填字段、合法值范围 |
| 默认值填充 | — | 为可选字段赋默认值 |
| 预渲染（可选） | — | 立即渲染模板并缓存 HTML（如 Clock） |
| 容器递归初始化 | `containerWidgetBase._initializeWidgets()` | 遍历子 Widget 并调用其 initialize() |

### 2.5 `widgetBase` 的三种缓存策略

- `withCacheDuration(d)`：按固定时长缓存，如 RSS 为 2 小时
- `withCacheOnTheHour()`：整点刷新，如 Weather
- 不调用任何方法则默认为 `cacheTypeInfinite`（永不过期），如 Clock、Bookmarks

---

## 3. 运行时注册表：`widgetByID` 的建立

### 3.1 注册表结构

`application` 结构体持有运行时注册表，定义于 [`internal/glance/glance.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/glance.go#L30-L45)：

```go
type application struct {
    // ...
    slugToPage map[string]*page
    widgetByID map[uint64]widget   // <-- Widget 运行时注册表：ID → Widget 实例
    // ...
}
```

### 3.2 初始化位置

在 `newApplication()` 中创建空 map，见 [`internal/glance/glance.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/glance.go#L47-L54)：

```go
app := &application{
    // ...
    widgetByID: make(map[uint64]widget),
}
```

### 3.3 注册过程

`newApplication()` 在 [`internal/glance/glance.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/glance.go#L153-L194) 遍历所有 Page 的**顶层** Widget 进行注册：

```go
for p := range config.Pages {
    page := &config.Pages[p]
    // ...

    // 注册 HeadWidgets（顶层）
    for i := range page.HeadWidgets {
        widget := page.HeadWidgets[i]
        app.widgetByID[widget.GetID()] = widget     // 写入注册表
        widget.setProviders(providers)              // 同时注入 Providers
    }

    // 注册 Columns 中的 Widget（顶层）
    for c := range page.Columns {
        column := &page.Columns[c]
        for w := range column.Widgets {
            widget := column.Widgets[w]
            app.widgetByID[widget.GetID()] = widget  // 写入注册表
            widget.setProviders(providers)           // 同时注入 Providers
        }
    }
}
```

### 3.4 ⚠️ 重要边界：容器子 Widget 的注册盲区

**经代码核对确认**：`widgetByID` 只注册了**顶层** Widget（`page.HeadWidgets` 和 `page.Columns[].Widgets`）。

- 如果一个 `groupWidget` 出现在 Column 中，只有 `groupWidget` 自身被注册到 `widgetByID`
- 其内部的 `clockWidget`、`bookmarksWidget` 等子 Widget **不会**被注册到 `widgetByID`
- 子 Widget 的 `setProviders()` 同样不是在 `newApplication()` 顶层完成的，而是由容器递归注入

这是一个设计上的盲区：容器内部的子 Widget 虽然有自己的全局唯一 ID（由 `newWidget` 中的 `atomic.Uint64` 分配），但在 `widgetByID` 中找不到。

### 3.5 Providers 注入的递归链

与初始化类似，Providers 的注入也分两层：

**顶层注入**（`newApplication()`）：
```go
widget.setProviders(providers)
```

**容器递归注入**（以 `groupWidget` 为例），见 [`internal/glance/widget-group.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-group.go#L42-L44)：

```go
func (widget *groupWidget) setProviders(providers *widgetProviders) {
    widget.containerWidgetBase._setProviders(providers)
}
```

`splitColumnWidget.setProviders()` 同理，见 [`internal/glance/widget-split-column.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-split-column.go#L35-L37)。

`_setProviders()` 定义于 [`internal/glance/widget-container.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-container.go#L44-L48)：

```go
func (widget *containerWidgetBase) _setProviders(providers *widgetProviders) {
    for i := range widget.Widgets {
        widget.Widgets[i].setProviders(providers)
    }
}
```

**递归注入链**：
```
newApplication() setProviders (顶层)
    └─► groupWidget.setProviders()
           └─► _setProviders()
                  ├─► clock.setProviders()
                  └─► bookmarks.setProviders()
```

---

## 4. 首次渲染

### 4.1 两段式渲染流程

用户浏览器请求页面时采用两段式渲染：

1. **服务端输出骨架**：`handlePageRequest` 渲染 `page.html`，页面内 `<div id="page-content">` 是空容器
2. **前端异步拉内容**：JS `setupPage()` 在 [`internal/glance/static/js/page.js`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/static/js/page.js#L746-L784) 异步请求 `/api/pages/{page}/content/`
3. **服务端渲染内容**：`handlePageContentRequest` 在 [`internal/glance/glance.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/glance.go#L334-L367) 处理：

```go
func (a *application) handlePageContentRequest(...) {
    page.mu.Lock()
    defer page.mu.Unlock()

    page.updateOutdatedWidgets()   // 先更新需要更新的 Widget
    err = pageContentTemplate.Execute(&responseBytes, pageData)  // 再渲染
}
```

### 4.2 `page.updateOutdatedWidgets()`

定义于 [`internal/glance/glance.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/glance.go#L233-L270)：

```go
func (p *page) updateOutdatedWidgets() {
    now := time.Now()
    var wg sync.WaitGroup
    ctx := context.Background()

    // 遍历所有顶层 HeadWidgets
    for w := range p.HeadWidgets {
        widget := p.HeadWidgets[w]
        if !widget.requiresUpdate(&now) { continue }
        wg.Add(1)
        go func() {
            defer wg.Done()
            widget.update(ctx)
        }()
    }

    // 遍历所有顶层 Columns.Widgets
    for c := range p.Columns {
        for w := range p.Columns[c].Widgets {
            widget := p.Columns[c].Widgets[w]
            if !widget.requiresUpdate(&now) { continue }
            wg.Add(1)
            go func() {
                defer wg.Done()
                widget.update(ctx)
            }()
        }
    }
    wg.Wait()
}
```

### 4.3 Update 的容器递归链

与 `initialize` 和 `setProviders` 相同，容器型 Widget 的 `update()` 也会递归调度子 Widget。

以 `groupWidget.update()` 为例，见 [`internal/glance/widget-group.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-group.go#L38-L40)：

```go
func (widget *groupWidget) update(ctx context.Context) {
    widget.containerWidgetBase._update(ctx)
}
```

`splitColumnWidget.update()` 同理，见 [`internal/glance/widget-split-column.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-split-column.go#L31-L33)。

`_update()` 定义于 [`internal/glance/widget-container.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-container.go#L23-L42)：

```go
func (widget *containerWidgetBase) _update(ctx context.Context) {
    var wg sync.WaitGroup
    now := time.Now()

    for w := range widget.Widgets {
        widget := widget.Widgets[w]
        if !widget.requiresUpdate(&now) { continue }
        wg.Add(1)
        go func() {
            defer wg.Done()
            widget.update(ctx)
        }()
    }
    wg.Wait()
}
```

同样，`requiresUpdate` 也是递归的，以 `groupWidget.requiresUpdate()` 为例见 [`internal/glance/widget-group.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-group.go#L46-L48)，其调用的 `_requiresUpdate()` 定义于 [`internal/glance/widget-container.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-container.go#L50-L58)：只要有任意一个子 Widget 需要更新，容器就返回 `true`。

**递归 update 链**：
```
page.updateOutdatedWidgets() (顶层并发 goroutine)
    └─► groupWidget.update()
           └─► _update() (第二层并发 goroutine)
                  ├─► clock.update() (空实现)
                  └─► weather.update() (HTTP 请求)
```

### 4.4 `requiresUpdate()` 判断逻辑

`widgetBase` 的默认实现定义于 [`internal/glance/widget.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L173-L183)：

```go
func (w *widgetBase) requiresUpdate(now *time.Time) bool {
    if w.cacheType == cacheTypeInfinite {
        return false  // 永不过期
    }
    if w.nextUpdate.IsZero() {
        return true   // 首次：nextUpdate 零值 → 需要更新
    }
    return now.After(w.nextUpdate)
}
```

**首次渲染关键**：`nextUpdate` 初始值是 `time.Time{}`（零值），所以首次 `requiresUpdate` 必定返回 `true`，触发 `update()`。

### 4.5 `update()` 典型实现与错误处理

以 `weatherWidget.update()` 为例 [`internal/glance/widget-weather.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-weather.go#L59-L77)：

```go
func (widget *weatherWidget) update(ctx context.Context) {
    if widget.Place == nil {
        place, err := fetchOpenMeteoPlaceFromName(widget.Location)
        if err != nil {
            widget.withError(err).scheduleEarlyUpdate()
            return
        }
        widget.Place = place
    }

    weather, err := fetchWeatherForOpenMeteoPlace(widget.Place, widget.Units)
    if !widget.canContinueUpdateAfterHandlingErr(err) {
        return
    }
    widget.Weather = weather
}
```

`canContinueUpdateAfterHandlingErr` 在 [`internal/glance/widget.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L293-L325)：
- 无错误 → `scheduleNextUpdate()` 按正常策略排程下次更新
- 部分内容错误（`errPartialContent`）→ `scheduleEarlyUpdate()` 指数退避重试，设置 Notice
- 严重错误 → 设置 `Error` 状态，`scheduleEarlyUpdate()`

### 4.6 模板渲染

`page-content.html` 模板在 [`internal/glance/templates/page-content.html`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/templates/page-content.html)：

```html
{{ range .Page.Columns }}
<div class="page-column page-column-{{ .Size }}">
    {{ range .Widgets }}
    {{ .Render }}
    {{ end }}
</div>
{{ end }}
```

每个 Widget 的 `Render()` 调用 `widgetBase.renderTemplate()`，定义于 [`internal/glance/widget.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L217-L241)。

`widget-base.html` 在 [`internal/glance/templates/widget-base.html`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/templates/widget-base.html) 定义了统一外壳，通过 `{{ block "widget-content" . }}` 让具体 Widget 模板覆盖内容区域。

**渲染容错**：若渲染出错，`renderTemplate()` 会**立即重新执行一次模板渲染**（此时 Widget 已被设置 Error 状态），避免模板半渲染导致标签未闭合而污染整个页面。

---

## 5. 交互刷新边界：`/api/widgets` 为何 NotImplemented

### 5.1 路由注册

路由注册于 [`internal/glance/glance.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/glance.go#L436-L457)：

```go
mux.HandleFunc("/api/widgets/{widget}/{path...}", a.handleWidgetRequest)
```

### 5.2 当前实现

`handleWidgetRequest` 定义于 [`internal/glance/glance.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/glance.go#L404-L425)：

```go
func (a *application) handleWidgetRequest(w http.ResponseWriter, r *http.Request) {
    // TODO: this requires a rework of the widget update logic so that rather
    // than locking the entire page we lock individual widgets
    w.WriteHeader(http.StatusNotImplemented)

    // 以下为被注释掉的参考实现：
    // widgetValue := r.PathValue("widget")
    // widgetID, err := strconv.ParseUint(widgetValue, 10, 64)
    // widget, exists := a.widgetByID[widgetID]
    // if !exists { ... }
    // widget.handleRequest(w, r)
}
```

### 5.3 三大阻塞原因（经代码核对）

这条路由被 NotImplemented 并非遗漏，而是遇到了三个结构性障碍：

#### 障碍一：锁粒度过粗（TODO 注释明确说明）

当前更新机制使用 **Page 级互斥锁** `page.mu`，见 [`internal/glance/glance.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/glance.go#L352-L358)：

```go
// handlePageContentRequest 中：
page.mu.Lock()
defer page.mu.Unlock()
page.updateOutdatedWidgets()
```

如果开放 `/api/widgets/{id}` 单 Widget 路由，会与 Page 级更新产生竞态——两个请求可能同时修改同一个 Widget 的状态。要安全开放此路由，必须将锁从 Page 级细化到 Widget 级，涉及大量重构。

#### 障碍二：`widgetByID` 注册表不完整（参见第 3.4 节）

- 容器内部的子 Widget 虽然有 ID，但没有注册到 `widgetByID`
- 通过 `/api/widgets/{id}` 访问这些子 Widget 会直接 404
- 需要将 `widgetByID` 注册改为递归遍历（容器内子 Widget 也需注册）

#### 障碍三：没有任何 Widget 重写 `handleRequest`

经全代码库 grep 核对，只有 `widgetBase` 提供默认实现，定义于 [`internal/glance/widget.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L205-L207)：

```go
func (widget *widgetBase) handleRequest(w http.ResponseWriter, r *http.Request) {
    http.Error(w, "not implemented", http.StatusNotImplemented)
}
```

没有任何具体 Widget（Todo、Calendar 等）提供自己的 `handleRequest` 实现。即使开放路由，所有请求也会返回 501。

> **补充观察**：Todo Widget 目前将数据存在浏览器 `localStorage`（前端 JS 实现），Calendar 也是纯前端 JS 组件——两者都不需要服务端交互路由。这也是 `handleRequest` 一直无人实现的原因。

---

## 6. 刷新调度与并发边界

### 6.1 请求驱动的被动刷新

Glance 没有后台 goroutine 做定时更新。刷新完全由请求驱动：

- 每次浏览器请求 `/api/pages/{page}/content/` 时才检查 `requiresUpdate()`
- 浏览器本身也没有自动轮询，仅在首次加载和手动刷新时触发

### 6.2 三种缓存策略

定义于 [`internal/glance/widget.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L141-L147)：

| 策略 | 说明 | 示例 |
|---|---|---|
| `cacheTypeInfinite` | 永不更新 | Clock、Bookmarks |
| `cacheTypeDuration` | 固定间隔 | RSS（2小时） |
| `cacheTypeOnTheHour` | 整点刷新 | Weather |

### 6.3 指数退避重试

`scheduleEarlyUpdate()` 定义于 [`internal/glance/widget.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L350-L367)：

```go
func (w *widgetBase) scheduleEarlyUpdate() *widgetBase {
    w.updateRetriedTimes++
    if w.updateRetriedTimes > 5 { w.updateRetriedTimes = 5 }
    // 指数退避：1²=1min, 2²=4min, 3²=9min, 4²=16min, 5²=25min
    nextEarlyUpdate := time.Now().Add(
        time.Duration(math.Pow(float64(w.updateRetriedTimes), 2)) * time.Minute)
    // 取「尽早更新」与「正常更新」的较早者
}
```

最多重试 5 次，平方退避，最多 25 分钟封顶。

### 6.4 并发边界层次

| 层次 | 机制 | 粒度 |
|---|---|---|
| Page 级 | `page.mu` 互斥锁 | 同 Page 的 update+渲染 串行执行 |
| Widget 级 | `sync.WaitGroup` + goroutine | 同 Page 内多个 Widget 的 update 并发执行 |
| 容器子 Widget 级 | 第二层 WaitGroup + goroutine | 同容器内多个子 Widget 的 update 并发执行 |
| 渲染级 | 独立 `templateBuffer` | 每个 Widget 有独立的渲染缓冲区，互不干扰 |

### 6.5 Singleflight 机制

`Singleflight[T]` 定义于 [`internal/glance/singleflight.go`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/singleflight.go)，用于避免重复请求。多个并发调用共享一次 `fn()` 的结果。

> 注：当前代码库中 Singleflight 已定义并实现完成，但尚未被广泛用于 Widget 更新流程中。

---

## 7. 前端调度边界

浏览器页面加载后在 [`internal/glance/static/js/page.js`](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/static/js/page.js#L746-L784) 执行：

1. `fetchPageContent()` 异步拉取服务端渲染好的 HTML
2. 注入 `#page-content`
3. 逐个调用 `setupXxx()` 激活各类 Widget 的前端交互：
   - `setupClocks()`：Clock 本地每分钟自更新（`setTimeout` 对齐到下一分钟）
   - `setupDynamicRelativeTime()`：相对时间每分钟刷新
   - `setupCalendars()`、`setupTodos()`：懒加载对应 JS 模块
   - `setupSearchBoxes()`、`setupGroups()`、`setupMasonries()` 等

Clock、Todo、Calendar 等属于**前端自更新型 Widget**——它们的数据更新完全在本地浏览器完成，不依赖服务端刷新。

---

## 8. 两类 Widget 的生命周期对比

| 阶段 | Clock（静态/前端自更新型） | Weather（数据/服务端刷新型） |
|---|---|---|
| 类型注册 | `newWidget("clock") → &clockWidget{}` | `newWidget("weather") → &weatherWidget{}` |
| initialize | 校验时区，**立即预渲染** cachedHTML | 校验 location，设置 `cacheOnTheHour` |
| widgetByID 可见 | 顶层注册可见；若在 group 内则不可见 | 顶层注册可见；若在 group 内则不可见 |
| requiresUpdate | `cacheTypeInfinite` → 永 false | 首次零值 → true；之后整点判断 |
| update | 空实现（数据由前端 JS 更新） | 调用 Open-Meteo HTTP API |
| Render | 返回 `cachedHTML` | 实时 `renderTemplate` |
| 前端调度 | `setupClocks` 本地每分钟自更新 | 无，依赖下次请求触发服务端更新 |

---

## 9. 全链路流程图

```
glance.yml
    │
    ▼
yaml.Unmarshal ──────► widgets.UnmarshalYAML
                            │
                            ▼
                      newWidget(type) ──► switch 类型注册表 + atomic ID
                            │
                            ▼
                      node.Decode(widget)
                            │
                            ▼
              ┌──────────────────────────────────────────┐
              │ newConfigFromYAML initialize 顶层遍历    │
              │   └─► group.initialize()                 │
              │          └─► _initializeWidgets() 递归  │
              └──────────────────────────────────────────┘
                            │
                            ▼
              ┌──────────────────────────────────────────┐
              │ newApplication()                         │
              │   • widgetByID 仅注册顶层（子盲区）      │
              │   • setProviders 顶层                    │
              │      └─► group.setProviders() 递归      │
              └──────────────────────────────────────────┘
                            │
                            ▼
            HTTP GET /{page} → handlePageRequest → 骨架 HTML
                            │
                            ▼
            前端 setupPage() → fetchPageContent()
                            │
                            ▼
            GET /api/pages/{page}/content/
                            │
                            ▼
              ┌──────────────────────────────────────────┐
              │  page.mu.Lock()                          │
              │  updateOutdatedWidgets() 顶层遍历        │
              │    ├─ requiresUpdate() ?                 │
              │    └─ 顶层 goroutine 并发 widget.update()│
              │          └─► group._update() 子层并发    │
              │  pageContentTemplate.Execute()           │
              │  page.mu.Unlock()                        │
              └──────────────────────────────────────────┘
                            │
                            ▼
                      .Render() ──► widget-base.html + block 子模板
                            │
                            ▼
              前端 setupXxx() 激活（Clock / Calendar / Todo ...）

            ╔══════════════════════════════════════════════╗
            ║ /api/widgets/{id} — 501 NotImplemented       ║
            ║   • 阻塞1：需要 Page 锁 → Widget 级锁重构    ║
            ║   • 阻塞2：widgetByID 容器子 Widget 缺失    ║
            ║   • 阻塞3：无 Widget 重写 handleRequest     ║
            ╚══════════════════════════════════════════════╝
```

---

## 10. 关键设计模式总结

1. **声明式配置驱动**：YAML → 结构体 → 生命周期方法，整条链路由配置驱动
2. **请求驱动刷新**：无后台定时器，每次请求时惰性检查更新
3. **四维递归链**：容器型 Widget 在 `initialize`、`setProviders`、`update`、`requiresUpdate` 四个维度均递归传递给子 Widget
4. **Page 级原子性 + Widget 级并发**：同 Page 更新+渲染持锁，Widget 间（含容器子 Widget）goroutine 并发执行
5. **指数退避重试**：失败时早重试，平方递增，25 分钟封顶
6. **模板继承**：`widget-base.html` 提供统一外壳，`{{ block "widget-content" }}` 供子模板覆盖内容
7. **渲染容错**：渲染失败立即重渲染错误状态，避免半开标签污染整页
8. **注册表盲区**：`widgetByID` 仅顶层可见，容器子 Widget 缺失注册，阻碍单 Widget API 开放
