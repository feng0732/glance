# Glance Search Widget 多引擎切换机制分析

## 概述

搜索栏的多引擎切换通过"Bang 快捷词"机制实现。整体流程分为四层：

1. **后端配置层** — Go 代码解析 YAML 配置，预处理 URL 模板，渲染 HTML
2. **数据传递层** — 模板将配置序列化为 DOM dataset 属性
3. **前端交互层** — JavaScript 监听输入变化，识别快捷词，维护状态
4. **跳转组装层** — 按 Enter 时根据当前状态组装最终 URL 并导航

---

## 一、快捷词解析（Bang Parsing）

### 1.1 数据结构

后端定义 Bang 结构位于 [widget-search.go](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/widget-search.go#L11-L15)：

```go
type SearchBang struct {
    Title    string   // 显示名（如 "YouTube"）
    Shortcut string   // 快捷词（如 "!yt"）
    URL      string   // URL 模板，含 {QUERY} 占位符
}
```

`searchWidget` 结构体包含 `Bangs []SearchBang` 字段，YAML 键为 `bangs`。

### 1.2 配置示例

```yaml
- type: search
  bangs:
    - title: YouTube
      shortcut: "!yt"
      url: https://www.youtube.com/results?search_query={QUERY}
    - title: GitHub
      shortcut: "!gh"
      url: https://github.com/search?q={QUERY}
```

### 1.3 前端识别逻辑

核心解析函数是 `handleInput`，位于 [page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L170-L184)：

```javascript
const handleInput = (event) => {
    const value = event.target.value.trim();

    // 精确匹配：输入恰好等于快捷词
    if (value in bangsMap) {
        changeCurrentBang(bangsMap[value]);
        return;
    }

    // 前缀匹配：第一个词 + 空格 + 查询内容
    const words = value.split(" ");
    if (words.length >= 2 && words[0] in bangsMap) {
        changeCurrentBang(bangsMap[words[0]]);
        return;
    }

    changeCurrentBang(null);
};
```

**匹配规则（优先级从高到低）：**

| 场景 | 输入示例 | 行为 |
|------|---------|------|
| 精确匹配 | `!yt` | 激活 YouTube Bang，Title 显示，但尚无查询词 |
| 前缀匹配 | `!yt cats` | 激活 YouTube Bang，查询词为 `cats` |
| 无匹配 | `hello world` | 重置为 `null`，使用默认搜索引擎 |

`bangsMap` 是在初始化时构建的字典：`shortcut → DOMElement`，见 [page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L118-L121)。

### 1.4 状态切换

`changeCurrentBang` 函数（[page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L165-L168)）：

```javascript
const changeCurrentBang = (bang) => {
    currentBang = bang;
    bangElement.textContent = bang != null ? bang.dataset.title : "";
}
```

- 更新闭包变量 `currentBang`
- 将标题渲染到 `.search-bang` DOM 元素（为空时隐藏，CSS 规则见 [widget-search.css](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/css/widget-search.css#L77-L79)）

---

## 二、默认引擎选择（Default Engine）

### 2.1 预设引擎表

后端内置 6 种搜索引擎，位于 [widget-search.go](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/widget-search.go#L34-L41)：

```go
var searchEngines = map[string]string{
    "duckduckgo": "https://duckduckgo.com/?q={QUERY}",
    "google":     "https://www.google.com/search?q={QUERY}",
    "bing":       "https://www.bing.com/search?q={QUERY}",
    "perplexity": "https://www.perplexity.ai/search?q={QUERY}",
    "kagi":       "https://kagi.com/search?q={QUERY}",
    "startpage":  "https://www.startpage.com/search?q={QUERY}",
}
```

### 2.2 初始化流程

`initialize()` 方法（[widget-search.go](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/widget-search.go#L43-L74)）：

```go
func (widget *searchWidget) initialize() error {
    // 步骤1：默认值 — 未配置时使用 duckduckgo
    if widget.SearchEngine == "" {
        widget.SearchEngine = "duckduckgo"
    }

    // 步骤2：名称解析 — 如果是预设键，替换为完整 URL
    if url, ok := searchEngines[widget.SearchEngine]; ok {
        widget.SearchEngine = url
    }
    // 否则用户可以直接写自定义 URL，如 https://custom.search?q={QUERY}

    // 步骤3：URL 模板转义 — 将 {QUERY} 替换为 !QUERY!
    widget.SearchEngine = convertSearchUrl(widget.SearchEngine)
    ...
}
```

**选择优先级：**
1. YAML 中 `search-engine` 字段（可以是预设名或自定义 URL）
2. 空值 → 默认 `duckduckgo`

### 2.3 为什么用 `!QUERY!` 而不是 `{QUERY}`

`convertSearchUrl` 函数注释（[widget-search.go](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/widget-search.go#L28-L32)）：

> Go's template is being stubborn and continues to escape the curlies in the URL regardless of what the type of the variable is so this is my way around it

Go 模板引擎会过度转义 `{}`，因此后端先用 `!QUERY!` 占位，前端 JavaScript 再执行最终替换。

---

## 三、输入状态（Input State）

### 3.1 焦点管理

事件监听绑定在 `focus` / `blur` 上（[page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L186-L193)）：

```javascript
inputElement.addEventListener("focus", () => {
    document.addEventListener("keydown", handleKeyDown);
    document.addEventListener("input", handleInput);
});
inputElement.addEventListener("blur", () => {
    document.removeEventListener("keydown", handleKeyDown);
    document.removeEventListener("input", handleInput);
});
```

**设计意图：** 只有输入框获得焦点时才监听全局键盘事件，避免与页面其他快捷键冲突。

### 3.2 快捷键聚焦

全局 `S` 键聚焦（[page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L195-L201)）：

```javascript
document.addEventListener("keydown", (event) => {
    if (['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName)) return;
    if (event.code != "KeyS") return;

    inputElement.focus();
    event.preventDefault();
});
```

条件：当前焦点不在输入框/文本域内。同时 `<kbd>S</kbd>` 元素也支持点击聚焦（[page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L203-L205)）。

### 3.3 键盘事件处理

`handleKeyDown` 处理三种按键（[page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L123-L163)）：

| 按键 | 行为 |
|------|------|
| `Escape` | 输入框失焦 |
| `Enter` | 执行搜索跳转（见下一节） |
| `ArrowUp` | 恢复上一次查询（从 `lastQuery` 变量） |

### 3.4 状态变量

每个搜索栏闭包内维护以下状态（[page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L113-L116)）：

```javascript
const bangsMap = {};    // 快捷词 → Bang DOM 元素映射
let currentBang = null; // 当前激活的 Bang
let lastQuery = "";     // 上一次成功提交的查询词
```

---

## 四、跳转组装（URL Assembly & Navigation）

### 4.1 Enter 键完整流程

位于 [page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L129-L157)：

```javascript
if (event.key == "Enter") {
    const input = inputElement.value.trim();
    let query;
    let searchUrlTemplate;

    // 步骤1：选择引擎 URL 模板
    if (currentBang != null) {
        // 使用 Bang 的 URL，截取掉快捷词 + 空格部分
        query = input.slice(currentBang.dataset.shortcut.length + 1);
        searchUrlTemplate = currentBang.dataset.url;
    } else {
        // 使用默认搜索引擎
        query = input;
        searchUrlTemplate = defaultSearchUrl;
    }

    // 步骤2：空输入保护 — 默认引擎且无输入则忽略
    if (query.length == 0 && currentBang == null) {
        return;
    }

    // 步骤3：组装最终 URL
    const url = searchUrlTemplate.replace("!QUERY!", encodeURIComponent(query));

    // 步骤4：决定打开方式（当前页 / 新标签）
    if (newTab && !event.ctrlKey || !newTab && event.ctrlKey) {
        window.open(url, target).focus();
    } else {
        window.location.href = url;
    }

    // 步骤5：保存记录并清空
    lastQuery = query;
    inputElement.value = "";

    return;
}
```

### 4.2 查询截取逻辑

当 `currentBang != null` 时：

```
输入:  "!yt cats dogs"
Shortcut: "!yt"  (长度 3)
slice(3 + 1) = slice(4) → "cats dogs"
```

`+1` 是为了去掉快捷词与查询词之间的**空格**。

### 4.3 URL 模板替换

后端把 `{QUERY}` 换成 `!QUERY!`（见 [widget-search.go](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/widget-search.go#L28-L32)），前端用 JS 做最终替换：

```
模板:  https://www.youtube.com/results?search_query=!QUERY!
query: "cats dogs"
URL:   https://www.youtube.com/results?search_query=cats%20dogs
```

`encodeURIComponent` 负责 URL 编码（空格 → `%20`，特殊字符转义等）。

### 4.4 新标签页逻辑（XOR 模式）

配置项 `new-tab`（YAML）控制默认行为：

```javascript
// 异或逻辑：newTab 配置与 ctrlKey 按键不一致时 → 新标签
if (newTab && !event.ctrlKey || !newTab && event.ctrlKey) {
    window.open(url, target).focus();  // 新标签
} else {
    window.location.href = url;        // 当前页
}
```

真值表：

| `new-tab` 配置 | Ctrl 按下 | 结果 |
|---------------|-----------|------|
| `false`（默认） | 否 | 当前页 |
| `false` | 是 | 新标签 |
| `true` | 否 | 新标签 |
| `true` | 是 | 当前页 |

即 **Ctrl 键始终反转默认行为**。`target` 参数（默认 `_blank`）传递给 `window.open()`。

### 4.5 空输入保护

```javascript
if (query.length == 0 && currentBang == null) {
    return;
}
```

只在"默认引擎 + 空查询"时拦截。**Bang 模式下即使 query 为空也会跳转**——例如用户输入 `!yt` 直接按 Enter，会访问 `https://www.youtube.com/results?search_query=`（空搜索），这可能导致意外行为，属于边界设计取舍。

---

## 五、端到端数据流

```
YAML 配置
   │
   ▼
widget-search.go:initialize()
   ├─ 设置默认 search-engine = "duckduckgo"
   ├─ 查找预设表 → 展开为 URL
   ├─ convertSearchUrl(): {QUERY} → !QUERY!
   └─ Bangs 同样做 URL 转义
   │
   ▼
search.html 模板渲染
   ├─ data-default-search-url = 处理后的 URL
   ├─ data-new-tab, data-target
   └─ 每个 Bang 渲染为 <input type="hidden" data-shortcut data-title data-url>
   │
   ▼
page.js:setupSearchBoxes()
   ├─ 读取 dataset → defaultSearchUrl, newTab, target
   ├─ 构建 bangsMap: {shortcut: DOMElement}
   ├─ 绑定 focus/blur → 挂载/卸载键盘监听
   ├─ handleInput → 实时匹配快捷词 → changeCurrentBang
   └─ handleKeyDown(Enter)
       ├─ 根据 currentBang 选择 URL 模板
       ├─ 截取 query
       ├─ !QUERY! 替换 + encodeURIComponent
       └─ 新标签/当前页跳转
```

---

## 六、涉及文件清单

| 文件 | 作用 |
|------|------|
| [widget-search.go](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/widget-search.go) | 后端数据结构、默认引擎、初始化、URL 预处理 |
| [search.html](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/templates/search.html) | 模板：渲染 DOM + data 属性 |
| [page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js) (L98-L207) | 前端交互核心：快捷词识别、状态管理、跳转组装 |
| [widget-search.css](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/css/widget-search.css) | 样式：Bang 标签动画、空状态隐藏 |
| [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/widget.go) (L66-L67) | 小部件类型注册 |
| [configuration.md](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/docs/configuration.md) (L1183-L1281) | 用户文档 |

---

## 七、隐藏的状态机边界分析

### 7.1 Bang 激活后状态为何会"幽灵保留"

这是最隐蔽的边界。关键差异在于 **DOM `input` 事件的触发条件**：

> 用户交互（键盘输入、粘贴、剪切、删除等）会触发 `input` 事件；
> **JavaScript 直接设置 `element.value = "..."` 不会触发 `input` 事件。**

看提交后的代码（[page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L153-L154)）：

```javascript
lastQuery = query;
inputElement.value = "";   // ← JS 直接赋值，不触发 input 事件
```

**完整的状态残留场景：**

| 步骤 | 操作 | `inputElement.value` | `currentBang` | `.search-bang` 显示 | 触发事件 |
|------|------|---------------------|---------------|---------------------|---------|
| 1 | 用户输入 `!yt cats` | `!yt cats` | `!yt` Bang | YouTube | input → handleInput 匹配 |
| 2 | 用户按 Enter | （处理中） | `!yt` Bang | YouTube | keydown → handleKeyDown |
| 3 | 代码执行 `.value = ""` | `""` | **仍为 `!yt` Bang** | **仍显示 YouTube** | ❌ 无事件 |

此时用户看到的输入框是空的，但右侧依然显示"YouTube"标签，`currentBang` 闭包变量仍然指向 Bang 对象——视觉和内部状态都**残留**了上一次的引擎选择。

用户在空输入框中再次按 Enter 会发生什么？回到 [page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L141-L143) 的空输入保护：

```javascript
if (query.length == 0 && currentBang == null) {
    return;
}
```

条件是 `query.length == 0 **且** currentBang == null`。由于 `currentBang` 残留为非 null，保护不生效，代码继续执行：

```javascript
query = input.slice(currentBang.dataset.shortcut.length + 1);
//      "".slice(3 + 1) = "".slice(4) = ""
const url = searchUrlTemplate.replace("!QUERY!", encodeURIComponent(""));
//      https://www.youtube.com/results?search_query=
```

最终跳转到 YouTube 的**空搜索页面**。这是状态残留导致的直接后果。

---

### 7.2 清空输入：两种清空方式的不同命运

#### 方式 A：用户手动清空（Backspace / Delete / Ctrl+A 再删）

用户逐字删除 `!yt cats` 直到为空，每一次按键都会触发 `input` 事件 → `handleInput`。

最后一次删除后 `value = ""`，trim 后仍为 `""`，执行：

```javascript
// "" in bangsMap → false（空字符串不可能是合法 shortcut，后端 initialize 会校验 shortcut 非空）
const words = "".split(" ");     // [""]
words.length >= 2                // false（只有一个空字符串元素）
changeCurrentBang(null);         // ✅ 状态被正确重置
```

**结果：** `currentBang = null`，`.search-bang` 隐藏。

#### 方式 B：代码赋值清空（Enter 提交后、ArrowUp 恢复后内部清空）

如 7.1 所述，`inputElement.value = ""` 不触发 `input` 事件，`handleInput` 不执行，状态残留。

#### 方式 C：form reset（如果存在的话）

如果搜索栏外层有 `<form>` 并被 reset，会触发 `input` 事件吗？在现代浏览器中 `form.reset()` 对每个 input 设置 `defaultValue`，**同样不触发 `input` 事件**，也会导致状态残留。不过此项目中搜索栏并未包裹在 `<form>` 内（见 [search.html](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/templates/search.html#L5-L23)），此路径不生效。

---

### 7.3 ArrowUp 恢复：引擎标记的"滞后性"

ArrowUp 处理代码（[page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L159-L162)）：

```javascript
if (event.key == "ArrowUp" && lastQuery.length > 0) {
    inputElement.value = lastQuery;   // ← JS 直接赋值，不触发 input 事件
    return;
}
```

又是 JS 直接赋值。**ArrowUp 恢复后，`currentBang` 不会跟随恢复的内容自动更新**。

**场景推演：**

| 步骤 | 操作 | `inputElement.value` | `currentBang` | 说明 |
|------|------|---------------------|---------------|------|
| 1 | 用户输入 `!yt cats`，按 Enter | `""` | `!yt` Bang（残留） | 见 7.1 |
| 2 | 用户按 Backspace 手动清空 | `""` | `null` | handleInput 重置，状态干净 |
| 3 | 用户按 ArrowUp | `cats` | **仍为 `null`** ❌ | lastQuery = "cats"，直接赋值不触发 input |
| 4 | 用户按 Enter | — | `null` | 走默认搜索引擎，而不是 YouTube |

第 3 步用户看到的输入框内容是 `cats`，但由于上次提交时 `lastQuery = query`（即只存了"cats"不含快捷词，见 [page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L153)），ArrowUp 只恢复了查询词本体，`currentBang` 也停留在 `null`。**视觉上无法区分这次 ArrowUp 恢复是要走 YouTube 还是默认引擎，但代码逻辑已经走默认引擎。**

更矛盾的场景：

| 步骤 | 操作 | `inputElement.value` | `currentBang` | `.search-bang` 显示 |
|------|------|---------------------|---------------|---------------------|
| 1 | 用户输入 `!yt cats`，按 Enter | `""` | `!yt` Bang（残留） | YouTube |
| 2 | 用户按 ArrowUp | `cats` | **仍为 `!yt` Bang** | **仍显示 YouTube** |
| 3 | 用户按 Enter | — | `!yt` Bang | 跳 YouTube 搜索 `cats` |

此时输入框显示的是纯 `cats`（没有 `!yt` 前缀），但右侧依然显示"YouTube"标签，最终跳转 YouTube——**输入框内容与实际引擎选择出现语义脱节**。用户如果没注意右侧标签，会以为搜索了默认引擎。

根本原因：`lastQuery` 只保存了截取后的纯查询词（`query`），没有保存对应的 `currentBang` 快照。ArrowUp 恢复是单边的，只恢复文本，不恢复引擎标记。

---

### 7.4 全局监听的副作用：哪些事件会污染状态

事件监听注册如下（[page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L186-L201)）：

```javascript
inputElement.addEventListener("focus", () => {
    document.addEventListener("keydown", handleKeyDown);   // 挂在 document
    document.addEventListener("input", handleInput);       // 挂在 document
});

// 这个监听器永久挂在 document，不受 focus 影响
document.addEventListener("keydown", (event) => {
    if (['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName)) return;
    if (event.code != "KeyS") return;
    inputElement.focus();
    event.preventDefault();
});
```

#### 7.4.1 永久全局 S 键监听

第二个 `keydown` 监听器**始终存在于 document 上**，不受 `focus`/`blur` 生命周期控制。它有两个守卫：

```javascript
if (['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName)) return;
if (event.code != "KeyS") return;
```

- `event.code != "KeyS"`：只拦截字母 S，不区分大小写（`code` 是物理按键，不受 Shift 影响）
- `activeElement` 守卫：当前焦点是 `<input>` 或 `<textarea>` 时放行

**边界问题：** 如果页面上还有其他可编辑元素不在 `INPUT`/`TEXTAREA` 白名单中——例如 `contenteditable="true"` 的 `<div>`（Glance 当前没有，但扩展可能引入）——在其中编辑时按 S 会被抢走焦点并跳转到搜索框。

另一个问题：多个搜索栏实例的情况。`setupSearchBoxes` 会为每个 `.search` 都注册一个独立的全局 S 键监听器。页面如果有两个搜索栏（虽然 UI 设计上不太可能），按一次 S 会**同时 focus 两个输入框**，结果是后者（DOM 中后出现的）获得焦点，前者被覆盖。

#### 7.4.2 focus 期间挂在 document 的 input 监听

`handleInput` 挂在 `document` 而不是 `inputElement` 上。`input` 事件冒泡，所以搜索框内的输入能被捕获。但在搜索框聚焦期间，**页面上任何其他 input 的输入事件也会冒泡到 document 并触发 handleInput**。

推演这个路径：

```
handleInput = (event) => {
    const value = event.target.value.trim();  // event.target 是事件源
    ...
}
```

`handleInput` 使用的是 `event.target.value`，不是 `inputElement.value`。所以如果用户在搜索框聚焦的同时（比如分屏或多光标）在另一个 input 里输入，`event.target` 会是那个 input，`handleInput` 依然会去匹配 `bangsMap`，然后调用 `changeCurrentBang` 更新**当前这个搜索栏**的状态——**跨元素的状态污染**。

不过在实际使用中，搜索框一旦 focus 就是当前活动元素，用户同时操作两个 input 的概率极低。但从代码正确性角度这是隐患。

#### 7.4.3 focus 期间挂在 document 的 keydown 监听

`handleKeyDown` 同样挂在 `document`，搜索框聚焦期间**整个页面的所有键盘事件**都会被它处理：

```javascript
if (event.key == "Escape") {
    inputElement.blur();   // 任意地方按 Escape 都会让搜索框失焦
    return;
}
if (event.key == "Enter") { ... }  // 在别处按 Enter 也触发搜索
if (event.key == "ArrowUp" && lastQuery.length > 0) {
    inputElement.value = lastQuery;  // 在别处按 ArrowUp 也改搜索框值
    return;
}
```

`Enter` 和 `ArrowUp` 没有检查 `event.target === inputElement`，所以搜索框聚焦期间，用户在页面上任何位置按 Enter 都会触发搜索跳转，按 ArrowUp 都会把 lastQuery 塞进搜索框。

**实际影响场景：**
- 搜索框聚焦时用鼠标点了页面上的按钮（不会 blur 搜索框，因为按钮不会抢 focus），然后按 Enter 确认——触发了搜索跳转而不是按钮点击
- 搜索框聚焦时打开了一个非 input 的弹窗/下拉，在其中按 Escape 同时关闭弹窗和让搜索框失焦（这个其实影响不大）

修复方向很明确：在 `handleKeyDown` 和 `handleInput` 开头加守卫 `if (event.target !== inputElement) return;`，或者干脆把监听直接挂在 `inputElement` 上而不是 `document`。

---

### 7.5 边界汇总表

| 边界场景 | 状态表现 | 根因 |
|---------|---------|------|
| Enter 提交后再按 Enter | 跳转残留 Bang 的空搜索页 | `.value=""` 不触发 input 事件，`currentBang` 残留；空输入保护只拦截 `currentBang==null` 情况 |
| 手动 Backspace 清空 | 状态正确重置为默认引擎 | 用户操作触发 input 事件 → `changeCurrentBang(null)` |
| ArrowUp 恢复查询 | 引擎标记不跟随内容更新 | `.value=lastQuery` 不触发 input 事件；`lastQuery` 只存纯查询词不存引擎快照 |
| ArrowUp 恢复 + Bang 残留 | 输入内容无快捷词但仍跳 Bang 引擎 | 输入视觉与引擎状态语义脱节 |
| 全局 S 键 + contenteditable | 在可编辑 div 中按 S 被抢走焦点 | 白名单仅包含 INPUT/TEXTAREA |
| 多搜索栏实例 + S 键 | 只有最后一个实例获得焦点 | 每个实例独立注册全局 S 监听，无互斥 |
| focus 期间别处按 Enter | 意外触发搜索跳转 | `handleKeyDown` 挂在 document 且无 `event.target` 守卫 |
| focus 期间别处按 ArrowUp | 意外恢复 lastQuery 到搜索框 | 同上 |
| focus 期间其他 input 输入 | 可能错误触发当前搜索栏的 Bang 切换 | `handleInput` 挂在 document 且用 `event.target` 取值 |

---

## 八、输入框事件生命周期详解

### 8.1 全局监听的挂载/卸载时机

先回顾监听注册代码（[page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L186-L201)）：

```javascript
// 第一组：受 focus/blur 控制的临时监听
inputElement.addEventListener("focus", () => {
    document.addEventListener("keydown", handleKeyDown);   // A
    document.addEventListener("input", handleInput);       // B
});
inputElement.addEventListener("blur", () => {
    document.removeEventListener("keydown", handleKeyDown);
    document.removeEventListener("input", handleInput);
});

// 第二组：永久存在的全局监听
document.addEventListener("keydown", (event) => {          // C
    if (['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName)) return;
    if (event.code != "KeyS") return;
    inputElement.focus();
    event.preventDefault();
});
```

**三组监听的生命周期对比：**

| 监听 | 挂载时机 | 卸载时机 | 是否永久存在 |
|------|---------|---------|------------|
| A：`handleKeyDown`（document） | 搜索 input 获得 `focus` | 搜索 input 触发 `blur` | ❌ 临时 |
| B：`handleInput`（document） | 搜索 input 获得 `focus` | 搜索 input 触发 `blur` | ❌ 临时 |
| C：S 键聚焦（document） | `setupSearchBoxes()` 初始化时 | 从不（除非销毁 DOM） | ✅ 永久 |

因此，"全局监听"分两类：**C 永远在**，**A 和 B 只在搜索框聚焦时存在**。

---

### 8.2 浏览器焦点行为基础

要理解副作用是否真实发生，必须先搞清楚浏览器的焦点模型：

#### 8.2.1 可聚焦 vs 不可聚焦元素

浏览器默认的可聚焦元素（`tabindex >= 0` 或原生可交互）：

| 元素 | 默认可聚焦 | 点击时是否获得焦点 |
|------|-----------|------------------|
| `<input>`（text/password/search 等） | ✅ | ✅ |
| `<input type="hidden">` | ❌ | ❌ |
| `<input type="radio/checkbox">` | ✅ | ✅ |
| `<button>` | ✅ | ✅ |
| `<textarea>` | ✅ | ✅ |
| `<a href>` | ✅ | ✅ |
| `<select>` | ✅ | ✅ |
| `<kbd>`（无 tabindex） | ❌ | ❌ |
| `<div> / <span> / <svg>（无 tabindex）` | ❌ | ❌ |
| `tabindex="-1"` 的任意元素 | ❌（Tab 键不能到达） | ✅（点击/JS 可聚焦） |

Glance 主页面（非登录页）上出现的 input（[page.html](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/templates/page.html#L61-L63)）：

```html
<input type="radio"    class="mobile-navigation-input">
<input type="checkbox" class="mobile-navigation-page-links-input">
```

它们**是可聚焦的**，点击时会获得焦点。

#### 8.2.2 焦点切换的事件顺序

点击一个可聚焦元素时的事件序列：

```
1. mousedown  （在被点击元素上）
2. blur       （在原焦点元素上 — 如果被点击元素可聚焦）
3. focus      （在被点击元素上 — 如果可聚焦）
4. mouseup    （在被点击元素上）
5. click      （在被点击元素上）
```

关键点：**`blur` 在 `mousedown` 之后、`mouseup` 之前同步触发**。如果点击的是**不可聚焦元素**，浏览器通常**不会触发 blur**，焦点保留在原元素上。

---

### 8.3 场景逐一推演

#### 8.3.1 场景：聚焦状态下点击 Group Widget 的 Tab 按钮

Group Widget 的标题按钮（[group.html](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/templates/group.html#L9)）：

```html
<button class="widget-group-title" ...>
```

按钮是原生可聚焦元素。

**事件序列：**

| 步骤 | 事件 | 焦点位置 | A/B 监听是否存在 | 影响 |
|------|------|---------|-----------------|------|
| 初始 | — | 搜索 input | ✅ 存在 | — |
| 1 | 鼠标在 button 上按下 | 搜索 input（尚未变） | ✅ 存在 | — |
| 2 | `blur` 触发（搜索 input） | 过渡中 | 🔴 blur 回调执行，A/B **立即被移除** | handleKeyDown 和 handleInput 失效 |
| 3 | `focus` 触发（button） | button | ❌ 已移除 | — |
| 4 | 用户在 button 上按 Enter | button | ❌ 已移除 | 只触发 button 默认行为，**不会触发搜索跳转** |

**结论：第七章中"点击按钮后按 Enter 触发搜索"的担忧是不成立的。** 因为按钮点击在 mousedown 阶段就同步触发了搜索框的 blur，A/B 监听被立即移除，后续 Enter 键不再被 handleKeyDown 捕获。

#### 8.3.2 场景：聚焦状态下切换到其他 input（如移动端导航 radio）

移动端导航的 radio input（[page.html](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/templates/page.html#L61-L63)）是可聚焦元素。

**事件序列：**

| 步骤 | 事件 | 焦点 | A/B 监听 | `activeElement.tagName` | C 监听（S 键）是否生效 |
|------|------|------|---------|------------------------|----------------------|
| 初始 | — | 搜索 input | ✅ 存在 | INPUT | ❌ 不生效（白名单拦截） |
| 1 | 点击 radio | — | — | — | — |
| 2 | 搜索 input `blur` | 过渡中 | 🔴 A/B 被移除 | — | — |
| 3 | radio `focus` | radio | ❌ 已移除 | INPUT | ❌ 不生效（白名单拦截） |
| 4 | 在 radio 上输入字符 | radio | ❌ 已移除 | INPUT | ❌ 不生效 |

**额外验证 C 监听（永久 S 键）：** radio 聚焦时 `activeElement.tagName === "INPUT"`，被白名单拦截，S 键不会抢焦点。

**结论：第七章中"其他 input 的输入冒泡到 document 触发 handleInput"的担忧也是不成立的。** 因为其他 input 获得焦点的瞬间，搜索 input 已经 blur，A/B 监听已经被 removeEventListener 卸载，根本不会收到事件。

#### 8.3.3 场景：聚焦状态下点击搜索栏内部的 `<kbd>S</kbd>` 标签

搜索栏内的 kbd 元素（[search.html](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/templates/search.html#L22)）：

```html
<kbd class="hide-on-mobile" title="Press [S] to focus the search input">S</kbd>
```

`<kbd>` 原生不可聚焦（没有 tabindex）。代码还专门绑定了 mousedown（[page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L203-L205)）：

```javascript
kbdElement.addEventListener("mousedown", () => {
    requestAnimationFrame(() => inputElement.focus());
});
```

为什么要用 `mousedown` + `requestAnimationFrame`？

- 如果只用 `click`：在 mousedown → blur（如果可聚焦）→ focus → mouseup → click 的流程中，搜索框可能先失焦再聚焦，产生闪烁
- 但 `<kbd>` 不可聚焦，mousedown 不会触发搜索框 blur
- 用 `requestAnimationFrame` 是为了让浏览器在**下一帧**执行 focus，避免 mousedown 处理中立即 focus 可能带来的浏览器默认行为冲突
- 如果搜索框本来就是聚焦状态，再次 focus 是 no-op

**事件序列：**

| 步骤 | 事件 | 焦点 | A/B 监听 |
|------|------|------|---------|
| 初始 | — | 搜索 input | ✅ 存在 |
| 1 | kbd 上 mousedown | 搜索 input（kbd 不可聚焦，不触发 blur） | ✅ 存在 |
| 2 | rAF 回调：`inputElement.focus()` | 搜索 input（无变化） | ✅ 存在 |
| 3 | mouseup / click | 搜索 input | ✅ 存在 |

**结论：** 点击 kbd 不会丢失焦点，A/B 监听持续存在。

#### 8.3.4 场景：聚焦状态下点击页面空白区域（普通 div / svg 等）

Glance 页面的大部分装饰性元素（svg 图标、div 容器、widget 标题等）没有 tabindex，是不可聚焦的。

**行为取决于浏览器：**
- Chrome / Firefox / Safari：点击不可聚焦元素时**不触发 blur**，焦点保留在搜索 input 上
- 极少数特殊情况下（如点击 `iframe`、浏览器 UI 元素）才会触发 blur

**事件序列（主流浏览器）：**

| 步骤 | 事件 | 焦点 | A/B 监听 | 用户在空白处按 Enter |
|------|------|------|---------|-------------------|
| 1 | 点击空白 div | 搜索 input（不变） | ✅ 存在 | — |
| 2 | 按 Enter | 搜索 input | ✅ 存在 | handleKeyDown 捕获 → **执行搜索跳转** |

**结论：第七章中"点击页面其他位置后按 Enter 触发搜索"在主流浏览器中**只对不可聚焦元素成立**。这确实是真实的副作用——用户点了一下页面空白处想"取消"，焦点没丢，随手按 Enter 反而触发了搜索。

但也要注意：这种情况下 `document.activeElement` 仍然是搜索 input，所以从语义上讲用户仍然在搜索框的上下文中，行为不算严重违反直觉。

#### 8.3.5 场景：聚焦状态下按 Escape 主动失焦

代码（[page.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/page.js#L124-L126)）：

```javascript
if (event.key == "Escape") {
    inputElement.blur();
    return;
}
```

**事件序列：**

| 步骤 | 事件 | 焦点 | A/B 监听 | C 监听（S 键）是否生效 |
|------|------|------|---------|----------------------|
| 初始 | — | 搜索 input | ✅ 存在 | ❌ 白名单 INPUT 拦截 |
| 1 | Escape keydown → `inputElement.blur()` | body/null | blur 回调同步移除 A/B | — |
| 2 | blur 完成 | body/null | ❌ 已移除 | ✅ 生效（activeElement 不再是 INPUT/TEXTAREA） |

**结论：** Escape 后 A/B 完全卸载，S 键恢复可用。用户可以按 S 重新聚焦。

#### 8.3.6 场景：聚焦状态下打开 Popover（鼠标悬停触发）

Glance 的 Popover（[popover.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/popover.js)）默认通过 `mouseenter` 触发，不是点击触发。

**关键代码（[popover.js](file:///d:/fz/0601/solo-dogfeeding/code/142-glance/internal/glance/static/js/popover.js#L104)）：**

```javascript
document.addEventListener("keydown", handleHidePopoverOnEscape);
```

Popover 显示时也会向 document 添加一个**独立的** Escape keydown 监听，与搜索框的 handleKeyDown 互不干扰。

**事件序列（搜索框聚焦 → 鼠标移到链接上 → Popover 出现 → 按 Escape）：**

| 步骤 | 焦点 | 活跃的 document keydown 监听 | Escape 触发结果 |
|------|------|---------------------------|---------------|
| 初始 | 搜索 input | 搜索框的 handleKeyDown（A） | 搜索框 blur，A/B 被移除 |
| Popover 打开 | 搜索 input（mouseenter 不改变焦点） | A + Popover 的 handleHidePopoverOnEscape | **两个监听都会执行**：Popover 关闭 + 搜索框 blur |

**结论：** 搜索框聚焦 + Popover 打开时按 Escape，两个监听都会被触发，Popover 关闭同时搜索框失焦。这是两个独立模块各自向 document 注册监听的叠加效果，但在实际使用中影响很小。

---

### 8.4 哪些副作用只是"表面担忧"

基于上面的推演，现在可以纠正第七章中过度估计的风险：

| 第七章中的担忧 | 是否真实存在 | 原因 |
|---------------|------------|------|
| "点击按钮后按 Enter 触发搜索跳转" | ❌ 不成立 | 按钮可聚焦，点击的 mousedown 阶段同步触发搜索框 blur → A/B 监听被立即移除 |
| "其他 input 的输入冒泡触发 handleInput" | ❌ 不成立 | 其他 input 获得焦点的同时搜索框 blur → A/B 监听被移除，不会收到后续事件 |
| "focus 期间别处按 Enter 意外触发搜索" | ⚠️ 仅限不可聚焦元素 | 只有点击了**不可聚焦**的空白区域/装饰元素后，焦点仍在搜索框，按 Enter 才会触发；点击按钮/其他 input 不会 |
| "focus 期间别处按 ArrowUp 意外恢复" | ⚠️ 同上 | 同上，只有焦点仍在搜索框时才会发生，而此时 activeElement 就是搜索 input，语义上不算"别处" |
| "全局 S 键在 contenteditable 中抢焦点" | ✅ 真实存在，但 Glance 无此场景 | 代码只白名单了 INPUT/TEXTAREA，扩展如果引入 contenteditable 会踩坑 |
| "Popover 的 Escape 与搜索框 Escape 冲突" | ⚠️ 叠加但不冲突 | 两个监听都会执行，只是同时关闭 Popover + 搜索框失焦，无报错 |

**真正需要警惕的只有两个：**
1. **JS 赋值 `.value` 不触发 input 事件导致的状态残留**（第七章 7.1/7.2/7.3）——这是纯逻辑 bug，与浏览器焦点无关
2. **`lastQuery` 只存纯查询词不存引擎快照**——也是纯逻辑问题

其余"跨元素副作用"大部分因为浏览器焦点模型的天然保护（点击可聚焦元素会触发 blur，从而移除监听）而不会实际发生。

---

### 8.5 完整生命周期状态图

```
页面加载 setupSearchBoxes()
   │
   ├─ 永久注册：C 监听（S 键聚焦）挂在 document（永远不卸载）
   │
   ▼
[搜索框未聚焦]
   │  A/B 监听不存在
   │  C 监听生效（按 S 可聚焦搜索框）
   │
   │  用户点击搜索框 / 按 S 键 / JS 调用 focus()
   ▼
[focus 事件触发]
   │
   ├─ document.addEventListener(keydown, handleKeyDown)   ← A 挂载
   ├─ document.addEventListener(input, handleInput)       ← B 挂载
   │
   ▼
[搜索框聚焦中]
   │  A/B 监听均存在
   │  C 监听失效（白名单 INPUT 拦截）
   │
   │  事件处理：
   │  ├─ 用户输入字符 → input 冒泡 → handleInput → 匹配/重置 Bang
   │  ├─ Enter → handleKeyDown → 组装 URL + 跳转（.value="" 不触发 input）
   │  ├─ ArrowUp → handleKeyDown → .value=lastQuery（不触发 input）
   │  └─ Escape → handleKeyDown → inputElement.blur()
   │
   │  焦点转移路径：
   │  ├─ 点击按钮 / 其他 input → mousedown → blur → [搜索框未聚焦]
   │  ├─ 点击不可聚焦元素 → 焦点不变 → 仍在 [搜索框聚焦中]
   │  ├─ Tab 键切走 → blur → [搜索框未聚焦]
   │  └─ Escape → blur → [搜索框未聚焦]
   │
   ▼
[blur 事件触发]
   │
   ├─ document.removeEventListener(keydown, handleKeyDown)  ← A 卸载
   └─ document.removeEventListener(input, handleInput)      ← B 卸载
   │
   ▼
回到 [搜索框未聚焦]
```
