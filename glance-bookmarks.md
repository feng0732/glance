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

**Step 2 — 按**首个 `:` **拆分前缀与剩余部分**（`strings.Cut` 只切第一个冒号）：

```
si:github                          → prefix="si",  icon="github"
di:jellyfin.png                    → prefix="di",  icon="jellyfin.png"
/assets/my-icon.png                → 无冒号 (!found)，整串作为 URL 直接返回
https://cdn.example.com/icon.svg   → prefix="https", icon="//cdn.example.com/icon.svg"
```

> ⚠️ **注意与常见误解的差异**：`https://...` 这样的完整 URL 是**含冒号**的，不会走 `!found` 的早返回路径，而是继续执行 Step 3 和 Step 4，最终在 Step 4 的 `switch` 的 `default` 分支才回退到原始值。

**Step 3 — 扩展名识别**（从冒号后的剩余部分解析，默认 `svg`，白名单仅 `svg/png`，非法扩展名回退到 `svg`）：

此步仅对 `si:` / `di:` / `mdi:` / `sh:` 等短前缀格式有意义；对于 `https://...` 这类完整 URL，Step 3 解析出的 basename/ext 是垃圾值（如从 `"//cdn.example.com/icon.svg"` 切出 `"//cdn"` / `"example"`），但不会被使用（见 Step 4 default 分支）。

**Step 4 — 前缀映射到 CDN**：

| prefix | 图标库 | 结果 | 自动 AutoInvert |
|--------|--------|------|----------------|
| `si` | Simple Icons | 拼接 CDN URL，忽略 Step 3 结果（强制 svg） | ✅ |
| `di` | Dashboard Icons | 用 Step 3 的 basename/ext 拼接 CDN | ❌ |
| `mdi` | Material Design Icons | 拼接 CDN URL，忽略 Step 3 结果（强制 svg） | ✅ |
| `sh` | selfh.st Icons | 用 Step 3 的 basename/ext 拼接 CDN | ❌ |
| `https` / `http` / 任意其它 | 非内置前缀 | `default` 分支：**原样回退到输入原始值** | 视 `auto-invert` 前缀而定（si/mdi 以外不会自动开启） |

**两种最终直出路径的对比**：

| 输入示例 | 走哪条路径 | 为何正确 |
|----------|-----------|---------|
| `/assets/icon.svg` | `!found` 早返回（Step 2） | 本地路径不含冒号 |
| `./icon.png` | `!found` 早返回（Step 2） | 相对路径不含冒号 |
| `https://a.com/x.svg` | `default` 分支回退（Step 4） | 含冒号但前缀不匹配任何内置库 |
| `ftp://a.com/x.svg` | `default` 分支回退（Step 4） | 同上 |
| `auto-invert https://a.com/x.svg` | `default` 分支回退 + AutoInvert=true | 先剥 auto-invert 前缀，再走同上路径 |

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

### 4.1 配置文件加载完整链路

```
磁盘文件 glance.yml (+ include 文件)
    │
    ▼  parseYAMLIncludes()        [config.go#242-303]
    │  • 正则扫描 !include: / $include: 指令
    │  • 递归读取被包含文件，保留缩进
    │  • 最大递归深度 20
    │  • 收集所有涉及文件路径 → includes map
    │
    ▼  parseConfigVariables()     [config.go#142-187]
    │  • 正则扫描 ${...} 占位符
    │  • env:    读环境变量
    │  • secret: 读 /run/secrets/<name>
    │  • readFileFromEnv: 读环境变量指向的文件内容
    │
    ▼  yaml.Unmarshal → config 结构体   [config.go#103]
    │     └─ 内部触发 widgets.UnmarshalYAML()   [widget.go#95-124]
    │           ├─ ① 仅解码 type 字段到临时 struct
    │           ├─ ② newWidget("bookmarks") → &bookmarksWidget{}
    │           └─ ③ node.Decode(widget)
    │                 完整解码节点，触发自定义 UnmarshalYAML
    │                 (customIconField, hslColorField, *bool 指针等)
    │
    ▼  isConfigStateValid() 逻辑校验   [config.go#108-110]
    │     （页面、列、分组等结构层面的一致性检查）
    │
    ▼  显式 for 循环调用 initialize()   [config.go#112-126]
          ├─ 遍历 Pages[].HeadWidgets[]
          │     └─ widget.initialize()
          │           · bookmarks: 属性继承计算 (SameTab/HideArrow/Target)
          │           · bookmarks: renderTemplate() 缓存 HTML
          └─ 遍历 Pages[].Columns[].Widgets[]
                └─ widget.initialize()
```

