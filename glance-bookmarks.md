# Glance 书签分组与图标回退代码实现梳理

本文档从代码实现角度梳理 Glance 书签（Bookmarks）Widget 的四大核心机制：**分组渲染**、**图标选择与回退**、**布局均衡**、**配置来源**。

---

## 一、分组渲染

### 1.1 数据结构

书签 Widget 的核心结构体定义在 [widget-bookmarks.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/widget-bookmarks.go#L9-L35)：

```go
type bookmarksWidget struct {
    widgetBase `yaml:",inline"`
    cachedHTML template.HTML `yaml:"-"`
    Groups     []struct {
        Title     string
        Color     *hslColorField
        SameTab   bool
        HideArrow bool
        Target    string
        Links     []struct {
            Title        string
            URL          string
            Description  string
            Icon         customIconField
            SameTabRaw   *bool   // Raw 指针用于判断用户是否显式设置
            SameTab      bool    // 模板使用的最终值
            HideArrowRaw *bool
            HideArrow    bool
            Target       string
        }
    }
}
```

**设计要点**：`SameTab/HideArrow` 使用双字段设计。YAML 指针字段（`SameTabRaw`）用于区分「用户未设置」与「用户设置为 false」，最终计算值（`SameTab`）供模板直接使用，避开 Go template 无法解引用指针的限制。

### 1.2 初始化与属性继承

初始化逻辑位于 [widget-bookmarks.go#initialize](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/widget-bookmarks.go#L37-L73)，完成属性继承链的计算：

| 属性 | 优先级（高→低） | 回退默认值 |
|------|----------------|-----------|
| `SameTab` | Link.SameTabRaw → Group.SameTab | `false` |
| `HideArrow` | Link.HideArrowRaw → Group.HideArrow | `false` |
| `Target` | Link.Target → Group.Target → (SameTab ? "" : "_blank") | `_blank` |

```
Link.Target 非空? ──是──→ 直接使用
     │否
Group.Target 非空? ──是──→ 使用 Group.Target
     │否
Link.SameTab == true? ──是──→ "" (同标签页)
     │否
     └──────→ "_blank" (新标签页)
```

初始化完成后调用 `renderTemplate()` 将渲染结果缓存到 `cachedHTML`，后续每次 `Render()` 直接返回缓存，避免重复计算。

### 1.3 模板渲染结构

模板定义在 [bookmarks.html](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/templates/bookmarks.html)，DOM 结构如下：

```
div.dynamic-columns.list-gap-24.list-with-separator
├── div.bookmarks-group  (style="--bookmarks-group-color: hsl(...)")
│   ├── div.bookmarks-group-title   (分组标题，可选)
│   └── ul.list.list-gap-2
│       └── li × N
│           ├── div.flex.items-center.gap-10
│           │   ├── div.bookmarks-icon-container  (可选，存在 Icon.URL 时)
│           │   │   └── img.bookmarks-icon[.flat-icon]
│           │   └── a.bookmarks-link[.bookmarks-link-no-arrow]
│           └── div.margin-bottom-5  (描述，可选)
└── div.bookmarks-group × (N-1)
```

分组颜色通过 CSS 自定义属性 `--bookmarks-group-color` 下传，标题文字与链接箭头均引用该变量，默认回退到 `var(--color-primary)`（见 [widget-bookmarks.css](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/static/css/widget-bookmarks.css#L1-L16)）。

---

## 二、图标选择与回退

### 2.1 customIconField 数据结构

图标字段定义在 [config-fields.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config-fields.go#L132-L135)：

```go
type customIconField struct {
    URL        template.URL
    AutoInvert bool
}
```

### 2.2 图标 URL 解析规则

解析函数 [newCustomIconField](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config-fields.go#L137-L178) 支持以下格式：

**Step 1 — 检测 `auto-invert` 前缀**：
```
"auto-invert <任意格式>"  →  AutoInvert = true，剥离前缀后继续解析
```

**Step 2 — 按 `:` 拆分前缀与图标名**：
```
si:github       → prefix="si", icon="github"
di:jellyfin.png → prefix="di", icon="jellyfin.png"
https://...     → 无冒号，整串作为 URL 直出
```

**Step 3 — 扩展名识别**（默认 `svg`，仅支持 `svg/png`）：

**Step 4 — 前缀映射到 CDN**：

| 前缀 | 图标库 | CDN 路径 | 自动 AutoInvert |
|------|--------|----------|----------------|
| `si` | Simple Icons | `cdn.jsdelivr.net/npm/simple-icons@latest/icons/{name}.svg` | ✅ |
| `di` | Dashboard Icons | `cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/{ext}/{name}.{ext}` | ❌ |
| `mdi` | Material Design Icons | `cdn.jsdelivr.net/npm/@mdi/svg@latest/svg/{name}.svg` | ✅ |
| `sh` | selfh.st Icons | `cdn.jsdelivr.net/gh/selfhst/icons/{ext}/{name}.{ext}` | ❌ |
| 其它 | 原样 URL | 直接使用输入值 | 视 `auto-invert` 前缀 |

### 2.3 模板渲染与条件显示

在 [bookmarks.html#14-18](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/templates/bookmarks.html#L14-L18) 中：

```html
{{- if ne "" .Icon.URL }}
<div class="bookmarks-icon-container">
    <img class="bookmarks-icon{{ if .Icon.AutoInvert }} flat-icon{{ end }}"
         src="{{ .Icon.URL }}" alt="" loading="lazy">
</div>
{{- end }}
```

- **显示回退**：`Icon.URL` 为空时整个图标容器不渲染，链接文字自然左移对齐
- **AutoInvert 标记**：`AutoInvert=true` 时追加 `flat-icon` 类

### 2.4 暗色主题图标反色（flat-icon 机制）

在 [utils.css#379-381](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/static/css/utils.css#L379-L381)：

```css
:root:not([data-scheme=light]) .flat-icon {
    filter: invert(1);
}
```

这是「图标回退」的核心：当根元素不处于 `data-scheme=light`（即暗色主题）时，所有带 `flat-icon` 类的图标通过 CSS `filter: invert(1)` 反色。这要求原始图标为黑色系，反色后变为白色系以适配暗色背景。

`si:`（Simple Icons）和 `mdi:`（Material Design Icons）前缀会自动开启 `AutoInvert=true`，用户也可通过 `auto-invert` 前缀手动为其他图标开启此行为。

### 2.5 图标视觉样式

在 [widget-bookmarks.css#18-31](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/static/css/widget-bookmarks.css#L18-L31)：

```css
.bookmarks-icon-container {
    background-color: var(--color-widget-background-highlight);
    border-radius: var(--border-radius);
    padding: 0.5rem;
    opacity: 0.7;
    flex-shrink: 0;
}
.bookmarks-icon {
    width: 20px;
    height: 20px;
    opacity: 0.8;
}
```

图标被包裹在一个半透明圆角背景块内，尺寸固定 20×20px，`flex-shrink: 0` 保证空间不足时图标不被压缩。

---

## 三、布局均衡（dynamic-columns 机制）

### 3.1 核心思路

书签分组布局使用 [utils.css](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/static/css/utils.css#L94-L163) 中定义的 `.dynamic-columns` 类，通过 **CSS `:has()` 伪类 + CSS 自定义属性 + Container Queries** 纯 CSS 实现「根据子元素数量与容器宽度自适应列数」的均衡布局。

### 3.2 基础 Grid 定义

```css
.dynamic-columns {
    display: grid;
    grid-template-columns: repeat(var(--columns-per-row), 1fr);
    gap: var(--widget-content-vertical-padding) var(--widget-content-horizontal-padding);
}
```

列数由 `--columns-per-row` 变量控制，所有列等宽（`1fr`）。

### 3.3 列数动态计算（:has + nth-child）

```css
.dynamic-columns:has(> :nth-child(1)) { --columns-per-row: 1; }
.dynamic-columns:has(> :nth-child(2)) { --columns-per-row: 2; }
.dynamic-columns:has(> :nth-child(3)) { --columns-per-row: 3; }
.dynamic-columns:has(> :nth-child(4)) { --columns-per-row: 4; }
.dynamic-columns:has(> :nth-child(5)) { --columns-per-row: 5; }
```

原理：`:nth-child(N)` 选择器只有在存在第 N 个子元素时才匹配，`:has()` 据此判断实际子元素数量，将列数设为 `min(子元素数, 5)`。例如有 3 个分组时，上面三条规则全部命中，最后一条生效（`--columns-per-row: 3`），实现自动均衡。

### 3.4 响应式断点（Container Queries）

布局基于容器宽度（`@container widget`）而非视口宽度，支持在任意宽度的列中正确展示：

| 容器宽度 | 最大列数 |
|----------|---------|
| < 599px | 1（单列纵向，分隔线改为上下 border-top） |
| 600 – 849px | 2 |
| 850 – 1249px | 3 |
| 1250 – 1499px | 4 |
| ≥ 1500px | 5 |

移动端点（<550px 视口）在 [mobile.css#208](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/static/css/mobile.css#L208) 额外强制 `--columns-per-row: 1`。

### 3.5 分隔线规则

- **桌面端（多列）**：除首个子元素外，其余元素左侧 `padding-left` + `border-left: 1px solid var(--color-separator)`，形成列间竖线分隔
- **移动端（单列）**：除首个子元素外，其余元素上方 `margin-top` + `border-top`，形成行间隔线分隔

首元素始终无 padding 和 border，避免多余留白。

---

## 四、配置来源

### 4.1 配置文件加载链路

```
glance.yml (YAML 字节流)
    │
    ▼
parseConfigVariables()   // 替换 ${ENV_VAR} 等变量
    │
    ▼
yaml.Unmarshal → config 结构体 (config.go)
    │
    ├─ Pages[].HeadWidgets[]
    │       └─ widgets.UnmarshalYAML()
    │
    └─ Pages[].Columns[].Widgets[]
            └─ widgets.UnmarshalYAML()
                    │
                    ▼
            1. 读 type 字段 → newWidget("bookmarks") → &bookmarksWidget{}
            2. node.Decode(widget) 完整解析 YAML
            3. widget.initialize()  // 计算属性继承、缓存 HTML
```

### 4.2 bookmarks 配置结构（YAML → Go 映射）

| YAML 路径 | Go 字段 | 类型 | 解析器 |
|-----------|---------|------|--------|
| `groups[].title` | `Group.Title` | string | 标准 |
| `groups[].color` | `Group.Color` | `*hslColorField` | 自定义 UnmarshalYAML |
| `groups[].same-tab` | `Group.SameTab` | bool | 标准 |
| `groups[].hide-arrow` | `Group.HideArrow` | bool | 标准 |
| `groups[].target` | `Group.Target` | string | 标准 |
| `groups[].links[].title` | `Link.Title` | string | 标准 |
| `groups[].links[].url` | `Link.URL` | string | 标准 |
| `groups[].links[].description` | `Link.Description` | string | 标准 |
| `groups[].links[].icon` | `Link.Icon` | `customIconField` | 自定义 UnmarshalYAML |
| `groups[].links[].same-tab` | `Link.SameTabRaw` | `*bool` | 指针区分「未设置」 |
| `groups[].links[].hide-arrow` | `Link.HideArrowRaw` | `*bool` | 指针区分「未设置」 |
| `groups[].links[].target` | `Link.Target` | string | 标准 |

### 4.3 hslColorField 颜色解析

定义在 [config-fields.go#25-94](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config-fields.go#L25-L94)。支持以下等价格式：

```
"hsl(230, 100%, 30%)"
"hsla(230, 100%, 30%)"
"230 100 30"
"230,100,30"
```

正则表达式：`^(?:hsla?\()?([\d\.]+)(?: |,)+([\d\.]+)%?(?: |,)+([\d\.]+)%?\)?$`

三值范围校验：H ∈ [0, 360]，S ∈ [0, 100]，L ∈ [0, 100]。

### 4.4 Widget 创建与分发

在 [widget.go#newWidget](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/widget.go#L20-L91) 的 `switch` 语句中，`"bookmarks"` case 返回 `&bookmarksWidget{}`，并分配自增 ID。

[widgets.UnmarshalYAML](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/widget.go#L95-L124) 采用**两阶段解析**：先只解析出 `type` 字段创建具体类型，再将整个节点解码到该实例，确保 YAML tag 被正确路由。

### 4.5 配置热加载

在 [config.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config.go) 中通过 `fsnotify` 监听配置文件变更，修改后自动重新执行上述整个加载流程。若新配置解析失败，保留旧配置继续运行并在控制台输出错误。

---

## 五、文件索引

| 文件 | 作用 |
|------|------|
| [widget-bookmarks.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/widget-bookmarks.go) | Bookmarks Widget 结构体、属性继承、HTML 缓存 |
| [config-fields.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config-fields.go) | `customIconField`（图标解析）、`hslColorField`（颜色解析） |
| [bookmarks.html](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/templates/bookmarks.html) | Go 模板：分组与链接的 DOM 结构 |
| [widget-bookmarks.css](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/static/css/widget-bookmarks.css) | 分组颜色变量、图标容器样式、链接箭头 |
| [utils.css](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/static/css/utils.css) | `.dynamic-columns` 布局均衡、`.flat-icon` 暗色反色 |
| [mobile.css](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/static/css/mobile.css) | 移动端强制单列 |
| [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/widget.go) | Widget 类型注册与多态反序列化 |
| [config.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config.go) | 顶层配置加载、变量替换、热加载 |
