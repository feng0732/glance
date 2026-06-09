# Glance 主题系统：源码可证实行为 vs 保守推断边界

## 文档目标

将前两篇分析中的结论严格分为两类：
1. **源码可证实**：有明确代码行、表达式、数据结构直接支撑
2. **保守推断**：代码无直接注释或显式逻辑，只能从结构和行为间接推理

所有结论均标注对应的源码文件与行号。

---

## 一、default-dark / default-light 显示条件

### 1.1 源码定位

核心逻辑位于 [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/glance.go#L104-L128)：

```go
if !config.Theme.DisablePicker {
    themeKeys := make([]string, 0, 2)
    themeProps := make([]*themeProperties, 0, 2)

    // default-dark 的条件注册
    defaultDarkTheme, ok := config.Theme.Presets.Get("default-dark")
    if ok && !config.Theme.SameAs(defaultDarkTheme) || !config.Theme.SameAs(&themeProperties{}) {
        themeKeys = append(themeKeys, "default-dark")
        themeProps = append(themeProps, &themeProperties{})
    }

    // default-light 的无条件注册
    themeKeys = append(themeKeys, "default-light")
    themeProps = append(themeProps, &themeProperties{
        Light:                    true,
        BackgroundColor:          &hslColorField{240, 13, 95},
        PrimaryColor:             &hslColorField{230, 100, 30},
        NegativeColor:            &hslColorField{0, 70, 50},
        ContrastMultiplier:       1.3,
        TextSaturationMultiplier: 0.5,
    })

    themePresets, err := newOrderedYAMLMap(themeKeys, themeProps)
    config.Theme.Presets = *themePresets.Merge(&config.Theme.Presets)
    // ...
}
```

辅助函数 [theme.go SameAs()](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L78-L107) 和 [config-fields.go hslColorField.SameAs()](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/config-fields.go#L39-L47)。

### 1.2 源码可证实：default-dark 的条件表达式拆解

条件表达式（Go 运算符优先级：`&&` 高于 `||`）：

```
ok && !config.Theme.SameAs(defaultDarkTheme) || !config.Theme.SameAs(&themeProperties{})
```

等价于：

```
(ok && !config.Theme.SameAs(defaultDarkTheme)) || (!config.Theme.SameAs(&themeProperties{}))
```

其中：
- `ok` = 用户 YAML 的 `theme.presets` 中是否存在 key 为 `"default-dark"` 的预设（[config.go Get()](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/config.go#L578-L581)）
- `&themeProperties{}` = 空主题结构体（所有指针 nil，所有数值零值）

### 1.3 源码可证实：`themeProperties{}` 空结构体的字段值

根据 Go 语言零值规则和 [theme.go](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L41-L54) 的结构体定义：

| 字段 | Go 零值 | SameAs 比较含义 |
|------|---------|----------------|
| `BackgroundColor` | `nil` | 未配置背景色 |
| `PrimaryColor` | `nil` | 未配置主色 |
| `PositiveColor` | `nil` | 未配置正向色 |
| `NegativeColor` | `nil` | 未配置负向色 |
| `Light` | `false` | 暗色模式 |
| `ContrastMultiplier` | `0.0` | 未配置对比度乘数 |
| `TextSaturationMultiplier` | `0.0` | 未配置文字饱和度乘数 |

### 1.4 源码可证实：default-dark 真值表

| 场景 | `ok` | `!SameAs(用户默认主题, 用户default-dark预设)` | `!SameAs(用户默认主题, 空结构体)` | 表达式结果 | default-dark 是否显示 |
|------|------|--------------------------------------------|-------------------------------|-----------|---------------------|
| A. 用户 YAML 无任何 theme 字段，无 presets | false | （`&&` 短路，不计算） | `false`（空 == 空） | **false** | 不显示 |
| B. 用户配置了 theme: { primary-color: ... }，无 presets | false | （`&&` 短路，不计算） | `true`（非空 != 空） | **true** | 显示 |
| C. 用户定义了 presets.default-dark，且默认主题与它不同 | true | `true`（不同） | （`||` 短路，不计算） | **true** | 显示 |
| D. 用户定义了 presets.default-dark，且默认主题就是它 | true | `false`（相同） | 取决于默认主题是否为空（一般为 false） | **false** | 不显示 |

### 1.5 源码可证实：default-light 的行为

- **无条件注册**：[glance.go#L114-L122](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/glance.go#L114-L122) 中没有任何 `if` 判断，直接 `append`
- **注册顺序**：先处理 default-dark（条件满足才加入），再加入 default-light
- **显示顺序**：default-dark → default-light → 用户自定义预设（按 YAML 中定义顺序），由 [config.go Merge()](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/config.go#L583-L600) 决定

### 1.6 源码可证实：Merge 后的覆盖规则

[config.go#L583-L600](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/config.go#L583-L600)：

```go
merged.keys = append(merged.keys, self.keys...)  // self = 内置预设（先）
maps.Copy(merged.data, self.data)                 // 内置预设数据先拷贝

for _, key := range other.keys {                  // other = 用户预设
    if _, exists := self.data[key]; !exists {
        merged.keys = append(merged.keys, key)    // 新 key 追加到 keys 末尾
    }
}
maps.Copy(merged.data, other.data)                // 用户预设后拷贝 → 覆盖同名 key
```

结论：
- 用户预设可以用同名 key（如 `default-dark`）**完全替换**内置预设的数据
- 但内置预设在 `keys` 中的**位置不变**（用户定义的同名预设不会追加新位置，只是替换 data）

### 1.7 保守推断：default-dark 条件注册的设计意图

代码中没有任何注释说明。只能保守推断：

- **推断 1**：场景 A（用户完全未配置主题）时不显示 default-dark，因为默认渲染结果就是 default-dark 的效果，显示它没意义
- **推断 2**：场景 D（用户预设了自己的 default-dark 且默认主题就是它）时不显示，避免选择器中出现两个视觉效果完全相同的选项
- **推断 3**：`ok && !SameAs(用户默认, 用户default-dark预设)` 分支处理的是"用户定义了自定义 default-dark 覆盖，但默认主题不是它"的边缘场景，此时仍需要显示该选项供用户切换

> ⚠️ **保守推断声明**：以上三条均无源码注释或测试直接支撑，仅从条件表达式反推。

---

## 二、主题选择器渲染路径

### 2.1 源码可证实：移动端预设全量渲染

[page.html#L72-L79](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/page.html#L72-L79)：

```html
<div data-popover-html>
    <div class="theme-choices">
        {{ .App.Config.Theme.PreviewHTML }}       <!-- 默认主题预览 -->
        {{ range $_, $preset := .App.Config.Theme.Presets.Items }}
        {{ $preset.PreviewHTML }}                   <!-- 每个预设预览 -->
        {{ end }}
    </div>
</div>
```

渲染顺序：**默认主题预览 → Presets.Items() 中的所有预设**（即 default-dark? → default-light → 用户预设）

### 2.2 源码可证实：桌面端预设初始为空

[page.html#L36-L44](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/page.html#L36-L44)：

```html
<div data-popover-html>
    <div class="theme-choices"></div>  <!-- 空！ -->
</div>
```

### 2.3 源码可证实：JS 克隆自移动端到桌面端

[page.js#L694-L704](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/js/page.js#L694-L704)：

```javascript
function initThemePicker() {
    const themeChoicesInMobileNav = find(".mobile-navigation .theme-choices");
    const themeChoicesInHeader = find(".header-container .theme-choices");

    if (themeChoicesInHeader) {
        themeChoicesInHeader.replaceWith(
            themeChoicesInMobileNav.cloneNode(true)  // 深克隆
        );
    }
    // ...
}
```

### 2.4 源码可证实：Preset 预览按钮 HTML 生成

[theme-preset-preview.html](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/theme-preset-preview.html) 中的关键行为：

- `PositiveColor` 未配置时**继承** `PrimaryColor`：
  ```html
  {{- if .PrimaryColor }}
      {{- $primary = .PrimaryColor.String | safeCSS }}
      {{- if not .PositiveColor }}
          {{- $positive = $primary }}
      {{- else }}
          {{- $positive = .PositiveColor.String | safeCSS }}
      {{- end }}
  {{- end }}
  ```
- 亮色主题按钮加 `theme-preset-light` class：
  ```html
  <button class="theme-preset{{ if .Light }} theme-preset-light{{ end }}" ...>
  ```
- 每个按钮携带 `data-key` 属性供 JS 读取：
  ```html
  <button ... data-key="{{ .Key }}">
  ```

---

## 三、`<meta name="color-scheme">`：源码可证实 vs 保守推断

### 3.1 源码可证实的行为

**证据 1 — 模板硬编码**：[document.html#L15](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L15)

```html
<meta name="color-scheme" content="dark">
```

这是字面量字符串 `dark`，没有任何 Go 模板变量。

**证据 2 — 无任何动态修改代码**：在整个代码库中搜索 `color-scheme`，仅这一处出现（之前的 grep 结果已证实）：

- `page.js changeTheme()` 不更新此 meta
- `handleThemeChangeRequest()` 不返回相关 header
- Go 后端无任何地方读取或修改此值

**证据 3 — `apple-mobile-web-app-status-bar-style` 同样硬编码**：[document.html#L19](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L19)

```html
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
```

也是字面量字符串，无模板变量。

### 3.2 源码可证实的行为结论

| 行为 | 源码证据 |
|------|---------|
| color-scheme 值**恒为 "dark"** | [document.html#L15](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L15) 字面量 |
| 切换主题时**不会改变** color-scheme | [page.js#L670-L692](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/js/page.js#L670-L692) changeTheme() 中无相关代码 |
| 刷新页面也**不会改变** color-scheme | 后端模板中无条件判断 |

### 3.3 保守推断：为何硬编码为 dark

代码中无任何注释说明。可能的原因（按可能性排序）：

1. **浏览器原生控件与自定义样式冲突**：Glance 对滚动条、表单元素有大量自定义 CSS（`background: none; border: 0` 等），如果 color-scheme 跟随主题变为 light，浏览器可能在某些控件上强制应用亮色样式，与自定义样式冲突
2. **简化动态切换**：color-scheme 变更会触发浏览器对原生 UI 的重绘（尤其是滚动条），可能与切换主题时的过渡动画产生冲突或闪烁
3. **历史遗留**：早期版本只支持暗色主题，添加亮色主题时遗漏了更新 color-scheme

> ⚠️ **保守推断声明**：以上三条均为推测。源码中没有注释、commit message（本分析不涉及 git 历史）或测试能够证明真正原因。

---

## 四、`<meta name="theme-color">`：源码可证实 vs 保守推断

### 4.1 源码可证实的行为

**证据 1 — 页面初次渲染时动态取值**：[document.html#L21](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L21)

```html
<meta name="theme-color" content="{{ .Request.Theme.BackgroundColorAsHex }}">
```

**证据 2 — BackgroundColorAsHex 的计算逻辑**：[theme.go#L69-L73](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L69-L73)

```go
if t.BackgroundColor != nil {
    t.BackgroundColorAsHex = t.BackgroundColor.ToHex()   // 用户配置了背景色 → 转换为 HEX
} else {
    t.BackgroundColorAsHex = "#151519"                    // 未配置 → 硬编码默认值
}
```

**证据 3 — 前端切换主题时不更新**：[page.js#L670-L692](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/js/page.js#L670-L692)

`changeTheme()` 函数只做了三件事：
1. 替换 `<style id="theme-style">` 的内容（L687）
2. 更新 `<html data-theme>` 属性（L688）
3. 更新 `<html data-scheme>` 属性（L689）

**没有任何代码更新 `<meta name="theme-color">`**。

**证据 4 — handleThemeChangeRequest 不返回背景色**：[theme.go#L15-L38](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L15-L38)

后端接口只返回：
- `Content-Type: text/css`
- `X-Scheme: light|dark`
- Body: CSS 字符串

**不返回 BackgroundColorAsHex**，因此前端即使想更新也拿不到值。

**证据 5 — PWA manifest.json 使用默认主题背景色**：[manifest.json#L4-L5](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/manifest.json#L4-L5)

```json
"background_color": "{{ .App.Config.Branding.AppBackgroundColor }}",
"theme_color": "{{ .App.Config.Branding.AppBackgroundColor }}",
```

而 [glance.go#L220-L221](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/glance.go#L220-L221)：

```go
if config.Branding.AppBackgroundColor == "" {
    config.Branding.AppBackgroundColor = config.Theme.BackgroundColorAsHex
}
```

manifest.json 在**应用启动时一次性生成**，使用的是 `config.Theme`（用户配置的默认主题），与用户当前选择的预设主题无关。

### 4.2 源码可证实的行为结论

| 行为 | 源码证据 |
|------|---------|
| 首次页面加载时，theme-color = 当前请求主题的背景色 HEX | [document.html#L21](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L21) + [theme.go#L69-L73](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L69-L73) |
| 未配置背景色时，theme-color = `"#151519"` | [theme.go#L72](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L72) |
| JS `changeTheme()` 切换主题时，**theme-color 不变** | [page.js#L670-L692](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/js/page.js#L670-L692) 无相关代码 |
| PWA manifest 的 theme_color = 默认主题背景色，与用户当前选择无关 | [manifest.json#L5](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/manifest.json#L5) + [glance.go#L220-L221](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/glance.go#L220-L221) |
| `color-scheme` 永远是 dark，`theme-color` 可能是亮也可能是暗 | 见上文各自证据 |

### 4.3 保守推断：为何 theme-color 不随 JS 切换更新

代码中无注释。可能的原因：

1. **实现遗漏**：添加亮色主题和前端切换功能时，忘记同步更新 theme-color meta
2. **浏览器兼容性/限制**：不同浏览器对动态修改 theme-color 的支持不一致（尤其 iOS Safari），开发者选择回避
3. **视觉影响小**：theme-color 仅影响 PWA 安装后和 Android Chrome 地址栏，用户主要在页面内操作，优先级低
4. **需要后端额外返回 HEX 值**：当前 `handleThemeChangeRequest()` 只返回 CSS 和 X-Scheme，若要动态更新还需在响应头或 body 中附加 HEX 值，增加复杂度

> ⚠️ **保守推断声明**：以上均为推测，源码中无直接证据。

### 4.4 源码可证实：color-scheme 与 theme-color 的"分叉"

两者行为不一致是源码明确证实的事实，不是推断：

| 维度 | color-scheme | theme-color |
|------|-------------|-------------|
| 初次渲染 | 恒为 `"dark"` | = 当前请求主题背景色（可能亮也可能暗） |
| JS 切换主题 | 不变 | 不变 |
| Cookie 选择亮色主题 + 刷新页面 | 仍为 `"dark"` | 更新为亮色主题背景色 |
| PWA manifest | 不涉及 | = 默认主题背景色（启动时固定） |

用户选了亮色主题并刷新后：color-scheme 仍声明 dark，但 theme-color 已经是亮色值。这就是源码层面可证实的"分叉"。

---

## 五、自定义样式覆盖：源码可证实的级联机制

### 5.1 源码可证实：四级样式加载顺序

[document.html#L25-L29](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L25-L29)

```html
<!-- 第 1 层 -->
<link rel="stylesheet" href='{{ .App.StaticAssetPath "css/bundle.css" }}'>
<!-- 第 2 层 -->
<style id="theme-style">{{ .Request.Theme.CSS }}</style>
<!-- 第 3 层（可选） -->
{{ if .App.Config.Theme.CustomCSSFile }}<link rel="stylesheet" href="{{ .App.Config.Theme.CustomCSSFile }}?v={{ .App.CreatedAt.Unix }}">{{ end }}
<!-- 第 4 层（可选） -->
{{ if .App.Config.Document.Head }}{{ .App.Config.Document.Head }}{{ end }}
```

### 5.2 源码可证实：bundle.css 的内容构成

[embed.go#L87-L151](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/embed.go#L87-L151) 运行时递归解析 `@import`，从 `css/main.css` 开始：

```css
@import "site.css";
@import "widgets.css";
@import "popover.css";
@import "utils.css";
@import "mobile.css";
```

所有文件合并、去注释、压缩后输出。

### 5.3 源码可证实：theme-style 只包含变量

[theme-style.gotmpl](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/theme-style.gotmpl) 输出内容仅限：

```css
:root {
    --bgh: ...; --bgs: ...; --bgl: ...;
    --cm: ...; --tsm: ...;
    --color-primary: ...;
    --color-positive: ...;
    --color-negative: ...;
}
```

**不包含任何具体的选择器规则**（如 `.widget-content { ... }`）。

### 5.4 源码可证实：内置 CSS 中已有的非 `:root` 覆盖

[utils.css#L379-L381](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/css/utils.css#L379-L381)：

```css
:root:not([data-scheme=light]) .flat-icon {
    filter: invert(1);
}
```

特异性为 **(0,2,1)**（一个伪类 `:not()` + 一个属性选择器 `[data-scheme=light]` + 一个类选择器 `.flat-icon`），远高于仅写 `:root` 的 (0,1,0)。

这从源码层面证实：自定义样式如果要覆盖类似规则，不能只写 `:root`，需要匹配或超越相应特异性。

### 5.5 源码可证实：`document.head` 可注入任意内容

[config.go](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/config.go) 中 `Document.Head` 为 `string` 类型，模板中直接输出（无 HTML 转义）：

```html
{{ if .App.Config.Document.Head }}{{ .App.Config.Document.Head }}{{ end }}
```

因此用户可以注入：
- `<style>` 标签（可包含任何 CSS）
- `<link>` 标签（外部样式表、字体）
- `<script>` 标签（任意 JavaScript）

### 5.6 保守推断：用户自定义 CSS 的典型用例

代码库中没有示例或文档说明用户通常如何自定义。只能根据 CSS 能力推断：

1. 调整 `:root` 变量（改变配色）
2. 用 `:root[data-scheme=light]` 针对亮色模式做特殊调整
3. 修改具体组件样式（圆角、间距、字体等）
4. 隐藏不需要的 UI 元素（如主题选择器）

> ⚠️ **保守推断声明**：实际用户如何使用完全取决于外部社区，不在本代码库范围内。

---

## 六、可证实 vs 推断 总览表

| 结论 | 分类 | 依据 |
|------|------|------|
| default-dark 有条件注册，default-light 无条件注册 | ✅ 可证实 | [glance.go#L108-L122](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/glance.go#L108-L122) |
| default-dark 在用户完全未配置主题时不显示 | ✅ 可证实 | 真值表场景 A |
| `<meta color-scheme>` 恒为 "dark" | ✅ 可证实 | [document.html#L15](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L15) 字面量 |
| `<meta theme-color>` 首次渲染随主题变化 | ✅ 可证实 | [document.html#L21](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L21) 模板变量 |
| JS 切换主题时 theme-color **不更新** | ✅ 可证实 | [page.js#L670-L692](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/js/page.js#L670-L692) 无相关代码 |
| PWA manifest theme_color = 默认主题固定值 | ✅ 可证实 | [manifest.json#L5](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/manifest.json#L5) + [glance.go#L220-L228](file://://d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/glance.go#L220-L228) |
| color-scheme 与 theme-color 行为分叉是事实 | ✅ 可证实 | 见上文各自证据的组合 |
| color-scheme 硬编码 dark 的"原因" | ⚠️ 推断 | 代码中无注释，仅能从结构推测 |
| theme-color 不随 JS 更新的"原因" | ⚠️ 推断 | 代码中无注释，仅能从结构推测 |
| default-dark 条件表达式的设计意图 | ⚠️ 推断 | 代码中无注释，仅能从真值表反推 |
| 自定义 CSS 的典型用户用例 | ⚠️ 推断 | 不在代码库范围内 |

---

## 七、可证实结论的端到端链路图

```
YAML 配置解析
    │
    ├─ [config.go UnmarshalYAML]
    │
    ▼
应用启动时预设注册 [glance.go#L104-L128]
    │
    ├─ default-dark:  ok && !SameAs(default, userDark) || !SameAs(default, empty)
    │                   ├─ ✅ 可证实：表达式结构
    │                   ├─ ✅ 可证实：真值表 4 种场景
    │                   └─ ⚠️ 推断：设计意图
    │
    ├─ default-light: ✅ 可证实：无条件 append
    │
    └─ Merge(用户预设): ✅ 可证实：用户同名预设覆盖内置预设的数据
    │
    ▼
每个预设 init() [theme.go#L56-L76]
    │
    ├─ 生成 CSS 变量字符串（仅变量，无规则） ✅ 可证实
    ├─ 生成预览按钮 HTML                      ✅ 可证实
    └─ 计算 BackgroundColorAsHex               ✅ 可证实
    │
    ▼
页面 HTML 渲染 [document.html + page.html]
    │
    ├─ <html data-scheme="{{Light?light:dark}}">    ✅ 可证实
    ├─ <meta color-scheme content="dark">            ✅ 可证实：恒为 dark
    ├─ <meta theme-color content="{{BackgroundColorAsHex}}">  ✅ 可证实：随请求主题
    │
    ├─ 第 1 层 <link bundle.css>                     ✅ 可证实
    ├─ 第 2 层 <style id="theme-style"> 变量        ✅ 可证实
    ├─ 第 3 层 <link custom-css-file>（可选）       ✅ 可证实
    └─ 第 4 层 document.head（可选，任意内容）       ✅ 可证实
    │
    ▼
浏览器端 JS 初始化 [page.js initThemePicker]
    │
    ├─ 克隆移动端 .theme-choices 到桌面端            ✅ 可证实
    └─ 为所有 preset 按钮绑定 click 事件             ✅ 可证实
    │
    ▼
用户点击切换主题 [page.js changeTheme + theme.go handleThemeChangeRequest]
    │
    ├─ 后端返回 CSS + X-Scheme header                ✅ 可证实
    ├─ 后端不返回 BackgroundColorAsHex               ✅ 可证实
    ├─ 前端更新 #theme-style 内容                    ✅ 可证实
    ├─ 前端更新 data-theme / data-scheme             ✅ 可证实
    ├─ 前端不更新 <meta theme-color>                 ✅ 可证实
    └─ 前端不更新 <meta color-scheme>                ✅ 可证实
```

---

## 八、方法论说明

本文档中"源码可证实"与"保守推断"的划分标准：

**源码可证实**：
- 存在明确的源代码语句（字面量、表达式、函数调用）直接支撑
- 可以给出精确的文件路径和行号
- 任何读者阅读对应代码行都能得出相同结论

**保守推断**：
- 代码中无直接注释、文档字符串、测试用例说明设计意图
- 仅能从代码结构、条件表达式、缺失的代码（未做某件事）间接推理
- 不同读者可能得出不同解释
- 本文档标注为推断，并明确指出证据缺失
