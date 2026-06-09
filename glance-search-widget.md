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
