# Glance 主题系统深度分析

## 概述

Glance 的主题系统采用 **"配置 → Go 结构体 → CSS 变量模板 → 浏览器级联"** 的四阶段管道。颜色值从 YAML 配置文件进入，经过后端结构体解析、模板渲染生成 CSS 自定义属性（CSS Variables），最终在浏览器中通过多层级联和计算派生出整套配色方案。

---

## 一、配置解析阶段（YAML → Go 结构体）

### 1.1 配置入口

配置结构在 [config.go](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/config.go#L48-L54) 中定义：

```go
Theme struct {
    themeProperties `yaml:",inline"`       // 内联展开，颜色等属性直接在 theme: 下
    CustomCSSFile   string `yaml:"custom-css-file"`
    DisablePicker   bool                                     `yaml:"disable-picker"`
    Presets         orderedYAMLMap[string, *themeProperties] `yaml:"presets"`
} `yaml:"theme"`
```

`yaml:",inline"` 标签使得 `themeProperties` 的字段可以直接写在 `theme:` 层级下，而不需要额外嵌套。

### 1.2 颜色字段类型

主题颜色使用自定义的 `hslColorField` 类型，在 [config-fields.go](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/config-fields.go#L25-L94) 中实现：

```go
type hslColorField struct {
    H float64  // Hue (色相) 0-360
    S float64  // Saturation (饱和度) 0-100
    L float64  // Lightness (亮度) 0-100
}
```

**YAML 解析支持的格式**（通过正则 `^(?:hsla?\()?([\d\.]+)(?: |,)+([\d\.]+)%?(?: |,)+([\d\.]+)%?\)?$` 匹配）：
- `240 8 9` —— 空格分隔
- `240, 8%, 9%` —— 逗号分隔，可选百分比符号
- `hsl(240, 8%, 9%)` —— 标准 CSS hsl() 格式

### 1.3 themeProperties 结构体

在 [theme.go](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L41-L54) 中定义完整的主题属性：

| 字段 | 类型 | YAML Key | 说明 |
|------|------|----------|------|
| `BackgroundColor` | `*hslColorField` | `background-color` | 背景色（HSL） |
| `PrimaryColor` | `*hslColorField` | `primary-color` | 主色（强调色） |
| `PositiveColor` | `*hslColorField` | `positive-color` | 正向状态色（成功、上升） |
| `NegativeColor` | `*hslColorField` | `negative-color` | 负向状态色（错误、下降） |
| `Light` | `bool` | `light` | 是否为亮色模式 |
| `ContrastMultiplier` | `float32` | `contrast-multiplier` | 对比度乘数（影响文字与背景对比度） |
| `TextSaturationMultiplier` | `float32` | `text-saturation-multiplier` | 文字饱和度乘数 |

指针类型（`*hslColorField`）的意义：**nil 表示未配置，将使用 CSS 默认值**。这样可以实现部分覆盖——用户只需在 YAML 中写需要修改的字段。

---

## 二、主题初始化阶段（Go 结构体 → CSS 字符串）

### 2.1 初始化时机

在 [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/glance.go#L101-L141) 的 `newApplication()` 函数中完成：

```go
// 内置两个默认预设（如果没有被禁用）
// default-dark: 空结构体 → 全部使用 CSS 默认值
// default-light: 预定义的亮色配置
if !config.Theme.DisablePicker {
    // ... 创建 default-dark 和 default-light 预设
    config.Theme.Presets = *themePresets.Merge(&config.Theme.Presets)

    // 初始化每个预设主题
    for key, properties := range config.Theme.Presets.Items() {
        properties.Key = key
        properties.init()  // ← 关键调用
    }
}

// 初始化用户配置的默认主题
config.Theme.Key = "default"
config.Theme.init()  // ← 关键调用
```

### 2.2 init() 方法

在 [theme.go](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L56-L76) 中：

```go
func (t *themeProperties) init() error {
    // 1. 执行 theme-style.gotmpl 模板，生成 CSS 变量定义
    css, err := executeTemplateToString(themeStyleTemplate, t)
    t.CSS = template.CSS(whitespaceAtBeginningOfLinePattern.ReplaceAllString(css, ""))

    // 2. 生成主题预览 HTML（用于主题选择器 UI）
    previewHTML, err := executeTemplateToString(themePresetPreviewTemplate, t)
    t.PreviewHTML = template.HTML(previewHTML)

    // 3. 背景色转 HEX（用于 PWA manifest 的 theme-color）
    t.BackgroundColorAsHex = t.BackgroundColor != nil ? t.BackgroundColor.ToHex() : "#151519"

    return nil
}
```

### 2.3 CSS 变量注入模板

[theme-style.gotmpl](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/theme-style.gotmpl) 是核心模板：

```css
:root {
    {{ if .BackgroundColor }}
    --bgh: {{ .BackgroundColor.H }};
    --bgs: {{ .BackgroundColor.S }}%;
    --bgl: {{ .BackgroundColor.L }}%;
    {{ end }}
    {{ if ne 0.0 .ContrastMultiplier }}--cm: {{ .ContrastMultiplier }};{{ end }}
    {{ if ne 0.0 .TextSaturationMultiplier }}--tsm: {{ .TextSaturationMultiplier }};{{ end }}
    {{ if .PrimaryColor }}--color-primary: {{ .PrimaryColor.String | safeCSS }};{{ end }}
    {{ if .PositiveColor }}--color-positive: {{ .PositiveColor.String | safeCSS }};{{ end }}
    {{ if .NegativeColor }}--color-negative: {{ .NegativeColor.String | safeCSS }};{{ end }}
}
```

**关键设计**：模板使用条件判断——只有非 nil/非零值的字段才会被输出到 CSS。未配置的字段将 **完全不出现** 在 `:root` 中，让浏览器回退到 main.css 的默认值。

例如，如果用户只配置了 `primary-color`，生成的 CSS 仅包含：
```css
:root {
    --color-primary: hsl(43, 50%, 70%);
}
```
其他变量（`--bgh`, `--bgs`, `--bgl`, `--cm` 等）仍使用 main.css 中的默认值。

---

## 三、CSS 打包与默认值阶段

### 3.1 CSS 运行时打包

Glance 在 [embed.go](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/embed.go#L87-L151) 中运行时递归解析 `@import`，将所有 CSS 合并为单一的 `bundledCSSContents`：

```go
var bundledCSSContents = func() []byte {
    // 从 css/main.css 开始，递归解析 @import
    contents, err := recursiveParseImports("css/main.css", 0)
    // 去除注释、行首空白、换行，压缩输出
    contents = cssSingleLineCommentPattern.ReplaceAll(contents, nil)
    contents = whitespaceAtBeginningOfLinePattern.ReplaceAll(contents, nil)
    contents = bytes.ReplaceAll(contents, []byte("\n"), []byte(""))
    return contents
}()
```

main.css 的导入顺序（[main.css](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/css/main.css#L60-L66)）：
```css
@import "site.css";       /* 亮暗模式切换 + 页面布局 */
@import "widgets.css";    /* 所有 widget 样式 */
@import "popover.css";    /* 弹出层 */
@import "utils.css";      /* 工具类 */
@import "mobile.css";     /* 移动端适配 */
```

### 3.2 CSS 变量默认值定义

所有 CSS 变量的 **默认值** 在 [main.css](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/css/main.css#L9-L58) 的 `:root` 中定义：

```css
:root {
    /* === 核心基础变量（可被主题覆盖） === */
    --scheme: ;          /* 亮暗模式操作符，暗色为空，亮色为 "100% -" */
    --bgh: 240;          /* Background Hue - 背景色相 */
    --bgs: 8%;           /* Background Saturation - 背景饱和度 */
    --bgl: 9%;           /* Background Lightness - 背景亮度 */
    --bghs: var(--bgh), var(--bgs);  /* 便捷组合变量 */
    --cm: 1;             /* Contrast Multiplier - 对比度乘数 */
    --tsm: 1;            /* Text Saturation Multiplier - 文字饱和度乘数 */

    /* === 语义化颜色（可被主题覆盖） === */
    --color-primary: hsl(43, 50%, 70%);
    --color-positive: var(--color-primary);
    --color-negative: hsl(0, 70%, 70%);

    /* === 派生计算变量（由基础变量计算得出，不可直接配置） === */
    --color-background: hsl(var(--bghs), var(--bgl));
    --color-widget-background: hsl(var(--bghs), calc(var(--bgl) + 1%));
    --color-widget-content-border: hsl(var(--bghs), calc(var(--scheme) (var(--scheme) var(--bgl) + 4%)));

    /* 文字颜色派生 */
    --ths: var(--bgh), calc(var(--bgs) * var(--tsm));
    --color-text-highlight: hsl(var(--ths), calc(var(--scheme) var(--cm) * 85%));
    --color-text-paragraph: hsl(var(--ths), calc(var(--scheme) var(--cm) * 73%));
    --color-text-base:      hsl(var(--ths), calc(var(--scheme) var(--cm) * 58%));
    --color-text-base-muted:hsl(var(--ths), calc(var(--scheme) var(--cm) * 52%));
    --color-text-subdue:    hsl(var(--ths), calc(var(--scheme) var(--cm) * 35%));
}
```

### 3.3 派生颜色的计算逻辑

所有派生颜色都基于 **背景色（--bgh/--bgs/--bgl）** 进行数学计算，而不是硬编码。这种设计的好处：

1. **配色一致性**——所有颜色都来自同一色相，天然和谐
2. **亮暗模式自动适配**——通过 `--scheme` 变量反转亮度
3. **对比度可调节**——通过 `--cm` 变量统一控制文字清晰度

#### 派生变量解析：

| 变量 | 计算方式 | 用途 |
|------|----------|------|
| `--color-widget-background` | `bgl + 1%` | Widget 背景比页面背景稍亮 |
| `--color-separator` | `bgl ± 4% * cm` | 分隔线 |
| `--color-text-highlight` | `bgs * tsm, ± cm * 85%` | 高亮文字（标题） |
| `--color-text-base` | `bgs * tsm, ± cm * 58%` | 正文文字 |
| `--color-text-subdue` | `bgs * tsm, ± cm * 35%` | 弱化文字（次要信息） |

---

## 四、亮暗模式切换机制

### 4.1 --scheme 变量

这是 Glance 亮暗模式最巧妙的设计。在 [site.css](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/css/site.css#L1-L3) 中：

```css
:root[data-scheme=light] {
    --scheme: 100% -;
}
```

暗色模式下，`--scheme` 为空字符串 `;`
亮色模式下，`--scheme` 为 `100% -;`

### 4.2 亮度反转原理

在 CSS `calc()` 中，`--scheme` 被作为 **操作符片段** 注入表达式：

**暗色模式（背景亮度 9%，文字需要更亮）：**
```
calc(var(--scheme) var(--cm) * 58%)
→ calc( 1 * 58%)
→ 58%  ← 文字亮度 58%，比背景 9% 亮得多
```

**亮色模式（背景亮度 95%，文字需要更暗）：**
```
calc(var(--scheme) var(--cm) * 58%)
→ calc(100% - 1 * 58%)
→ 42%  ← 文字亮度 42%，比背景 95% 暗得多
```

这种"注入操作符"的技巧用极少的代码实现了亮暗两套配色。

**注意双重 `var(--scheme)` 模式**：某些表达式用了两次 `--scheme`，如：
```
calc(var(--scheme) (var(--scheme) var(--bgl) + 4%))
```
- 暗色：`( 9% + 4%) = 13%`（比背景亮）
- 亮色：`100% - (100% - 95% + 4%) = 100% - 9% = 91%`（比背景暗）

确保无论明暗，widget 边框都相对于背景有正确的方向差。

### 4.3 data-scheme 属性设置

在 [document.html](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L2)：

```html
<html lang="en" id="top" data-theme="{{ .Request.Theme.Key }}" data-scheme="{{ if .Request.Theme.Light }}light{{ else }}dark{{ end }}">
```

后端根据主题的 `Light` 布尔值输出 `data-scheme="light"` 或 `data-scheme="dark"`，CSS 通过属性选择器匹配。

---

## 五、HTML 注入与级联顺序

### 5.1 页面渲染中的主题选择

在 [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/glance.go#L290-L304) 的 `populateTemplateRequestData()` 中：

```go
func (a *application) populateTemplateRequestData(data *templateRequestData, r *http.Request) {
    theme := &a.Config.Theme.themeProperties  // 默认主题

    if !a.Config.Theme.DisablePicker {
        selectedTheme, err := r.Cookie("theme")
        if err == nil {
            preset, exists := a.Config.Theme.Presets.Get(selectedTheme.Value)
            if exists {
                theme = preset  // 用户选择了预设主题
            }
        }
    }

    data.Theme = theme
}
```

优先级：**Cookie 中的预设主题 > 配置文件中的默认主题**

### 5.2 四级样式级联

在 [document.html](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L25-L29) 中按以下顺序加载样式：

```html
<!-- 第 1 层：打包的基础 CSS（含默认变量值 + 所有布局/组件样式） -->
<link rel="stylesheet" href='{{ .App.StaticAssetPath "css/bundle.css" }}'>

<!-- 第 2 层：内联主题样式（覆盖默认 CSS 变量） -->
<style id="theme-style">{{ .Request.Theme.CSS }}</style>

<!-- 第 3 层：用户自定义 CSS 文件（可选，最后加载） -->
{{ if .App.Config.Theme.CustomCSSFile }}
<link rel="stylesheet" href="{{ .App.Config.Theme.CustomCSSFile }}?v={{ .App.CreatedAt.Unix }}">
{{ end }}

<!-- 第 4 层：document.head 配置的任意 HTML（可包含额外 <style>） -->
{{ if .App.Config.Document.Head }}{{ .App.Config.Document.Head }}{{ end }}
```

**级联优先级（从高到低）**：

| 层级 | 来源 | 说明 |
|------|------|------|
| 4 | `document.head` 配置 | 用户在 YAML 的 `document.head:` 中写的任何 `<style>` |
| 3 | `custom-css-file` | 用户指定的外部 CSS 文件 |
| 2 | `<style id="theme-style">` | 后端根据 YAML 主题配置渲染的 CSS 变量 |
| 1 | `bundle.css` | 内置默认 CSS（变量默认值 + 所有样式规则） |

浏览器 CSS 级联规则：**后加载的同名变量覆盖先加载的**。由于所有层都使用 `:root` 选择器，特异性相同，因此加载顺序决定了最终值。

---

## 六、前端动态切换主题

### 6.1 切换流程

在 [page.js](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/js/page.js#L670-L692) 的 `changeTheme()` 函数中：

```javascript
async function changeTheme(key, onChanged) {
    const themeStyleElem = find("#theme-style");

    // 1. POST 请求后端获取新主题的 CSS 和 Scheme 信息
    const response = await fetch(`${pageData.baseURL}/api/set-theme/${key}`, {
        method: "POST",
    });
    const newThemeStyle = await response.text();  // CSS 内容
    const scheme = response.headers.get("X-Scheme");  // "light" 或 "dark"

    // 2. 临时禁用所有过渡动画，避免切换时的闪烁
    const tempStyle = elem("style")
        .html("* { transition: none !important; }")
        .appendTo(document.head);

    // 3. 替换内联样式和 HTML 属性
    themeStyleElem.html(newThemeStyle);
    document.documentElement.setAttribute("data-theme", key);
    document.documentElement.setAttribute("data-scheme", scheme);

    // 4. 10ms 后移除临时样式，恢复动画
    setTimeout(() => { tempStyle.remove(); }, 10);
}
```

### 6.2 后端主题切换接口

在 [theme.go](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L15-L39)：

```go
func (a *application) handleThemeChangeRequest(w http.ResponseWriter, r *http.Request) {
    themeKey := r.PathValue("key")

    // 获取预设主题属性，"default" 使用配置中的默认主题
    properties, exists := a.Config.Theme.Presets.Get(themeKey)
    if !exists && themeKey != "default" {
        w.WriteHeader(http.StatusNotFound)
        return
    }
    if themeKey == "default" {
        properties = &a.Config.Theme.themeProperties
    }

    // 1. 设置 Cookie（2 年过期），下次刷新页面时会读取
    http.SetCookie(w, &http.Cookie{
        Name:     "theme",
        Value:    themeKey,
        Path:     a.Config.Server.BaseURL + "/",
        SameSite: http.SameSiteLaxMode,
        Expires:  time.Now().Add(2 * 365 * 24 * time.Hour),
    })

    // 2. 返回 CSS 内容
    w.Header().Set("Content-Type", "text/css")
    w.Header().Set("X-Scheme", ternary(properties.Light, "light", "dark"))
    w.Write([]byte(properties.CSS))
}
```

---

## 七、完整数据流图示

```
┌─────────────────────────────────────────────────────────────────┐
│                     glance.yml 配置文件                          │
│  theme:                                                          │
│    background-color: 240 8 9                                     │
│    primary-color: hsl(43, 50%, 70%)                              │
│    light: false                                                  │
│    contrast-multiplier: 1.2                                      │
│    presets:                                                      │
│      teal-city: { ... }                                          │
└────────────────────────┬────────────────────────────────────────┘
                         │ yaml.Unmarshal
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│         Go: themeProperties 结构体                               │
│  BackgroundColor: &hslColorField{H:240, S:8, L:9}               │
│  PrimaryColor: &hslColorField{H:43, S:50, L:70}                 │
│  Light: false, ContrastMultiplier: 1.2, ...                     │
└────────────────────────┬────────────────────────────────────────┘
                         │ init() → 执行 theme-style.gotmpl
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│        渲染后的 CSS 字符串（存入 .CSS 字段）                      │
│  :root {                                                         │
│    --bgh: 240;                                                   │
│    --bgs: 8%;                                                    │
│    --bgl: 9%;                                                    │
│    --cm: 1.2;                                                    │
│    --color-primary: hsl(43.0, 50.0%, 70.0%);                    │
│  }                                                               │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTP 响应 / HTML 模板注入
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      浏览器中的级联顺序                           │
│                                                                 │
│  ① bundle.css (:root 默认变量)    ← 优先级最低                   │
│     --bgh: 240; --bgl: 9%; --cm: 1; ...                          │
│                                                                 │
│  ② <style id="theme-style">       ← 覆盖配置过的变量             │
│     --bgh: 240; --bgl: 9%; --cm: 1.2; --color-primary: ...      │
│                                                                 │
│  ③ <link custom-css-file>         ← 用户自定义样式               │
│                                                                 │
│  ④ document.head <style>         ← 优先级最高                   │
└────────────────────────┬────────────────────────────────────────┘
                         │ CSS calc() 运行时计算
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                 最终可用的 CSS 变量                               │
│  --color-text-base: hsl(240, 8%, calc(1 * 1.2 * 58%))           │
│                    = hsl(240, 8%, 69.6%)                        │
│                                                                 │
│  --color-widget-background: hsl(240, 8%, 10%)                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 八、关键设计要点总结

### 8.1 部分覆盖机制

通过 **指针类型 + 条件模板输出** 实现。用户在 YAML 中只需写需要自定义的字段，未配置的字段不会出现在 `<style id="theme-style">` 中，因此 bundle.css 中的默认值仍然生效。这避免了"全量覆盖"带来的配置冗余。

### 8.2 HSL 色彩空间

所有颜色计算基于 HSL（色相-饱和度-亮度）而非 RGB，因为：
- 调整亮度只需修改 L 通道
- 调整对比度只需对 L 做乘法
- 亮暗模式反转通过 `100% - L` 实现
- 保持色相 H 不变即可保证配色和谐

### 8.3 Scheme 操作符注入

`--scheme` 变量将亮暗模式的差异压缩为一个 CSS 变量（空 或 `100% -`），配合 `calc()` 实现零 JavaScript 的亮暗切换。这是整个主题系统最精巧的部分。

### 8.4 Cookie + 内联样式

首次页面加载时通过后端模板直接注入主题（避免 FOUC 闪烁），后续动态切换通过替换 `<style>` 内容和 Cookie 持久化，兼顾了首屏性能和交互体验。

### 8.5 四级样式覆盖

内置样式 → 主题配置 → 自定义 CSS 文件 → document.head 内联样式。每一层都可以覆盖上一层的变量或规则，为用户提供了从简单配置到完全自定义的渐进式能力。
