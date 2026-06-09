# Glance Widget 生命周期深度解析

本文档沿代码路径追踪 Widget 从 YAML 声明到屏幕渲染再到定时刷新的完整运转机制，涵盖类型注册、初始化、首次渲染与调度边界四个核心阶段。

---

## 0. 核心接口与基类

所有 Widget 都实现了 `widget` 接口，定义于 [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L126-L139)：

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

具体 Widget（如 `clockWidget`、`weatherWidget`）通过嵌入 `widgetBase` 结构体获得默认实现。`widgetBase` 定义于 [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L149-L167)，持有 ID、缓存策略、错误状态、渲染缓冲区。

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
          - type: rss
            feeds:
              - url: https://example.com/feed.xml
```

配置加载始于 [config.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/config.go)。

### 1.2 反序列化触发点：`newConfigFromYAML` 调用 `yaml.Unmarshal`，当解析到 `widgets` 字段时，会触发自定义 UnmarshalYAML` 方法。

### 1.3 `widgets.UnmarshalYAML` 方法定义于 [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L95-L124)。

执行流程：

1. 先读取每个节点的 `type` 字段
2. 调用 `newWidget(meta.Type)` 创建对应类型
3. 再用 `node.Decode(widget)` 将剩余字段解码到具体结构体

```go
func (w *widgets) UnmarshalYAML(node *yaml.Node) error {
    // ...
    for _, node := range nodes {
        meta := struct {
            Type string `yaml:"type"`
        }{}
        node.Decode(&meta)

        widget, err := newWidget(meta.Type)  // 类型注册
        // ...
        node.Decode(widget)                   // 字段填充
        *w = append(*w, widget)
    }
}
```

### 1.4 `newWidget` 函数定义于 [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L20-L91)，本质是一个巨大的 switch 语句，是所有 Widget 类型的注册中心：

```go
func newWidget(widgetType string) (widget, error) {
    switch widgetType {
    case "calendar":
        w = &calendarWidget{}
    case "clock":
        w = &clockWidget{}
    case "weather":
        w = &weatherWidget{}
    // ... 约 30+ 种类型
    }
    w.setID(widgetIDCounter.Add(1))  // 原子递增分配全局唯一 ID
    return w, nil
}
```

**关键点：**
- 使用全局 `widgetIDCounter` 是 `atomic.Uint64` 原子计数器，确保并发安全
- 每个 Widget 在创建时获得唯一 ID
- 这是「类型映射注册表」模式：字符串 → 具体 Go 结构体的工厂函数

### 1.5 容器型 Widget（如 `groupWidget`、`splitColumnWidget`）嵌入 `containerWidgetBase`，其 `Widgets widgets` 字段会递归触发上述解析流程。

---

## 2. 初始化：`initialize()`

### 2.1 触发时机

