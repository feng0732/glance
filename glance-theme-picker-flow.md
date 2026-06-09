# Glance 主题选择器与样式覆盖深度分析

## 概述

本文深入分析 Glance 主题系统中三个容易令人困惑的实现细节：
1. **default-dark / default-light 预设的注册逻辑** —— 为什么有时 default-dark 不显示？
2. **主题选择器 UI 的桌面/移动端双轨实现** —— preset 预览按钮如何进入桌面头部 popover？
3. **浏览器原生配色提示的分叉** —— 为什么 `color-scheme` 固定为 dark 而 `theme-color` 跟随主题？
4. **自定义样式覆盖的级联复杂性** —— 为什么不只是简单覆盖 `:root` 变量？

---

## 一、default-dark / default-light 预设注册逻辑

### 1.1 注册代码位置

预设注册发生在应用启动阶段的 [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/glance.go#L104-L141)：

```go
if !config.Theme.DisablePicker {
    themeKeys := make([]string, 0, 2)
    themeProps := make([]*themeProperties, 0, 2)

    // default-dark 条件注册（见下文）
    defaultDarkTheme, ok := config.Theme.Presets.Get("default-dark")
    if ok && !config.Theme.SameAs(defaultDarkTheme) || !config.Theme.SameAs(&themeProperties{}) {
        themeKeys = append(themeKeys, "default-dark")
        themeProps = append(themeProps, &themeProperties{})  // 空结构体
    }

    // default-light 无条件注册
    themeKeys = append(themeKeys, "default-light")
    themeProps = append(themeProps, &themeProperties{
        Light:                    true,
        BackgroundColor:          &hslColorField{240, 13, 95},
        PrimaryColor:             &hslColorField{230, 100, 30},
        NegativeColor:            &hslColorField{0, 70, 50},
        ContrastMultiplier:       1.3,
        TextSaturationMultiplier: 0.5,
    })

    // 合并顺序：内置预设在前，用户预设在后
    themePresets, err := newOrderedYAMLMap(themeKeys, themeProps)
    config.Theme.Presets = *themePresets.Merge(&config.Theme.Presets)

    // 初始化所有预设（生成 CSS、预览 HTML）
    for key, properties := range config.Theme.Presets.Items() {
        properties.Key = key
        properties.init()
    }
}
```

### 1.2 default-dark 的条件注册判定

`default-dark` 是否出现在选择器中，取决于这行复杂的条件判断：

```go
if ok && !config.Theme.SameAs(defaultDarkTheme) || !config.Theme.SameAs(&themeProperties{})
```

拆解为逻辑表达式（`||` 优先级低于 `&&`）：

```
(用户预设中已有 default-dark  AND  用户默认主题 != 那个 default-dark)
OR
(用户默认主题 != 空主题结构体)
```

**三种场景**：

| 场景 | default-dark 是否显示 | 原因 |
|------|----------------------|------|
| 用户 YAML 中没有自定义任何 theme 字段 | 不显示 | 用户默认主题 == 空结构体，选择它没意义 |
| 用户 YAML 中配置了 theme 字段（如改了 primary-color） | 显示 | 提供"回到纯默认暗色"的选项 |
| 用户预设中定义了自己的 `default-dark` 覆盖了内置，且默认主题就是它 | 不显示 | 避免显示重复的选项 |

### 1.3 default-dark 的"空结构体"含义

`&themeProperties{}` 意味着所有指针字段都是 `nil`，所有数值字段都是零值：
- `BackgroundColor = nil` → 不输出 `--bgh/--bgs/--bgl` → 使用 CSS 默认值（HSL 240, 8%, 9%）
- `PrimaryColor = nil` → 不输出 `--color-primary` → 使用 CSS 默认值（hsl(43, 50%, 70%)）
- `Light = false` → 暗色模式
- `ContrastMultiplier = 0` → 不输出 `--cm` → 使用 CSS 默认值 1

**关键洞察**：default-dark 不是"配置了一套暗色参数"，而是"什么都不配置，让 CSS 默认值全部生效"。这是一种零配置、零维护的内置主题方案。

### 1.4 Merge 顺序与用户覆盖权

```go
config.Theme.Presets = *themePresets.Merge(&config.Theme.Presets)
```

`Merge` 方法在 [config.go](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/config.go#L583-L600) 中实现：

```go
func (self *orderedYAMLMap[K, V]) Merge(other *orderedYAMLMap[K, V]) *orderedYAMLMap[K, V] {
    // self.keys（内置预设）先加入
    merged.keys = append(merged.keys, self.keys...)
    maps.Copy(merged.data, self.data)

    // other.keys（用户预设）后加入，已存在的 key 不重复加入 keys，但会覆盖 data
    for _, key := range other.keys {
        if _, exists := self.data[key]; !exists {
            merged.keys = append(merged.keys, key)
        }
    }
    maps.Copy(merged.data, other.data)  // 后 copy 的覆盖先 copy 的
    return merged
}
```

结果：
- **显示顺序**：内置预设（default-dark → default-light）在前，用户预设在后
- **同名覆盖**：用户可以在 YAML 中定义 `presets.default-dark: {...}` 来完全替换内置的 default-dark

---

## 二、主题选择器 UI：桌面与移动端双轨实现

### 2.1 桌面端 HTML 结构（popover 触发）

在 [page.html](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/page.html#L36-L45) 中：

```html
<div class="theme-picker self-center"
     data-popover-type="html"
     data-popover-position="below"
     data-popover-show-delay="0">
    <!-- 触发器：当前主题的预览按钮 -->
    <div class="current-theme-preview">
        {{ .Request.Theme.PreviewHTML }}
    </div>
    <!-- popover 内容容器：注意！这里是空的！ -->
    <div data-popover-html>
        <div class="theme-choices"></div>
    </div>
</div>
```

**桌面端的 `.theme-choices` 是空的**，预设按钮不会在这里渲染。

### 2.2 移动端 HTML 结构（点击触发，预设已全量渲染）

在 [page.html](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/page.html#L70-L93) 中：

```html
<div class="theme-picker flex justify-between items-center"
     data-popover-type="html"
     data-popover-position="above"
     data-popover-show-delay="0"
     data-popover-hide-delay="100"
     data-popover-anchor=".current-theme-preview"
     data-popover-trigger="click">       <!-- 关键：点击触发，非 hover -->
    <div data-popover-html>
        <!-- 移动端全量渲染所有预设！ -->
        <div class="theme-choices">
            {{ .App.Config.Theme.PreviewHTML }}       <!-- 默认主题预览 -->
            {{ range $_, $preset := .App.Config.Theme.Presets.Items }}
            {{ $preset.PreviewHTML }}                   <!-- 每个预设预览 -->
            {{ end }}
        </div>
    </div>
    ...
    <div class="current-theme-preview">
        {{ .Request.Theme.PreviewHTML }}
    </div>
</div>
```

### 2.3 JS 克隆：移动端 → 桌面端

在 [page.js](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/js/page.js#L694-L744) 的 `initThemePicker()` 中：

```javascript
function initThemePicker() {
    const themeChoicesInMobileNav = find(".mobile-navigation .theme-choices");
    if (!themeChoicesInMobileNav) return;

    const themeChoicesInHeader = find(".header-container .theme-choices");

    // 桌面端的空 theme-choices 被移动端的完整克隆替换
    if (themeChoicesInHeader) {
        themeChoicesInHeader.replaceWith(
            themeChoicesInMobileNav.cloneNode(true)  // deep clone
        );
    }

    // 然后统一为所有 .theme-choices 中的按钮绑定事件
    const presetElems = findAll(".theme-choices .theme-preset");
    // ...
}
```

**为什么不直接在两端都渲染？**
- 减少服务端模板重复代码
- 保证两端预设列表、顺序、视觉完全一致
- 事件绑定只需做一次

### 2.4 Popover 机制：DOM 节点的"乾坤大挪移"

当鼠标悬停（桌面）或点击（移动端）触发 popover 时，[popover.js](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/js/popover.js#L73-L89) 会执行一段精妙的 DOM 操作：

```javascript
if (popoverType === "html") {
    const htmlContent = activeTarget.querySelector(htmlContentSelector);
    // 用注释节点占位，保持原位置
    const placeholder = document.createComment("");
    htmlContent.replaceWith(placeholder);

    // 将真实内容移入全局 popover 容器
    contentElement.replaceChildren(htmlContent);
    htmlContent.removeAttribute("data-popover-html");

    // 关闭时还原
    cleanupOnHidePopover = () => {
        htmlContent.setAttribute("data-popover-html", "");
        placeholder.replaceWith(htmlContent);
        placeholder.remove();
    };
}
```

**为什么不 innerHTML 而是移动 DOM 节点？**
> The reason for all of the below shenanigans is that I want to preserve all attached event listeners of the original HTML content.

如果用 `innerHTML` 克隆，`initThemePicker()` 中已经绑定到按钮上的点击事件会丢失。移动节点而非复制，保证了事件监听器、JS 引用全部存活。

### 2.5 桌面 vs 移动端触发差异对比

| 特性 | 桌面端 | 移动端 |
|------|--------|--------|
| 触发方式 | `mouseenter`（hover） | `click`（点击） |
| popover 位置 | `below`（下方） | `above`（上方） |
| `.theme-choices` 渲染 | 初始为空，JS 克隆自移动端 | 服务端全量渲染 |
| 锚点元素 | `.theme-picker` 整体 | `.current-theme-preview` |

---

## 三、Preset 预览 HTML 生成

### 3.1 预览模板

每个主题的预览按钮由 [theme-preset-preview.html](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/theme-preset-preview.html) 生成：

```html
{{- $background := "hsl(240, 8%, 9%)" | safeCSS }}
{{- $primary := "hsl(43, 50%, 70%)" | safeCSS  }}
{{- $positive := "hsl(43, 50%, 70%)" | safeCSS }}
{{- $negative := "hsl(0, 70%, 70%)" | safeCSS }}
{{- if .BackgroundColor }}{{ $background = .BackgroundColor.String | safeCSS }}{{ end }}
{{- if .PrimaryColor }}
    {{- $primary = .PrimaryColor.String | safeCSS }}
    {{- if not .PositiveColor }}
        {{- $positive = $primary }}       {{/* positive 未配置时继承 primary */}}
    {{- else }}
        {{- $positive = .PositiveColor.String | safeCSS }}
    {{- end }}
{{- end }}
{{- if .NegativeColor }}{{ $negative = .NegativeColor.String | safeCSS }}{{ end }}

<button class="theme-preset{{ if .Light }} theme-preset-light{{ end }}"
        style="--color: {{ $background }}"
        data-key="{{ .Key }}">
    <div class="theme-color" style="--color: {{ $primary }}"></div>
    <div class="theme-color" style="--color: {{ $positive }}"></div>
    <div class="theme-color" style="--color: {{ $negative }}"></div>
</button>
```

### 3.2 预览按钮的 CSS 渲染

在 [site.css](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/css/site.css#L342-L389) 中：

```css
.theme-preset {
    background-color: var(--color);   /* 取自身 style 中的 --color（背景色） */
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.5rem;
    height: 2rem;
    /* ... */
}

.theme-color {
    background-color: var(--color);   /* 取各自 style 中的 --color（主/正/负色） */
    width: 0.9rem;
    height: 0.9rem;
    border-radius: 0.2rem;
}

.theme-preset-light {
    height: 1.8rem;
    /* 亮色主题的预览按钮稍矮，色块稍大 */
}
.theme-preset-light .theme-color {
    width: 1rem;
    height: 1rem;
    border-radius: 0.3rem;
}
```

巧妙之处：每个元素通过内联 `style="--color: xxx"` 设置**自己的局部 CSS 变量**，然后由通用类读取。这种模式避免了内联样式覆盖类样式的问题。

### 3.3 预览按钮在主题切换后的同步更新

在 [page.js](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/js/page.js#L726-L741) 中，切换主题成功后：

```javascript
changeTheme(themeKey, function() {
    // ...
    // 将页面中所有 .current-theme-preview（桌面头部 + 移动端各一个）
    // 内部的预览按钮替换为被点击的那个 preset 的克隆
    Array.from(themePreviewElems).forEach((preview) => {
        preview.querySelector(".theme-preset").replaceWith(
            presetElement.cloneNode(true)
        );
    });
    // ...
});
```

这确保了"当前主题指示器"与用户选择的视觉完全同步。

---

## 四、浏览器原生配色：color-scheme 与 theme-color 为何分叉

### 4.1 两个 meta 标签的当前行为

在 [document.html](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L14-L21) 中：

```html
<!-- 固定为 "dark"，永不随主题变化 -->
<meta name="color-scheme" content="dark">

<!-- 跟随当前主题的背景色动态变化 -->
<meta name="theme-color" content="{{ .Request.Theme.BackgroundColorAsHex }}">

<!-- 也固定为 "black-translucent" -->
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
```

### 4.2 两个 meta 的职责差异

| meta 标签 | 浏览器作用 | 为何固定/动态 |
|-----------|-----------|--------------|
| `color-scheme` | 控制浏览器**原生 UI 元素**的配色：<br>• 滚动条颜色<br>• `<input>` 表单控件（日期选择器、下拉框等）<br>• `::selection` 选中高亮<br>• 右键菜单、弹出框 | **固定为 dark**：<br>避免亮色主题切换时原生控件出现样式冲突。如果用户选择亮色主题但页面仍有大量暗色元素（如某些 widget 内嵌），原生控件闪烁切换会产生视觉割裂。固定 dark 可保持一致性。 |
| `theme-color` | 控制 **PWA/浏览器外壳** 的颜色：<br>• Android Chrome 地址栏背景<br>• 安装到桌面后应用窗口标题栏<br>• iOS Safari 全屏时的状态栏周边 | **动态跟随主题**：<br>这是用户感知最明显的"应用颜色"，需要与页面实际背景色一致，否则页面上方会出现色块不协调。 |
| `apple-mobile-web-app-status-bar-style` | 仅 iOS PWA：状态栏文字颜色（`default`/`black-translucent`/`black`） | **固定为 black-translucent**：<br>iOS 不支持根据主题动态切换此值（除非用 JS 手动刷新页面）。`black-translucent` 意味着状态栏文字为白色，适合暗色背景，也能兼容大部分亮色主题（需要亮色主题自行保证顶部有足够对比度）。 |

### 4.3 color-scheme 与 CSS 自定义主题的互动

CSS 中有一个容易被忽略的交互：浏览器对 `color-scheme` 的响应会影响 `hsl()` 中某些颜色的渲染，以及滚动条的默认外观。

Glance 选择硬编码 `content="dark"` 的更深层原因：
1. **避免滚动条反色**：如果 `color-scheme` 跟随主题变成 `light`，浏览器滚动条会变成浅色，而 Glance 的滚动条颜色由 `::-webkit-scrollbar` 自定义控制，与 `color-scheme` 冲突会导致双重样式
2. **表单控件一致性**：所有 widget 中的表单元素都使用了自定义 CSS（`background: none; border: 0`），但某些原生下拉、日期选择仍由浏览器绘制，固定 dark 可避免它们在亮色主题下"跳出"
3. **简化动态切换**：`theme-color` 可以在 JS 切换主题时通过 `document.querySelector('meta[name=theme-color]')` 更新（虽然当前 `changeTheme()` 尚未做），但 `color-scheme` 的变更会触发整页重绘，影响动画和过渡效果

### 4.4 动态切换时 theme-color 的更新（当前未实现）

当前 [page.js](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/js/page.js#L670-L692) 的 `changeTheme()` 函数只更新了：
- `<style id="theme-style">` 的 CSS 内容
- `<html data-theme>` 和 `<html data-scheme>` 属性

**但没有更新 `<meta name="theme-color">`**。这意味着如果用户在前端切换到亮色主题，Android Chrome 地址栏的颜色仍然是初始加载时的暗色背景色。这是一个潜在的改进点。

---

## 五、自定义样式覆盖为何不只是 `:root` 覆盖

### 5.1 四级样式来源回顾

从 [document.html](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L25-L29) 加载顺序：

```html
<!-- 1. bundle.css（内置）:root 默认变量 + 所有布局规则 -->
<link rel="stylesheet" href='{{ .App.StaticAssetPath "css/bundle.css" }}'>

<!-- 2. 内联 theme-style（YAML 主题配置生成） -->
<style id="theme-style">{{ .Request.Theme.CSS }}</style>

<!-- 3. custom-css-file（用户指定外部 CSS） -->
{{ if .App.Config.Theme.CustomCSSFile }}
<link rel="stylesheet" href="{{ .App.Config.Theme.CustomCSSFile }}?v={{ .App.CreatedAt.Unix }}">
{{ end }}

<!-- 4. document.head（YAML 中可写任意 HTML） -->
{{ if .App.Config.Document.Head }}{{ .App.Config.Document.Head }}{{ end }}
```

### 5.2 自定义 CSS 的能力远超 `:root` 变量

用户通过 `custom-css-file` 或 `document.head` 能做的事情远不止覆盖 CSS 变量：

#### 能力 1：使用属性选择器针对特定主题/模式

内置 CSS 本身已经在使用这种模式，例如 [utils.css](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/css/utils.css#L379-L381)：

```css
:root:not([data-scheme=light]) .flat-icon {
    filter: invert(1);
}
```

用户也可以写：

```css
/* 仅在暗色模式生效 */
:root:not([data-scheme=light]) .widget-content {
    border: 1px solid #333;
}

/* 仅在亮色模式生效 */
:root[data-scheme=light] .nav-item {
    color: #222;
}

/* 仅当选择了某个特定预设 */
:root[data-theme="teal-city"] .logo {
    filter: hue-rotate(30deg);
}
```

这些选择器的特异性是 **(0,1,1)** 或 **(0,2,0)**，远高于仅写 `:root` 的 **(0,1,0)**，因此能可靠地覆盖内置样式。

#### 能力 2：覆盖具体的 CSS 规则（非变量）

用户不仅能改变 `--color-primary`，还能直接覆盖具体组件的样式：

```css
/* 改变所有 widget 的圆角 */
.widget-content {
    border-radius: 12px !important;
}

/* 隐藏主题选择器（不需要 YAML 配置 disable-picker） */
.theme-picker {
    display: none !important;
}

/* 只给 clock widget 添加特殊背景 */
.widget-type-clock .widget-content {
    background: linear-gradient(135deg, var(--color-widget-background), transparent);
}
```

#### 能力 3：针对特定容器尺寸使用 container query

Glance 使用了 CSS Container Query，例如 [utils.css](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/css/utils.css#L120-L163)：

```css
@container widget (max-width: 599px) {
    .dynamic-columns { /* ... */ }
}
```

用户也可以在自定义 CSS 中写 container query 来响应 widget 尺寸变化。

#### 能力 4：document.head 中注入任意内容

通过 YAML 配置：

```yaml
document:
  head: |
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter">
    <style>
      body { font-family: 'Inter', sans-serif; }
      /* 可以写任何你想要的 CSS */
    </style>
    <script>
      console.log('自定义 JS 也能注入');
    </script>
```

### 5.3 级联的特异性矩阵

当用户样式与内置样式冲突时，最终生效的是哪个？取决于 **特异性（Specificity）+ 加载顺序**：

| 用户写法 | 特异性 | 是否能覆盖 bundle.css 中的同名规则 |
|----------|--------|----------------------------------|
| `:root { --x: y }` | (0,1,0) | 能（加载顺序靠后） |
| `:root[data-scheme=light] { --x: y }` | (0,2,0) | 能（特异性更高 + 顺序靠后） |
| `.widget-content { border: ... }` | (0,1,0) | 能（顺序靠后） |
| `.header .nav-item { color: ... }` | (0,2,0) | 能（特异性更高 + 顺序靠后） |
| `body { color: ... !important }` | (1,0,0) + !important | 绝对能 |

**反例**（用户写的也可能不生效）：

```css
/* 不生效，因为 bundle.css 中已经有更高特异性 */
:root .flat-icon {
    filter: invert(0);
}
/* 内置是 :root:not([data-scheme=light]) .flat-icon 特异性 (0,2,1) */
/* 用户上面这个只有 (0,2,0) */
```

正确写法：
```css
:root:not([data-scheme=light]) .flat-icon {
    filter: invert(0) !important;
}
```

### 5.4 为什么 custom-css-file 有 `?v=时间戳`

```html
<link rel="stylesheet" href="{{ .App.Config.Theme.CustomCSSFile }}?v={{ .App.CreatedAt.Unix }}">
```

`App.CreatedAt` 是应用启动时的时间戳。每次重启应用，URL 的 query string 都会变化，强制浏览器重新加载 CSS 文件而不是使用缓存。这保证用户修改自定义 CSS 后重启即可看到效果。

---

## 六、端到端流程图：从预设注册到浏览器渲染

```
┌──────────────────────────────────────────────────────────────────────────┐
│  1. 应用启动 (newApplication)                                            │
│                                                                          │
│   ┌─────────────────────┐    ┌──────────────────────┐                   │
│   │ default-dark (条件) │    │ default-light (固定) │                   │
│   │  &themeProperties{} │    │  Light + 亮色配色     │                   │
│   └──────────┬──────────┘    └──────────┬───────────┘                   │
│              └────────────┬──────────────┘                               │
│                           ▼                                              │
│                 Merge(用户 presets)                                       │
│                 → 用户预设覆盖同名内置预设                                 │
│                           │                                              │
│                           ▼                                              │
│              对每个 preset 调用 init():                                   │
│                • 执行 theme-style.gotmpl → 生成 CSS 变量字符串            │
│                • 执行 theme-preset-preview.html → 生成预览按钮 HTML       │
│                • 背景色转 HEX → BackgroundColorAsHex                      │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  2. 页面请求 (handlePageRequest)                                          │
│                                                                          │
│   读取 Cookie "theme":                                                    │
│     • 存在且是有效 preset → 使用 preset 主题                               │
│     • 否则 → 使用 config.Theme（用户配置的默认主题）                       │
│                            │                                              │
│                            ▼                                              │
│   渲染 document.html:                                                      │
│     <html data-theme="xxx" data-scheme="dark|light">                      │
│     <meta name="color-scheme" content="dark">     ← 固定！                │
│     <meta name="theme-color" content="#151519">   ← 动态！                │
│     <link rel="stylesheet" href="bundle.css">    ← 第1层 CSS             │
│     <style id="theme-style">:root { --bgh: 240; ... }</style> ← 第2层     │
│     <link rel="stylesheet" href="custom.css?v=xxx"> ← 第3层 (可选)        │
│     document.head 任意内容 ...                    ← 第4层 (可选)         │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  3. 浏览器端 JS 执行 (setupPage)                                          │
│                                                                          │
│   initThemePicker():                                                      │
│     • 找到移动端 .theme-choices（已有完整预设列表）                        │
│     • cloneNode(true) → 替换桌面端的空 .theme-choices                     │
│     • 为所有 .theme-preset 按钮绑定 click 事件                             │
│                            │                                              │
│                            ▼                                              │
│   setupPopovers():                                                         │
│     • 为 [data-popover-type] 绑定 mouseenter/click 事件                    │
│     • hover 时把 [data-popover-html] 从原位移到全局 popover 容器             │
│       （保留事件监听器）                                                    │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  4. 用户点击切换主题 (changeTheme)                                         │
│                                                                          │
│   POST /api/set-theme/{key}:                                              │
│     • 服务端设置 Cookie: theme=xxx (2年过期)                               │
│     • 返回该 preset 的 CSS 字符串                                          │
│     • X-Scheme header: "light" 或 "dark"                                  │
│                            │                                              │
│                            ▼                                              │
│   前端更新：                                                               │
│     • 临时插入 * { transition: none !important } 避免动画闪烁              │
│     • 替换 <style id="theme-style"> 内容                                   │
│     • 更新 <html data-theme> 和 <html data-scheme>                        │
│     • 同步所有 .current-theme-preview 的预览按钮                           │
│     • ⚠️ 未更新 <meta name="theme-color">（潜在遗漏）                      │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 七、关键设计要点总结

### 7.1 预设注册的"空结构体"设计

default-dark 不是"配置了一套暗色值"，而是**完全不配置**——所有指针为 nil，所有数值为 0，让 CSS 默认值自动生效。这种设计消除了两套默认值（Go 代码 vs CSS）之间的维护成本。

### 7.2 桌面/移动端预设列表的"单渲染源"

只在移动端模板中渲染完整预设列表，桌面端通过 JS 克隆。这避免了服务端模板重复代码，也确保了两端的预设顺序、样式、事件绑定完全一致。

### 7.3 Popover 的 DOM 移动而非克隆

使用注释节点占位 + DOM 节点移动的技巧，保证了预设按钮上已绑定的事件监听器不会因为 popover 显示/隐藏而丢失。这比 `innerHTML` 复制更健壮。

### 7.4 color-scheme / theme-color 的有意分叉

- `color-scheme` 固定 dark：保证浏览器原生控件（滚动条、表单）不随主题切换产生闪烁或样式冲突
- `theme-color` 动态跟随：保证 PWA/浏览器外壳颜色与页面视觉一致
- 两者解决的问题不同，因此"分叉"是有意的设计选择，而非疏忽

### 7.5 自定义样式的多层覆盖能力

用户自定义 CSS 不只是 `:root { --variable: value }` 这么简单：
- 可以用 `:root[data-scheme=...]` 针对特定亮/暗模式
- 可以用 `:root[data-theme=...]` 针对特定预设
- 可以直接覆盖具体组件规则
- 可以使用 Container Queries、伪元素、`!important` 等全套 CSS 能力
- 还可以通过 `document.head` 注入字体、脚本、额外样式表