> ⚠️ **顺序关键差异**：`widgets.UnmarshalYAML`（步骤①②③）完全发生在 `yaml.Unmarshal` 内部；而 `initialize()` 是 `yaml.Unmarshal` 结束、`isConfigStateValid` 通过之后才**单独**执行的显式循环。两者不在同一阶段，这也解释了为什么 `initialize` 中可以访问已完整反序列化的所有字段——包括依赖自定义 UnmarshalYAML 的 `SameTabRaw` 指针、`customIconField.URL` 等。

---

### 4.2 多文件包含（YAML Include）

#### 4.2.1 语法与匹配规则

定义在 [config.go#240](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config.go#L240)，正则：

```
(?m)^([ \t]*)(?:-[ \t]*)?(?:!|\$)include:[ \t]*(.+)$
```

匹配行首（`(?m)` 多行模式），支持两种等价写法：

| 写法 | 示例 |
|------|------|
| `!include:` | `  !include: ./bookmarks.yml` |
| `$include:` | `  $include: bookmarks/groups.yml` |

前缀 `-` 可选（作为数组项时），行首空白会被捕获用于保持缩进一致。

#### 4.2.2 递归解析实现

核心函数 [recursiveParseYAMLIncludes](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config.go#L246-L303)：

1. **深度保护**：超过 `CONFIG_INCLUDE_RECURSION_DEPTH_LIMIT`（20）立即报错，防止循环引用死循环
2. **路径解析**：相对路径基于**包含文件所在目录**（`mainFileDir`）解析，而非进程工作目录
3. **缩进保持**：被包含文件内容通过 [prefixStringLines](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/utils.go#L102-L110) 在每一行前补上 include 指令前的空白字符，保证 YAML 嵌套层级正确
4. **包含追踪**：所有被包含文件的绝对路径存入 `includes map[string]struct{}`，返回给调用方用于文件监听

---

### 4.3 环境变量、Secret 与文件内容注入

#### 4.3.1 占位符语法

正则定义在 [config.go#132](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config.go#L132)：

```
(^|.)\$\{(?:([a-zA-Z]+):)?([a-zA-Z0-9_-]+)\}
```

解析函数 [parseConfigVariables](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config.go#L142-L187) 逐字节替换，支持三种变量类型与转义：

| 语法 | 类型 | 行为 |
|------|------|------|
| `${API_KEY}` | `env`（默认） | 读 `os.LookupEnv("API_KEY")`，变量名需匹配 `^[A-Z0-9_]+$`，不存在则报错 |
| `${secret:api_key}` | `secret` | 读 `/run/secrets/api_key` 文件，`strings.TrimSpace` 后返回（Docker Swarm / Kubernetes secret 标准路径） |
| `${readFileFromEnv:SECRET_PATH}` | `readFileFromEnv` | 先读环境变量 `SECRET_PATH` 获取文件路径，**要求绝对路径**，再读取该文件内容 |
| `\${API_KEY}` | 转义 | 反斜杠前缀：输出字面量 `${API_KEY}`，反斜杠本身被剥离 |

若变量名不匹配 `^[A-Z0-9_]+$` 规则，函数返回原值（`returnOriginal=true`），不做替换也不报错。

#### 4.3.2 三种类型的分发逻辑

在 [parseConfigVariableOfType](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config.go#L190-L234) 的 `switch` 中：

```
variableType == "env"
  └─ os.LookupEnv(name) → 不存在报错

variableType == "secret"
  └─ os.ReadFile("/run/secrets/" + name) → TrimSpace

variableType == "readFileFromEnv"
  ├─ os.LookupEnv(name) → filePath
  ├─ 校验 filepath.IsAbs(filePath) → 非绝对路径报错
  └─ os.ReadFile(filePath)
```

所有读取失败都会通过闭包 `err` 变量向上冒泡，导致整个配置加载失败。

---

### 4.4 本地图标静态资源地址来源

图标地址存在**两套并行的静态资源系统**，来源不同，URL 路径不同：

#### 4.4.1 编译期内嵌静态资源（/static/）

实现于 [embed.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/embed.go#L20-L81)：

```go
//go:embed static
var _staticFS embed.FS

var staticFS, _ = fs.Sub(_staticFS, "static")
```

编译时 `internal/glance/static/` 目录整体打包进二进制。服务启动时计算所有内嵌文件的 MD5 哈希取前 10 位作为 `staticFSHash`，用于 HTTP 缓存击穿：

```
/static/{hash}/icons/github.svg
/static/{hash}/css/main.css
/static/{hash}/fonts/JetBrainsMono-Regular.woff2
```

路由注册在 [glance.go#459-465](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/glance.go#L459-L465)，缓存策略 `STATIC_ASSETS_CACHE_DURATION`。

提供给 Widget 使用的解析函数 [StaticAssetPath](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/glance.go#L427-L429)：

```go
func (a *application) StaticAssetPath(asset string) string {
    return a.Config.Server.BaseURL + "/static/" + staticFSHash + "/" + asset
}
```

此函数被包装为 `assetResolver` 注入到 [widgetProviders](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/widget.go#L169-L171)，供 releases 等 Widget 解析内置图标使用（bookmarks Widget 不直接使用它，bookmarks 的图标走用户自定义 URL 或 CDN）。

#### 4.4.2 用户自定义资源目录（/assets/）

用户可在配置中指定：

```yaml
server:
  assets-path: /home/user/my-glance-assets
```

实现于 [glance.go#484-489](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/glance.go#L484-L489)：

```go
if a.Config.Server.AssetsPath != "" {
    assetsFS := fileServerWithCache(http.Dir(a.Config.Server.AssetsPath), 2*time.Hour)
    mux.Handle("/assets/{path...}", http.StripPrefix("/assets/", assetsFS))
}
```

直接使用 `http.Dir` 暴露用户目录，缓存 2 小时。**bookmarks 中引用本地图标需写完整 URL 路径**，例如：

```yaml
icon: /assets/my-custom-icon.png     # 用户自定义目录
icon: si:github                       # Simple Icons CDN（见 2.2 节）
icon: https://example.com/icon.svg    # 任意远程 URL
```

注意：bookmarks 的 `customIconField` 不经过 `assetResolver`，只做前缀匹配和 URL 透传，所以 `/assets/...` 路径需用户手动写出完整前缀。

---

### 4.5 bookmarks 配置结构（YAML → Go 映射）

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
| `groups[].links[].icon` | `Link.Icon` | `customIconField` | 自定义 UnmarshalYAML（见 2.2 节） |
| `groups[].links[].same-tab` | `Link.SameTabRaw` | `*bool` | 指针区分「未设置」与「显式 false」 |
| `groups[].links[].hide-arrow` | `Link.HideArrowRaw` | `*bool` | 指针区分「未设置」与「显式 false」 |
| `groups[].links[].target` | `Link.Target` | string | 标准 |

### 4.6 hslColorField 颜色解析

定义在 [config-fields.go#25-94](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config-fields.go#L25-L94)。支持以下等价格式：

```
"hsl(230, 100%, 30%)"
"hsla(230, 100%, 30%)"
"230 100 30"
"230,100,30"
```

正则：`^(?:hsla?\()?([\d\.]+)(?: |,)+([\d\.]+)%?(?: |,)+([\d\.]+)%?\)?$`

范围校验：H ∈ [0, 360]，S ∈ [0, 100]，L ∈ [0, 100]。`String()` 方法序列化为标准 `hsl(H, S%, L%)` 格式注入 CSS 变量。

### 4.7 Widget 创建与分发

在 [widget.go#newWidget](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/widget.go#L20-L91) 的 `switch` 语句中，`"bookmarks"` case 返回 `&bookmarksWidget{}`，并分配自增 ID（`atomic.Uint64`）。

[widgets.UnmarshalYAML](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/widget.go#L95-L124) 采用**两阶段解析**：先只读 `type` 字段创建具体类型实例，再将整个 YAML 节点 `Decode` 到该实例，从而触发各字段的自定义 `UnmarshalYAML`（如 `customIconField`、`hslColorField`、`*bool` 指针）。

---

### 4.8 配置变更热重载与旧配置保留

#### 4.8.1 双入口：首次加载走哪条路径

[serveApp](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/main.go#L93-L181) 的整体结构决定了首次加载有**两条互斥入口**，取决于 `configFilesWatcher` 是否启动成功：

```
serveApp(configPath)
│
├─ ① parseYAMLIncludes(configPath)      // 仅做一次，拿到初始 contents/includes
│
├─ ② configFilesWatcher(configPath, contents, includes, onChange, onErr)
│      │
│      ├─ err == nil  → [ 正常分支 ]
│      │     ├─ defer stopWatching()       // 注册进程退出时关闭 watcher
│      │     └─ 首次加载入口：
│      │           configFilesWatcher 函数末尾 [config.go#436]
│      │           无条件执行 onChange(lastContents)
│      │           （此时 stopServer == nil，不打印"reloading"日志，直接启动 server）
│      │
│      └─ err != nil  → [ Fallback 分支 ]
│            ├─ 日志输出 watcher 启动失败
│            └─ 首次加载入口：
│                  main.go#L163-L177 显式调用
│                  newConfigFromYAML → newApplication → app.server() → startServer()
│                  （不经过 onChange 回调，hadValidConfigOnStartup 永远保持 false）
│
└─ ③ <-exitChannel  // 阻塞等待退出信号
```

> ⚠️ **常见误解**：正常分支下，`configFilesWatcher` 返回成功并不意味着「还要在 main.go 里再调一次加载」——首次加载已经在 `configFilesWatcher` 内部作为最后一步同步执行了（[config.go#436](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config.go#L436)）。`onChange(lastContents)` 被调用时 `stopServer == nil`，因此跳过 `stopServer()` 调用和 `"Config file changed, reloading..."` 日志，直接走首次启动逻辑。

#### 4.8.2 watcher 内部事件处理

[configFilesWatcher](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config.go#L305-L445) 启动一个独立 goroutine 监听 fsnotify：

| 事件 | 处理 |
|------|------|
| `fsnotify.Write` | 触发 debounce（500ms，200ms 滑动窗口）→ 重新 parseYAMLIncludes → 内容比对 |
| `fsnotify.Rename` | Linux 下 rename 后文件不再被 watch：等待最多 2 秒（10 × 200ms）轮询看文件是否重新出现，然后走完整比对 |
| `fsnotify.Remove` | 从 includes 中移除，走完整比对 |
| `fsnotify.Chmod` | 忽略 |
| watcher.Errors | 传 onErr 回调输出日志，不中断服务 |

每次变更后 [parseAndCompareBeforeCallback](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config.go#L349-L371) 做三件事：
1. 重新 `parseYAMLIncludes` 得到 `currentContents` 与 `currentIncludes`
2. 比对 includes set：有增删则调用 `watcher.Add / watcher.Remove` 动态更新监听列表
3. 比对字节内容 `bytes.Equal(lastContents, currentContents)`：不同才触发 `onChange`

所有状态操作（`lastContents`、`lastIncludes`、`debounceTimer`）受 `sync.Mutex` 保护，避免多 goroutine 竞态。

#### 4.8.3 onChange 回调：旧配置保留的精确触发路径

[main.go#onChange](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/main.go#L101-L146) 的完整决策树：

```
onChange(newContents []byte)
│
│  入口判断：
│  stopServer != nil ? ──是──→ 打印 "Config file changed, reloading..."
│       │否
│       └──── （首次加载，不打印任何 reload 日志）
│
├─ [失败点 A] newConfigFromYAML(newContents)
│     │                                   ↑
│     │成功                               │  parseConfigVariables 变量替换失败
│     │                                   │  yaml.Unmarshal 反序列化失败
│     │                                   │  isConfigStateValid 结构校验失败
│     │                                   │  任一 widget.initialize() 失败
│     │
│     └─ 失败 → 日志 "Config has errors: %v"
│              │
│              ├─ hadValidConfigOnStartup == false
│              │   └─ close(exitChannel) → 进程退出（无旧配置可保留）
│              │
│              └─ hadValidConfigOnStartup == true
│                  └─ return   ◀── 【旧配置保留触发点 ①】
│                              不执行后续任何代码，旧 app/server 完好
│
├─ [失败点 B] newApplication(config)
│     │
│     └─ 失败 → 日志 "Failed to create application: %v"
│              同上分支：首次失败退出，非首次 return
│                       ◀── 【旧配置保留触发点 ②】
│
├─ hadValidConfigOnStartup = true   ←  A、B 都通过后才置位
│
├─ stopServer != nil ? ──是──→ stopServer()   // 停旧 HTTP server（可能 err，仅日志）
│       │否
│       └──── （首次加载，无需停旧 server）
│
└─ go func() {
       startServer, stopServer = app.server()   // 替换闭包
       startServer()  // 启动新 server，失败仅日志
   }()
```

**旧配置保留的精确触发条件**（需同时满足）：

1. 进程历史上**至少有一次** A + B 全部通过（`hadValidConfigOnStartup == true`）
2. 本次变更后 A 或 B **任一失败**
3. **不会**触发 `stopServer()`，也不会触碰 `app` 和 `stopServer` 闭包变量
4. 旧 server 在原有的 goroutine 中继续监听原端口，完全不受影响

**不会触发保留、直接退出的情形**：

- 进程首次启动（`hadValidConfigOnStartup == false`）且 A 或 B 失败
- 这只会发生在正常分支的首次 `onChange(lastContents)` 调用期间，因为一旦成功过一次 flag 就会被置 true

**Fallback 分支的特殊情况**：

watcher 启动失败走 Fallback 时，代码不经过 `onChange`，所以 `hadValidConfigOnStartup` 永远保持 `false`。但此时也不存在后续的 reload 调用机会，保留机制自然不会被触发——配置失败时直接 `return fmt.Errorf(...)` 退出进程。

---

## 五、文件索引

| 文件 | 作用 |
|------|------|
| [widget-bookmarks.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/widget-bookmarks.go) | Bookmarks Widget 结构体、属性继承、HTML 缓存 |
| [config-fields.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config-fields.go) | `customIconField`（图标前缀解析）、`hslColorField`（颜色解析） |
| [bookmarks.html](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/templates/bookmarks.html) | Go 模板：分组与链接的 DOM 结构 |
| [widget-bookmarks.css](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/static/css/widget-bookmarks.css) | 分组颜色变量、图标容器样式、链接箭头 |
| [utils.css](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/static/css/utils.css) | `.dynamic-columns` 布局均衡、`.flat-icon` 暗色反色 |
| [mobile.css](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/static/css/mobile.css) | 移动端强制单列 |
| [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/widget.go) | Widget 类型注册、多态反序列化、`widgetProviders`（assetResolver） |
| [config.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/config.go) | `parseYAMLIncludes`（多文件包含）、`parseConfigVariables`（env/secret/readFileFromEnv 注入）、`configFilesWatcher`（fsnotify 热重载） |
| [main.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/main.go) | `onChange` 回调（旧配置保留逻辑、stop/start server 切换） |
| [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/glance.go) | `StaticAssetPath`（内嵌资源）、`/assets/` 用户自定义资源路由 |
| [embed.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/embed.go) | `//go:embed static` 二进制内嵌、`staticFSHash` 计算 |
| [utils.go](file:///d:/fz/0601/solo-dogfeeding/code/143-glance/internal/glance/utils.go) | `prefixStringLines`（include 缩进保持） |