配置反序列化完成后，`newConfigFromYAML` 在 [config.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/config.go#L112-L126) 遍历所有 Page 的 HeadWidgets 和 Columns 中的所有 Widget，逐个调用 `initialize()`：

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

### 2.2 典型 `initialize()` 做什么

以 `weatherWidget.initialize()` 为例，定义于 [widget-weather.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-weather.go#L35-L57)：

```go
func (widget *weatherWidget) initialize() error {
    widget.withTitle("Weather").withCacheOnTheHour()  // 设置标题和缓存策略
    // 参数校验与默认值填充
    if widget.Location == "" {
        return fmt.Errorf("location is required")
    }
    // ...
}
```

以 `clockWidget.initialize()` 为例，定义于 [widget-clock.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-clock.go#L22-L44)：

```go
func (widget *clockWidget) initialize() error {
    widget.withTitle("Clock").withError(nil)
    // 参数校验
    // 立即渲染一次并缓存结果
    widget.cachedHTML = widget.renderTemplate(widget, clockWidgetTemplate)
    return nil
}
```

### 2.3 初始化阶段的核心职责

| 职责 | 对应方法/字段 | 说明 |
|---|---|---|
| 设置标题 | `withTitle()` | 设置 Widget 标题 |
| 设置缓存策略 | `withCacheDuration()` / `withCacheOnTheHour()` | 决定刷新频率 |
| 参数校验 | - | 校验必填字段、合法值 |
| 默认值填充 | - | 为可选字段赋默认值 |
| 预渲染（可选） | 立即渲染并缓存 HTML |
| 容器递归初始化 | `containerWidgetBase._initializeWidgets()` | 容器 Widget 递归子 Widget |

### 2.4 `widgetBase` 的缓存策略设置方法：

- `withCacheDuration(d)`：按固定时长缓存，如 RSS 2 小时
- `withCacheOnTheHour()`：整点刷新，如天气
- 不设置则默认为 `cacheTypeInfinite`（永不过期（如 Clock 这类静态 Widget

### 2.5 Providers 注入

在 `newApplication()` 在 [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/glance.go#L175-L193) 遍历所有 Widget，调用 `widget.setProviders(providers)注入 `assetResolver` 静态资源解析器。

---

## 3. 首次渲染

### 3.1 首次请求流程

用户浏览器请求页面时：

1. 服务端：`handlePageRequest` 渲染页面骨架（仅包含 `<div id="page-content">` 是空的
2. 客户端 JS `setupPage()` 在 [page.js](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/static/js/page.js#L746-L784) 异步请求 `/api/pages/{page}/content/
3. 服务端 `handlePageContentRequest` 在 [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/glance.go#L334-L367) 处理：

```go
func (a *application) handlePageContentRequest(...) {
    page.mu.Lock()
    defer page.mu.Unlock()

    page.updateOutdatedWidgets()   // 先更新需要更新的 Widget
    err = pageContentTemplate.Execute(&responseBytes, pageData)  // 再渲染
}
```

### 3.2 `page.updateOutdatedWidgets()` 在 [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/glance.go#L233-L270)：

```go
func (p *page) updateOutdatedWidgets() {
    now := time.Now()
    var wg sync.WaitGroup
    ctx := context.Background()

    // 遍历所有 HeadWidgets 和 Columns 中所有 Widget
    for ... {
        if !widget.requiresUpdate(&now) {
            continue
        }
        wg.Add(1)
        go func() {
            defer wg.Done()
            widget.update(ctx)   // 并发更新
        }()
    }
    wg.Wait()
}
```

### 3.3 `requiresUpdate()` 判断逻辑在 [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L173-L183)：

```go
func (w *widgetBase) requiresUpdate(now *time.Time) bool {
    if w.cacheType == cacheTypeInfinite {
        return false  // 永不过期
    }
    if w.nextUpdate.IsZero() {
        return true   // 首次，nextUpdate 是零值，需要更新
    }
    return now.After(w.nextUpdate)
}
```

**首次渲染关键：** `nextUpdate` 初始值是 `time.Time{}`（零值），所以 `requiresUpdate` 首次必定返回 `true`，触发 `update()`。

### 3.4 `update()` 典型实现

以 `weatherWidget.update()` 为例 [widget-weather.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-weather.go#L59-L77)：

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

### 3.5 `canContinueUpdateAfterHandlingErr` 在 [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L293-L325)：

- 无错误 → `scheduleNextUpdate()` 按正常策略排程下次更新
- 部分内容错误 → `scheduleEarlyUpdate()` 指数退避重试
- 严重错误 → 设置 `Error` 状态

### 3.6 模板渲染

`page-content.html` 模板在 [page-content.html](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/templates/page-content.html)：

```html
{{ range .Page.Columns }}
<div class="page-column page-column-{{ .Size }}">
    {{ range .Widgets }}
    {{ .Render }}
    {{ end }}
</div>
{{ end }}
```

每个 Widget 的 `Render()` 调用 `widgetBase.renderTemplate()` 在 [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L217-L241)，将 `widget-base.html` 在 [widget-base.html](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/templates/widget-base.html) 定义了统一外壳：

```html
<div class="widget widget-type-{{ .GetType }}">
    <div class="widget-header">...</div>
    <div class="widget-content">
        {{ if .ContentAvailable }}
            {{ block "widget-content" . }}
        {{ else }}
            <!-- 错误展示
        {{ end }}
    </div>
</div>
```

`widget-base.html` 通过 `block "widget-content"` 让具体 Widget 模板覆盖内容。

### 3.7 首次渲染时序：

```
YAML 解析 → widgets.UnmarshalYAML
    ↓
newWidget() 类型映射
    ↓
initialize() 参数校验 + 缓存策略
    ↓
用户请求页面
    ↓
handlePageContentRequest
    ↓
page.updateOutdatedWidgets()
    ↓
requiresUpdate() → nextUpdate 零值 → true
    ↓
widget.update() 并发获取数据
    ↓
canContinueUpdateAfterHandlingErr 排程下次更新
    ↓
.Render() → renderTemplate()
    ↓
widget-base.html + 具体模板
    ↓
返回 HTML
```

---

## 4. 刷新调度与边界

### 4.1 服务端调度

#### 4.1.1 客户端轮询

客户端没有显式轮询。页面加载完成后，没有主动轮询机制是**被动刷新**：每次客户端 JS 并没有自动请求 `/api/pages/{page}/content/` 时才触发服务端检查 `updateOutdatedWidgets()`。

#### 4.1.2 服务端的刷新边界：

服务端的调度完全由**请求驱动**，没有后台 goroutine 定时更新。

每次请求来临时才检查 `requiresUpdate()`。

#### 4.1.3 缓存策略三种类型在 [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L141-L147)：

| 策略 | 说明 | 示例 |
|---|---|---|
| `cacheTypeInfinite` | 永不更新 | Clock、Bookmarks |
| `cacheTypeDuration` | 固定间隔 | RSS（2小时） |
| `cacheTypeOnTheHour` | 整点刷新 | Weather |

#### 4.1.4 下次更新时间计算：

`getNextUpdateTime()` 在 [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L327-L341)：

- Duration 类型：`now.Add(cacheDuration)
- OnTheHour 类型：计算到下一个整点
- Infinite：返回零值（永不更新

#### 4.1.5 指数退避重试：

`scheduleEarlyUpdate()` 在 [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget.go#L350-L367)：

```go
func (w *widgetBase) scheduleEarlyUpdate() *widgetBase {
    w.updateRetriedTimes++
    if w.updateRetriedTimes > 5 {
        w.updateRetriedTimes = 5
    }
    // 指数退避：1²=1min, 2²=4min, 3²=9min, 4²=16min, 5²=25min
    nextEarlyUpdate := time.Now().Add(
        time.Duration(math.Pow(float64(w.updateRetriedTimes), 2)) * time.Minute)
    // 取「尽早更新」和「正常更新」取较早者
}
```

最多重试 5 次，平方退避，最多 25 分钟。

### 4.2 并发调度边界

#### 4.2.1 Page 级锁：page.mu

`handlePageContentRequest` 在 [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/glance.go#L352-L358)：

```go
page.mu.Lock()
defer page.mu.Unlock()
```

**同一 Page 的请求会串行执行 `updateOutdatedWidgets()` 和渲染是原子操作。

#### 4.2.2 Widget 级并发

Page 锁内，**同一 Page 内多个 Widget 的 `update()` 是并发执行的（goroutine + WaitGroup）。

```go
for ... {
    if !widget.requiresUpdate(&now) {
        continue
    }
    wg.Add(1)
    go func() { widget.update(ctx) }()
}
wg.Wait()
```

#### 4.2.3 容器型 Widget 的子 Widget 也是并发更新的，见 `containerWidgetBase._update()` 在 [widget-container.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/widget-container.go#L23-L42)。

#### 4.2.4 Singleflight 机制

`Singleflight[T]` 在 [singleflight.go](file:///d:/fz/0601/solo-dogfeeding/code/135-glance/internal/glance/singleflight.go) 用于避免重复请求。多个并发请求共享一次 fn() 的结果。（注：当前代码库中 Singleflight 已定义但未广泛使用。

#### 4.2.5 渲染边界：

- **渲染隔离：每个 Widget 有独立的 `templateBuffer`
- **错误隔离：渲染失败时，`renderTemplate()` 会**立即重新渲染一次错误状态，避免页面半渲染状态污染页面
- **内容边界：`ContentAvailable` 标记决定显示内容还是错误占位

### 4.3 前端调度边界（browser 页面加载后：

1. `setupPage()` 异步拉取 content
2. 各类 Widget 独立的 JS setupXxx()` 激活（`setupClocks`、`setupCalendars`、`setupDynamicRelativeTime` 等）
3. Clock 等前端每秒自更新（无需服务端）

```
setupClocks() 每分钟本地更新
    ↓
updateClocks → setTimeout(..., (60 - now.getSeconds()) * 1000)
```

RelativeTime 每分钟更新。

---

## 5. 两类 Widget 的生命周期对比

| 阶段 | Clock（静态型） | Weather（数据型） |
|---|---|---|
| 类型注册 | newWidget("clock") → &clockWidget{} | newWidget("weather") → &weatherWidget{} |
| initialize | 校验时区，**立即预渲染 cachedHTML | 校验 location，设置 cacheOnTheHour |
| requiresUpdate | cacheTypeInfinite → 永 false | 首次零值 → true；之后整点判断 |
| update | 空实现 | 调 Open-Meteo API |
| Render | 返回 cachedHTML | 实时 renderTemplate |
| 前端调度 | setupClocks 本地自更新 | 无，依赖下次请求触发服务端更新 |

---

## 6. 全链路流程图

```
glance.yml
    │
    ▼
yaml.Unmarshal ──────► widgets.UnmarshalYAML
                            │
                            ▼
                      newWidget(type) ──► switch 类型注册表
                            │
                            ▼
                      node.Decode(widget)
                            │
                            ▼
              ┌──────────────────────────────────┐
              │  initialize()           │
              │  • withTitle()        │
              │  • withCacheXXX()     │
              │  • 参数校验           │
              │  • （可选）预渲染     │
              └──────────────────────────────────┘
                            │
                            ▼
              newApplication() → setProviders()
                            │
                            ▼
            HTTP GET /{page}
                            │
                            ▼
            handlePageRequest → 输出页面骨架
                            │
                            ▼
            前端 setupPage() → fetchPageContent()
                            │
                            ▼
            GET /api/pages/{page}/content/
                            │
                            ▼
              ┌──────────────────────────────────┐
              │  page.mu.Lock()             │
              │  updateOutdatedWidgets()    │
              │    │                         │
              │    ├─ requiresUpdate() ? │
              │    └─ 并发 widget.update() │
              │  pageContentTemplate      │
              │  page.mu.Unlock()           │
              └──────────────────────────────────┘
                            │
                            ▼
                      .Render()
                            │
                            ▼
              widget-base.html 外壳
            + block "widget-content" 具体内容
                            │
                            ▼
                      前端 setupXxx() 激活（Clock、
```

---

## 7. 关键设计模式总结

1. **声明式配置驱动**：YAML → 结构体 → 生命周期方法，整条链路由配置驱动
2. **请求驱动刷新**：无后台定时器，每次请求时惰性检查更新
3. **Page 级原子性**：同 Page 更新+渲染持锁，Widget 间并发执行
4. **指数退避重试**：失败时早重试，最多 25 分钟封顶
5. **模板继承**：widget-base.html 提供统一外壳，block 供子模板覆盖内容
6. **渲染容错**：渲染失败立即重渲染错误状态，避免半开标签污染
