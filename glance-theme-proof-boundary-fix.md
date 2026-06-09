# Glance 主题系统：源码可证实行为 vs 保守推断边界（修正版）

## 修正说明

本文档是 `glance-theme-proof-boundary.md` 的修正版。**主要修正**：default-dark 的场景 D（用户定义了 presets.default-dark 且默认主题等于它）之前被错误判断为"不显示"，经逐行代码验算，实际上在绝大多数子场景下 default-dark **仍然显示**。详见第 1.4 节的完整验算过程。

---

## 一、default-dark / default-light 显示条件

### 1.1 源码定位

核心代码位于 [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/glance.go#L104-L112)：

```go
defaultDarkTheme, ok := config.Theme.Presets.Get("default-dark")
if ok && !config.Theme.SameAs(defaultDarkTheme) || !config.Theme.SameAs(&themeProperties{}) {
    themeKeys = append(themeKeys, "default-dark")
    themeProps = append(themeProps, &themeProperties{})
}
```

辅助代码：
- `SameAs` 方法：[theme.go#L78-L107](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L78-L107)
- `hslColorField.SameAs`：[config-fields.go#L39-L47](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/config-fields.go#L39-L47)
- `themeProperties` 结构体定义：[theme.go#L41-L54](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L41-L54)

### 1.2 源码可证实：表达式拆解

**运算符优先级**：Go 语言中 `&&` 优先级高于 `||`（与 C/Java/Python 一致）。因此：

```go
ok && !config.Theme.SameAs(defaultDarkTheme) || !config.Theme.SameAs(&themeProperties{})
```

等价于（加括号显式分组）：

```go
(ok && !config.Theme.SameAs(defaultDarkTheme)) || (!config.Theme.SameAs(&themeProperties{}))
```

**符号定义**：
| 符号 | 含义 | 取值来源 |
|------|------|---------|
| `ok` | 用户 YAML 的 `theme.presets` 中是否存在 key 为 `"default-dark"` 的预设 | [config.go Get()](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/config.go#L578-L581) |
| `默认主题` | `config.Theme.themeProperties`（内嵌结构体，YAML 中 `theme:` 下直接写的字段） | [config.go#L48-L54](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/config.go#L48-L54) |
| `用户default-dark` | `config.Theme.Presets.Get("default-dark")` 返回的值（如果存在） | 同上 |
| `空结构体` | `&themeProperties{}`（所有指针 nil，所有数值零值） | Go 语言零值规则 |

**`SameAs` 逐字段比较清单**（[theme.go#L85-L105](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L85-L105)）：

```go
// 只要有一个字段不同就返回 false
t1.Light != t2.Light
t1.ContrastMultiplier != t2.ContrastMultiplier
t1.TextSaturationMultiplier != t2.TextSaturationMultiplier
!t1.BackgroundColor.SameAs(t2.BackgroundColor)   // nil == nil 为 true
!t1.PrimaryColor.SameAs(t2.PrimaryColor)          // nil != 非nil 为 false
!t1.PositiveColor.SameAs(t2.PositiveColor)
!t1.NegativeColor.SameAs(t2.NegativeColor)
```

### 1.3 源码可证实：`themeProperties{}` 空结构体的字段值

根据 [theme.go#L41-L54](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L41-L54) 的结构体定义和 Go 零值规则：

| 字段 | Go 零值 | SameAs 比较含义 |
|------|---------|----------------|
| `BackgroundColor` | `nil` | 未配置背景色 |
| `PrimaryColor` | `nil` | 未配置主色 |
| `PositiveColor` | `nil` | 未配置正向色 |
| `NegativeColor` | `nil` | 未配置负向色 |
| `Light` | `false` | 暗色模式 |
| `ContrastMultiplier` | `0.0` | 未配置对比度乘数 |
| `TextSaturationMultiplier` | `0.0` | 未配置文字饱和度乘数 |

`ContrastMultiplier` 和 `TextSaturationMultiplier` 的零值 `0.0` 在 [theme-style.gotmpl](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/theme-style.gotmpl) 中被特殊处理：
```css
{{ if ne 0.0 .ContrastMultiplier }}--cm: {{ .ContrastMultiplier }};{{ end }}
```
即 `0.0` 等于"未配置"，使用 CSS 默认值 `1`。但 `SameAs` 比较时 `0.0 != 1.3`，所以如果用户配置了 `contrast-multiplier: 1.3`，与空结构体比较会返回 `false`。

---

### 1.4 源码可证实：逐场景验算

以下每个场景均给出完整代码路径的验算步骤。

---

#### 场景 A：用户 YAML 完全没有 theme 相关配置

**YAML**：不存在 `theme:` 块，或 `theme:` 为空，且无任何 `presets`。

**变量赋值**：
- `config.Theme.themeProperties` = 空结构体（所有 nil 和零值）
- `config.Theme.Presets` = 空 map，因此 `ok = false`

**左边子表达式** `(ok && !SameAs(默认主题, 用户default-dark))`：
- `ok = false`
- Go 的 `&&` 是短路运算符，`false && X` 直接返回 `false`，不计算 `X`
- 左边 = `false`

**右边子表达式** `(!SameAs(默认主题, 空结构体))`：
- 默认主题 = 空结构体
- `SameAs(空结构体, 空结构体)` → 所有字段相等 → `true`
- `!true = false`
- 右边 = `false`

**最终结果**：`false || false = false` → default-dark **不显示**

---

#### 场景 B：用户配置了 theme 字段（任何一项），但未定义 presets

**YAML 示例**：
```yaml
theme:
  primary-color: hsl(43, 50%, 70%)
```

**变量赋值**：
- `config.Theme.PrimaryColor = &hslColorField{H:43, S:50, L:70}`（非 nil）
- 其余字段为零值
- `ok = false`（Presets 为空 map）

**左边子表达式**：
- `ok = false`
- 短路 → 左边 = `false`

**右边子表达式** `(!SameAs(默认主题, 空结构体))`：
- `SameAs` 比较到 `PrimaryColor` 时：
  - 默认主题：`PrimaryColor = &hslColorField{43,50,70}`（非 nil）
  - 空结构体：`PrimaryColor = nil`
  - `hslColorField.SameAs` 逻辑：[config-fields.go#L43-L44](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/config-fields.go#L43-L44)
    ```go
    if c1 == nil || c2 == nil {
        return false    // 一个 nil 一个非 nil → 不相等
    }
    ```
  - 所以 `PrimaryColor.SameAs(nil) = false`
- `themeProperties.SameAs` 中：`!false = true` → 直接返回 `false`（整体不相等）
- `!false = true`
- 右边 = `true`

**最终结果**：`false || true = true` → default-dark **显示**

> **关键观察**：只要用户在 `theme:` 下配置了**任何一个字段**（颜色、light、contrast-multiplier、text-saturation-multiplier 中任意一个），右边就是 `true`，default-dark 就会显示。

---

#### 场景 C：用户定义了 presets.default-dark，且默认主题 ≠ 用户 default-dark

**YAML 示例**：
```yaml
theme:
  primary-color: hsl(0, 100%, 50%)        # 默认主题：红色主色
  presets:
    default-dark:
      primary-color: hsl(120, 100%, 50%)  # 用户自定义 default-dark：绿色主色
```

**变量赋值**：
- `ok = true`（Presets 中存在 key `"default-dark"`）
- `默认主题.PrimaryColor = &hslColorField{H:0, S:100, L:50}`（红色）
- `用户default-dark.PrimaryColor = &hslColorField{H:120, S:100, L:50}`（绿色）

**左边子表达式** `(ok && !SameAs(默认主题, 用户default-dark))`：
- `ok = true`，不短路，继续计算
- `SameAs(默认主题, 用户default-dark)` 比较：
  - `PrimaryColor.SameAs`：H=0 vs H=120 → 不相等 → `false`
  - 整体 `SameAs` = `false`
- `!false = true`
- 左边 = `true && true = true`

**右边子表达式**：
- `||` 短路运算符，左边为 `true` 时不计算右边

**最终结果**：`true || (不计算) = true` → default-dark **显示**

---

#### 场景 D：用户定义了 presets.default-dark，且默认主题 = 用户 default-dark

这个场景需要**分两种子情况**，因为右边子表达式 `!SameAs(默认主题, 空结构体)` 的值取决于用户 default-dark 本身是否为"空"。

---

##### 子场景 D1：用户 default-dark 有实际配置（非空），默认主题 = 它

**YAML 示例**：
```yaml
theme:
  primary-color: hsl(120, 100%, 50%)      # 默认主题：绿色主色
  presets:
    default-dark:
      primary-color: hsl(120, 100%, 50%)  # 用户 default-dark 也是绿色主色
```

**变量赋值**：
- `ok = true`
- `默认主题.PrimaryColor = 用户default-dark.PrimaryColor = &hslColorField{H:120, S:100, L:50}`
- 两者所有字段完全相同

**左边子表达式** `(ok && !SameAs(默认主题, 用户default-dark))`：
- `ok = true`
- `SameAs(默认主题, 用户default-dark)` → 所有字段相等 → `true`
- `!true = false`
- 左边 = `true && false = false`

**右边子表达式** `(!SameAs(默认主题, 空结构体))`：
- 默认主题有 `PrimaryColor`（非 nil），空结构体没有
- 同场景 B 的右边计算逻辑 → `SameAs = false`
- `!false = true`
- 右边 = `true`

**最终结果**：`false || true = true` → default-dark **显示**

> **⚠️ 之前的错误**：此场景之前被误判为"不显示"，实际上右边子表达式恒为 `true`（因为默认主题非空），所以 default-dark **仍然显示**。

---

##### 子场景 D2：用户 default-dark 是空结构体，默认主题 = 它（理论边界情况）

**YAML 示例**（实际中几乎不会有人这么写）：
```yaml
theme:
  # 默认主题：完全不配置任何字段
  presets:
    default-dark: {}  # 明确写一个空 preset
```

**变量赋值**：
- `ok = true`
- 默认主题 = 用户 default-dark = 空结构体

**左边子表达式**：
- `ok = true`
- `SameAs(空结构体, 空结构体) = true`
- `!true = false`
- 左边 = `true && false = false`

**右边子表达式**：
- `SameAs(空结构体, 空结构体) = true`
- `!true = false`
- 右边 = `false`

**最终结果**：`false || false = false` → default-dark **不显示**

> 这个子场景在逻辑上等价于**场景 A**（默认主题就是空结构体），唯一区别是用户多写了一个无意义的空 preset。实际使用中几乎不会出现。

---

### 1.5 源码可证实：修正后的真值表

| # | 场景 | `ok` | 左边<br>`ok && !SameAs(默认, 用户D)` | 右边<br>`!SameAs(默认, 空)` | 最终结果 | default-dark 显示? |
|---|------|------|------------------------------------|---------------------------|---------|-------------------|
| A | 无任何 theme 配置 | false | `false`（短路） | `false` | `false` | ❌ 不显示 |
| B | 有 theme 字段，无 presets | false | `false`（短路） | `true` | `true` | ✅ 显示 |
| C | 有 presets.default-dark，默认 ≠ 它 | true | `true`（短路） | （不计算） | `true` | ✅ 显示 |
| D1 | 有 presets.default-dark，默认 = 它（且非空） | true | `false` | `true` | `true` | ✅ 显示 |
| D2 | 有 presets.default-dark，默认 = 它（且为空，边界） | true | `false` | `false` | `false` | ❌ 不显示 |

**一句话总结**：default-dark **仅在默认主题等于空结构体时不显示**（场景 A 和 D2，D2 是 A 的理论等价变体）。只要用户配置了任何 theme 字段，default-dark 就会显示。

### 1.6 源码可证实：default-light 的行为

- **无条件注册**：[glance.go#L114-L122](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/glance.go#L114-L122) 中直接 `append`，无任何 `if` 判断
- **注册顺序**：先处理 default-dark（条件满足才加入），再加入 default-light
- **显示顺序**：default-dark? → default-light → 用户自定义预设（按 YAML 中定义顺序），由 [config.go Merge()](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/config.go#L583-L600) 决定
- **覆盖规则**：用户预设可以用同名 key `default-light` 完全替换内置的 default-light 数据（同 default-dark 的覆盖逻辑）

---

## 二、保守推断区：设计原因（无源码直接证据）

以下内容在代码中**没有注释、文档字符串或测试用例**直接说明设计意图，仅为基于代码结构的合理推断。

### 2.1 default-dark 条件表达式的设计原因

**推断 1：避免"无意义的重复选项"**

当用户完全没有配置任何主题（场景 A）时，Glance 渲染出的默认效果与选择 default-dark 完全相同（因为两者都是"让 CSS 默认值全部生效"）。此时在选择器中显示 default-dark 是多余的——用户当前看到的就是 default-dark 的效果。

**推断 2：提供"回到纯默认"的逃生舱**

当用户修改了主题配置后（场景 B、C、D1），选择器中提供 default-dark 选项，允许用户一键回到"完全不配置任何字段、使用 CSS 默认值"的状态，而不需要手动删除 YAML 中的 theme 字段并重启服务。

**推断 3：左边子表达式处理的边缘场景**

`ok && !SameAs(默认主题, 用户default-dark)` 左边子表达式处理的是：用户覆盖了内置的 default-dark preset 且当前默认主题不是它。此时需要显示用户自定义的 default-dark（通过 Merge 覆盖了内置数据）。

> ⚠️ **保守推断声明**：以上三条仅为推测。代码中无任何注释或设计文档证明作者的真实意图。条件表达式写得如此复杂，也可能是历史迭代中逐渐叠加形成的，而非一次性精心设计的结果。

### 2.2 color-scheme 硬编码为 dark 的原因

可能的原因（按合理性排序，但均无代码证据）：

1. **浏览器原生控件样式冲突**：Glance 对滚动条、表单元素有大量自定义 CSS（`background: none; border: 0` 等），如果 `color-scheme` 跟随主题变为 light，浏览器可能在某些控件上强制应用亮色基础样式，与自定义样式产生视觉冲突
2. **避免动态切换时的重绘闪烁**：`color-scheme` 变更会触发浏览器对原生 UI（滚动条、选中高亮、右键菜单等）的重绘，可能与主题切换的 CSS 过渡动画产生冲突
3. **历史遗留**：早期版本仅支持暗色主题，添加亮色主题功能时遗漏了更新此 meta 标签

### 2.3 theme-color 不随 JS 切换更新的原因

可能的原因：

1. **实现遗漏**：添加前端动态切换主题功能时，忘记同步更新 `<meta name="theme-color">`
2. **浏览器兼容性不一致**：不同浏览器（尤其 iOS Safari）对运行时动态修改 theme-color 的支持和行为不一致，开发者选择回避此问题
3. **后端响应缺失必要数据**：当前 `handleThemeChangeRequest()` 仅返回 CSS 字符串和 `X-Scheme` header，不返回 `BackgroundColorAsHex`。若要在前端更新 theme-color，需要后端额外返回此值（通过 header 或修改响应格式），增加了复杂度
4. **视觉影响优先级低**：theme-color 仅影响 Android Chrome 地址栏和 PWA 安装后的窗口标题栏，用户主要关注页面内容，此细节的优先级较低

---

## 三、源码可证实：color-scheme 与 theme-color 的行为分叉

此部分为**纯源码可证实**的事实，不包含任何推断。

### 3.1 代码证据链

**`<meta name="color-scheme">`**：
- [document.html#L15](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L15)：字面量字符串 `content="dark"`，无模板变量
- 全代码库搜索 `color-scheme` 仅此一处出现（grep 结果已证实）
- [page.js#L670-L692](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/js/page.js#L670-L692) `changeTheme()` 函数无任何相关更新代码

**`<meta name="theme-color">`**：
- [document.html#L21](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L21)：模板变量 `content="{{ .Request.Theme.BackgroundColorAsHex }}"`
- [theme.go#L69-L73](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L69-L73)：`BackgroundColorAsHex` 的计算逻辑
  - 配置了背景色 → `BackgroundColor.ToHex()`
  - 未配置 → `"#151519"`（硬编码默认值）
- [page.js#L670-L692](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/js/page.js#L670-L692) `changeTheme()` 中**无任何更新此 meta 的代码**

**PWA manifest.json**：
- [manifest.json#L4-L5](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/manifest.json#L4-L5)：
  ```json
  "background_color": "{{ .App.Config.Branding.AppBackgroundColor }}",
  "theme_color": "{{ .App.Config.Branding.AppBackgroundColor }}"
  ```
- [glance.go#L220-L228](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/glance.go#L220-L228)：应用启动时一次性渲染，使用 `config.Theme.BackgroundColorAsHex`（默认主题），与用户运行时选择的预设无关

### 3.2 行为对比表

| 维度 | color-scheme | theme-color | 结论 |
|------|-------------|-------------|------|
| 首次渲染值 | 恒为 `"dark"` | = 当前请求主题背景色（可能亮或暗） | ✅ 可证实分叉 |
| JS `changeTheme()` 切换后 | 不变 | 不变 | ✅ 均不变 |
| Cookie 选亮色主题 + 刷新页面 | 仍为 `"dark"` | 更新为亮色主题背景色 HEX | ✅ 可证实分叉 |
| PWA manifest | 不涉及 | = 默认主题背景色（启动时固定） | ✅ 与运行时选择无关 |

**分叉是源码层面的客观事实**：用户选择亮色主题并刷新页面后，`color-scheme` 仍然声明 dark（浏览器原生控件按暗色渲染），但 `theme-color` 已是亮色值（浏览器地址栏/标题栏按亮色渲染）。

---

## 四、源码可证实：自定义样式覆盖的级联机制

### 4.1 四级样式加载顺序（源码可证实）

[document.html#L25-L29](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L25-L29)：

```html
<!-- 第 1 层：内置 bundle.css -->
<link rel="stylesheet" href='{{ .App.StaticAssetPath "css/bundle.css" }}'>
<!-- 第 2 层：YAML 主题配置生成的内联变量 -->
<style id="theme-style">{{ .Request.Theme.CSS }}</style>
<!-- 第 3 层（可选）：用户自定义外部 CSS -->
{{ if .App.Config.Theme.CustomCSSFile }}<link rel="stylesheet" href="{{ .App.Config.Theme.CustomCSSFile }}?v={{ .App.CreatedAt.Unix }}">{{ end }}
<!-- 第 4 层（可选）：document.head 配置的任意 HTML -->
{{ if .App.Config.Document.Head }}{{ .App.Config.Document.Head }}{{ end }}
```

### 4.2 theme-style 仅包含变量（源码可证实）

[theme-style.gotmpl](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/theme-style.gotmpl) 的全部输出仅限于 `:root` 选择器内的 CSS 变量，不包含任何具体组件的样式规则。

### 4.3 内置 CSS 已有非 `:root` 的高特异性覆盖（源码可证实）

[utils.css#L379-L381](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/css/utils.css#L379-L381)：

```css
:root:not([data-scheme=light]) .flat-icon {
    filter: invert(1);
}
```

特异性为 **(0,2,1)**，远高于 `:root` 的 (0,1,0)。这从源码层面证实：用户自定义 CSS 要覆盖此类规则，**不能只写 `:root`**，需要匹配或超越相应的特异性。

### 4.4 document.head 可注入任意内容（源码可证实）

`Document.Head` 字段类型为 `string`，模板中直接输出（无 HTML 转义），因此可以包含 `<style>`、`<link>`、`<script>` 等任意 HTML 标签。

---

## 五、可证实 vs 推断 最终总览表

| 结论 | 分类 | 依据 |
|------|------|------|
| default-dark 仅在默认主题=空结构体时不显示 | ✅ 可证实 | 1.4 节 5 种子场景的逐行验算 |
| default-light 无条件注册 | ✅ 可证实 | [glance.go#L114-L122](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/glance.go#L114-L122) |
| 用户 preset 可覆盖同名内置 preset 的数据 | ✅ 可证实 | [config.go#L583-L600](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/config.go#L583-L600) Merge 逻辑 |
| `<meta color-scheme>` 恒为 "dark" | ✅ 可证实 | [document.html#L15](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L15) 字面量 |
| `<meta theme-color>` 首次渲染随主题变化 | ✅ 可证实 | [document.html#L21](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L21) + [theme.go#L69-L73](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/theme.go#L69-L73) |
| JS 切换主题时 theme-color 不更新 | ✅ 可证实 | [page.js#L670-L692](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/js/page.js#L670-L692) 无相关代码 |
| PWA manifest theme_color = 默认主题固定值 | ✅ 可证实 | [manifest.json#L5](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/manifest.json#L5) + [glance.go#L220-L228](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/glance.go#L220-L228) |
| color-scheme 与 theme-color 行为分叉是事实 | ✅ 可证实 | 上表各维度证据组合 |
| 四级样式覆盖的加载顺序 | ✅ 可证实 | [document.html#L25-L29](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/templates/document.html#L25-L29) |
| 用户仅覆盖 `:root` 无法覆盖 (0,2,1) 特异性规则 | ✅ 可证实 | [utils.css#L379-L381](file:///d:/fz/0601/solo-dogfeeding/code/146-glance/internal/glance/static/css/utils.css#L379-L381) + CSS 特异性规则 |
| default-dark 条件表达式的设计意图 | ⚠️ 推断 | 2.1 节，代码中无注释支撑 |
| color-scheme 硬编码 dark 的原因 | ⚠️ 推断 | 2.2 节，代码中无注释支撑 |
| theme-color 不随 JS 更新的原因 | ⚠️ 推断 | 2.3 节，代码中无注释支撑 |
| 自定义 CSS 的典型用户用例 | ⚠️ 推断 | 不在代码库范围内 |

---

## 六、端到端可证实链路图

```
YAML 配置 → config.go UnmarshalYAML
    │
    ▼
应用启动预设注册 glance.go#L104-L128
    │
    ├─ default-dark:
    │   (ok && !SameAs(默认, 用户D)) || !SameAs(默认, 空)
    │   ├─ ✅ 可证实：5 种子场景真值表
    │   └─ ⚠️ 推断：设计意图
    │
    ├─ default-light: ✅ 可证实：无条件 append
    │
    └─ Merge(用户presets):
        ✅ 可证实：用户同名 preset 覆盖内置 preset 数据
    │
    ▼
每个 preset init() theme.go#L56-L76
    │
    ├─ ✅ CSS 变量字符串（仅变量，无规则）
    ├─ ✅ PreviewHTML 按钮
    └─ ✅ BackgroundColorAsHex（nil → "#151519"）
    │
    ▼
document.html 渲染
    │
    ├─ <html data-scheme="{{Light?light:dark}}">       ✅
    ├─ <meta color-scheme content="dark">               ✅ 恒为 dark
    ├─ <meta theme-color content="{{BackgroundColorAsHex}}">  ✅ 随请求主题
    │
    ├─ 第1层 bundle.css                                 ✅
    ├─ 第2层 <style id="theme-style"> CSS 变量          ✅
    ├─ 第3层 <link custom-css-file>（可选）             ✅
    └─ 第4层 document.head（可选，任意内容）             ✅
    │
    ▼
浏览器 JS 初始化 page.js initThemePicker
    │
    ├─ ✅ 克隆移动端 .theme-choices → 桌面端
    └─ ✅ 绑定 preset 按钮 click 事件
    │
    ▼
changeTheme() page.js#L670-L692
    │
    ├─ ✅ POST /api/set-theme/{key}
    ├─ ✅ 后端返回 CSS + X-Scheme header（不返回 HEX）
    ├─ ✅ 更新 #theme-style 内容
    ├─ ✅ 更新 data-theme / data-scheme
    ├─ ✅ 不更新 <meta theme-color>
    └─ ✅ 不更新 <meta color-scheme>
```
