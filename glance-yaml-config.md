# Glance YAML 配置加载与校验流程

## 整体流程概览

```
CLI 入口
   ↓
parseYAMLIncludes()        ← 递归解析 !include/$include 指令，合并多文件
   ↓
parseConfigVariables()     ← 变量插值（环境变量 / Docker secrets / 文件读取）
   ↓
yaml.Unmarshal()           ← 标准 YAML 反序列化（含自定义 UnmarshalYAML）
   ↓
isConfigStateValid()       ← 逻辑合法性校验
   ↓
widget.initialize()        ← 逐个 Widget 初始化
   ↓
newApplication()           ← 默认值合并 + 应用层二次校验/初始化
```

---

## 1. 配置文件包含（Include 解析）

**核心函数**: [recursiveParseYAMLIncludes](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L246-L303)

### 语法

```yaml
# 两种形式等价
- !include: ./extra-config.yml
$include: ./another-config.yml
```

### 实现机制

1. **正则匹配**: 使用 `(?m)^([ \t]*)(?:-[ \t]*)?(?:!|\$)include:[ \t]*(.+)$` 逐行扫描
   - 捕获缩进前缀（第1组）：用于保持被包含文件内容的 YAML 层级
   - 捕获文件路径（第2组）

2. **递归深度限制**: `CONFIG_INCLUDE_RECURSION_DEPTH_LIMIT = 20`，防止循环引用

3. **相对路径解析**: 以当前文件所在目录为基准拼接路径，再转换为绝对路径

4. **缩进处理**: `prefixStringLines(indent, content)` 给被包含文件的每一行加上外层缩进，保证 YAML 结构正确

5. **已包含文件追踪**: 返回 `map[string]struct{}` 记录所有被包含的绝对路径，供文件监视器使用

**调用入口**:
- `serveApp()`: 服务启动时首次解析
- `configFilesWatcher()`: 文件变更触发重新解析
- `cliIntentConfigValidate` / `cliIntentConfigPrint`: CLI 命令

---

## 2. 变量插值（Variable Interpolation）

**核心函数**: [parseConfigVariables](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L142-L187)

### 支持的变量格式

| 语法 | 类型 | 说明 |
|------|------|------|
| `${API_KEY}` | `env` | 读取环境变量 |
| `${secret:api_key}` | `secret` | 从 `/run/secrets/api_key` 读取 Docker Secret |
| `${readFileFromEnv:PATH_VAR}` | `readFileFromEnv` | 先读环境变量获取文件路径，再读取该文件内容（路径必须是绝对路径）|
| `\${API_KEY}` | 转义 | 反斜杠转义，结果为字面量 `${API_KEY}` |

### 正则模式

```
(^|.)\$\{(?:([a-zA-Z]+):)?([a-zA-Z0-9_-]+)\}
```

- 第1组 `(^|.)`: 前缀字符（用于检测转义符 `\`）
- 第2组 `([a-zA-Z]+)`: 可选的变量类型前缀（env/secret/readFileFromEnv）
- 第3组 `([a-zA-Z0-9_-]+)`: 变量名

### 关键实现细节

- 环境变量名必须匹配 `^[A-Z0-9_]+$`（全大写+下划线+数字），不匹配则保留原样
- 文件内容读取后自动 `TrimSpace` 去除首尾空白
- 使用 `ReplaceAllFunc` 闭包捕获错误，遇到第一个错误立即终止整个替换过程
- **注意**: 目前不区分 YAML 注释，注释中的变量也会被替换（见 TODO 注释）

---

## 3. YAML 反序列化与自定义字段解析

### 3.1 标准反序列化

`yaml.Unmarshal(contents, config)` 将处理后的字节流映射到 `config` 结构体。

结构体定义在 [config.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L30-L69)，主要层级：

```
config
├── Server      (host, port, proxied, assets-path, base-url)
├── Auth        (secret-key, users)
├── Document    (head)
├── Theme       (inline themeProperties, custom-css-file, presets)
├── Branding    (footer, logo, favicon, app-name, etc.)
└── Pages[]     (name, slug, columns, widgets, ...)
```

### 3.2 自定义字段 UnmarshalYAML

所有自定义解析在 [config-fields.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config-fields.go)

#### hslColorField — HSL 颜色解析
```
正则: ^(?:hsla?\()?([\d\.]+)(?: |,)+([\d\.]+)%?(?: |,)+([\d\.]+)%?\)?$
```
支持格式：`hsl(240, 13%, 95%)` 或 `240 13 95` 或 `240,13%,95%`
- H: 0–360, S: 0–100, L: 0–100，超出范围报错

#### durationField — 时间时长解析
```
正则: ^(\d+)(s|m|h|d)$
```
支持：`30s` / `5m` / `2h` / `1d`
- `d` 按 24 小时换算

#### customIconField — 图标解析
支持前缀简写自动映射 CDN URL：

| 前缀 | 说明 | 示例 URL |
|------|------|---------|
| `si:` | Simple Icons (auto-invert) | `https://cdn.jsdelivr.net/npm/simple-icons@latest/icons/github.svg` |
| `mdi:` | Material Design Icons (auto-invert) | `https://cdn.jsdelivr.net/npm/@mdi/svg@latest/svg/home.svg` |
| `di:` | Homarr Dashboard Icons | `https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/rss.svg` |
| `sh:` | Selfhst Icons | `https://cdn.jsdelivr.net/gh/selfhst/icons/svg/home.svg` |
| `auto-invert ` | 前缀修饰，设 AutoInvert=true | — |
| 无前缀 | 直接作为 URL 使用 | — |

文件后缀默认 `.svg`，支持显式指定 `.png`。

#### proxyOptionsField — 代理选项
支持两种写法：

```yaml
# 简写（仅 URL）
proxy: http://proxy:8080

# 完整形式
proxy:
  url: http://proxy:8080
  allow-insecure: true
  timeout: 30s
```

解析时先尝试按字符串解码（简写），失败则按结构体解码（完整形式）。最终构造 `*http.Client`。

#### queryParametersField — 查询参数
将 YAML 中的值统一归一化为 `map[string][]string`，支持类型：
- 单个值（string / int / float / bool）→ 单元素数组
- `[]string` → 直接追加
- `[]any` → 逐项转换后追加
- 不支持的类型直接返回错误

#### orderedYAMLMap — 有序映射
自定义泛型结构体，内部用 `[]K` 保序 + `map[K]V` 存值，详见下文「默认值合并」。

#### widgets — Widget 列表反序列化

[widgets.UnmarshalYAML](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L95-L124) 的执行流程：
1. 先将外层节点 Decode 为 `[]yaml.Node`（每个 widget 为一个独立节点）
2. 对每个 widget 节点，先 Decode 出临时结构体 `meta` 中的 `type` 字段
3. 调用 `newWidget(meta.Type)` 通过 switch 工厂创建具体 widget 结构体实例，并分配全局自增 ID
4. 将完整 widget 节点 Decode 到具体结构体上

反序列化共有 **5 个报错分支**，定位信息的处理各不相同，详见第 11.4 节。

---

## 4. 配置状态校验

**核心函数**: [isConfigStateValid](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L451-L537)

### 校验清单

| 校验项 | 规则 |
|--------|------|
| Pages 非空 | 至少配置 1 个页面 |
| Auth 一致性 | 配置了 Users 则必须有 SecretKey |
| 用户名 | 不能为空，至少 3 字符 |
| 用户密码 | password 或 password-hash 二选一，明文密码至少 6 字符 |
| Assets 路径 | 若配置则目录必须存在 |
| 页面 name | 不能为空 |
| 页面 width | `wide` / `slim` / `default` / 空 |
| 页面列数 | slim 页面 ≤ 2 列，其他 ≤ 3 列 |
| 列 size | 只能是 `small` 或 `full` |
| full 列数量 | 每页必须恰好 1 或 2 个 full 列 |

### 设计说明
- 此阶段**只读校验**，不修改数据
- `newApplication()` 中还会进行**二次校验+数据修改**（如 slug 生成、密码哈希、主题初始化等）
- 代码 TODO 明确说明希望将两处校验合并为单一阶段

---

## 5. 阶段一内部：newConfigFromYAML 的精确执行顺序与状态变化

**核心函数**: [newConfigFromYAML](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L94-L129)

此前对"阶段一"的描述存在误导——阶段一并非纯"只读校验"，它包含**6 个顺序步骤**，其中 Step 1、2、3、5 会修改字节流、config 对象、widget 内部状态或全局计数器：

```
Step 1  parseConfigVariables(contents)      ← 修改输入字节流（不涉及 config 对象）
   ↓
Step 2  config := &config{}
        config.Server.Port = 8080           ← 修改 config 对象（⚠️ 在 Unmarshal 之前写入）
   ↓
Step 3  yaml.Unmarshal(contents, config)    ← 修改 config 对象（填充 YAML 字段）
                                           ← ⚠️ 内部 widgets.UnmarshalYAML 还递增全局 widgetIDCounter
   ↓
Step 4  isConfigStateValid(config)          ← 只读校验，不修改任何数据
   ↓
Step 5  遍历 pages/columns/widgets
        widget.initialize()                 ← 修改 widget 内部状态（标题、缓存策略、预渲染等）
   ↓
Step 6  return config
```

### Step 1：变量插值（字节流层面，不涉及 config 对象）

见第 2 章。输入/输出均为 `[]byte`。

### Step 2：写入 Port 默认值（在 Unmarshal 之前）

```go
config := &config{}         // 所有字段为 Go 零值：Port=0, 字符串="", 切片=nil, map=nil
config.Server.Port = 8080   // 先写入默认值
```

**关键细节**：这是一个**Unmarshal 前默认值**策略——先写 8080，再让 YAML 决定是否覆盖。如果用户 YAML 里写了 `server.port: 9090`，Unmarshal 会把 8080 覆盖成 9090；如果没写，就保留 8080。

这也是整个阶段一里**唯一在 Unmarshal 之前被硬编码**的字段。其他所有字段在 Step 3 之前都保持 Go 零值。

### Step 3：yaml.Unmarshal 递归填充结构体

涉及的自定义 UnmarshalYAML（见第 3 章）：
- `hslColorField.UnmarshalYAML` — 解析 HSL 颜色，内部转为 `hslColor` 结构体
- `durationField.UnmarshalYAML` — 解析 `30s/5m/2h/1d`，内部转为 `time.Duration`
- `customIconField.UnmarshalYAML` — 解析图标前缀（si:/mdi:/di:/sh:），内部转为完整 CDN URL + AutoInvert 标志
- `proxyOptionsField.UnmarshalYAML` — 解析代理（简写 URL 或完整对象），并**构造 `*http.Client`**（包含 Transport + Timeout）
- `queryParametersField.UnmarshalYAML` — 归一化为 `map[string][]string`
- `orderedYAMLMap.UnmarshalYAML` — 保序的键值对映射（`[]K` 保序 + `map[K]V` 存值）
- `widgets.UnmarshalYAML` — 按 type 工厂创建具体 widget 结构体，**并递增全局 `widgetIDCounter`**

**状态变化**：
1. 所有 `yaml` tag 标记的字段被填充
2. ⚠️ `widgetBase.ID`（虽为 `yaml:"-"`）在此阶段通过 `newWidget → w.setID(widgetIDCounter.Add(1))` 被赋值为**全局唯一自增 ID**，并非零值
3. ⚠️ `proxyOptionsField` 内部的 `*http.Client` 在此阶段已构造完成
4. 其余 `yaml:"-"` 字段（如 `PrimaryColumnIndex`、`user.PasswordHash`、`widgetBase.cacheType` 等）仍为 Go 零值

**全局副作用**：`widgetIDCounter`（`atomic.Uint64`）被多次 `Add(1)`，热重载时每次重新加载都会让计数器从上次的位置继续递增，不会复位。

### Step 4：isConfigStateValid 只读校验

见第 4 章。不修改任何数据。

### Step 5：Widget initialize()——阶段一最隐蔽的状态修改

[newConfigFromYAML#L112-L126](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L112-L126) 按 `pages → headWidgets`、`pages → columns → widgets` 两层嵌套遍历，对每个 widget 调用 `initialize()`。

每个 widget 的 `initialize()` 会修改以下内部状态（`yaml:"-"` 字段）：

| 修改操作 | 辅助方法 | 说明 |
|---------|---------|------|
| 默认标题 | `withTitle("Videos")` | `w.Title == ""` 时才写入，用户 YAML 写了 title 则不覆盖 |
| 默认标题链接 | `withTitleURL("https://...")` | 同上，空则写入 |
| 缓存策略 | `withCacheDuration(2*time.Hour)` / `withCacheOnTheHour()` | 设置 `cacheType` 和 `cacheDuration`；若用户写了 `cache:` 字段（`CustomCacheDuration > 0`）则优先用用户值 |
| 内容可用标记 | `withError(nil)` | 若 `ContentAvailable == false`，置为 `true` |
| 特有字段默认值 | — | 如 `rssWidget.Limit <= 0 → Limit=5`、`videosWidget.Limit <= 0 → Limit=5` 等 |
| 预渲染 HTML | — | 如 `todoWidget` 在 initialize 时就执行 `renderTemplate` 写入 `cachedHTML` |
| WIP 标记 | — | 如 `serverStatsWidget` 将 `WIP = true` |

**为什么放在阶段一而不是阶段二？**
- initialize 属于 widget 自身的"反序列化后自洽"逻辑，不依赖 application 上下文（不需要 `slugToPage`、`authSecretKey`、`providers` 等）
- 但 `widget.setProviders()` 是在**阶段二**执行的，因为 provider 需要 `app.StaticAssetPath`

### 阶段一出口时的状态快照

| 类别 | 状态 |
|------|------|
| `Server.Port` | 用户值或 8080 |
| 所有 YAML tag 字段 | 已填充 |
| `yaml:"-"` 运行时字段 | 大部分为 Go 零值（如 `PrimaryColumnIndex=0`、`user.PasswordHash=nil`、`widgetBase.cacheType=0`）；⚠️ **`widgetBase.ID` 已通过 `widgetIDCounter` 赋全局唯一值** |
| widget 内部（`yaml:"-"`） | `initialize()` 已执行，默认标题/缓存策略/预渲染已写入 |
| `proxyOptionsField` 内部 | `*http.Client` 已构造 |
| 全局计数器 `widgetIDCounter` | 已递增 N 次（N = 所有页面 widget 总数），热重载时继续累加不复位 |
| Auth 密码 | 明文仍在 `user.Password` 字段，未哈希 |
| Theme CSS / BackgroundColorAsHex | 尚未计算 |
| slugToPage / widgetByID | 尚未建立索引 |

---

## 6. 阶段二内部：newApplication 的执行顺序与默认值依赖链

**核心函数**: [newApplication](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/glance.go#L47-L231)

阶段二的默认值分为**归一化**（将 YAML 值规范为内部表示）、**继承**（从已有字段取值）、**派生**（基于其他字段计算新值）三类，且有严格的执行顺序，后一步依赖前一步的输出：

```
Step 1   Config 复制到 app.Config，取别名 config = &app.Config
   ↓
Step 2   Init Auth（密码哈希、secret 解码）
   ↓
Step 3   Init Theme Presets（内置 + 用户 Merge → presets.init()）
   ↓
Step 4   默认主题 config.Theme.init()  ← 计算出 BackgroundColorAsHex，后续 Branding 依赖
   ↓
Step 5   Init Pages
   │      ├─ 5a PrimaryColumnIndex = -1（复位）
   │      ├─ 5b Slug 空 → titleToSlug 派生
   │      ├─ 5c Slug 保留字校验
   │      ├─ 5d Width "default" → 置空（归一化）
   │      ├─ 5e DesktopNavigationWidth 空 → 继承 Width（⚠️ 依赖 5d 归一化结果）
   │      ├─ 5f 注册 head widgets（widgetByID + setProviders）
   │      └─ 5g PrimaryColumnIndex = 第一个 full 列下标（注册时顺带计算）
   ↓
Step 6   URL 与 Branding 默认值
   │      ├─ 6a BaseURL 去尾斜杠（归一化）
   │      ├─ 6b CustomCSSFile / LogoURL 解析资产路径
   │      ├─ 6c FaviconURL 空 → StaticAssetPath 派生
   │      ├─ 6d FaviconType 根据 FaviconURL 后缀推断（依赖 6c）
   │      ├─ 6e AppName 空 → "Glance"
   │      ├─ 6f AppIconURL 空 → StaticAssetPath 派生
   │      └─ 6g AppBackgroundColor 空 → Theme.BackgroundColorAsHex 派生（依赖 Step 4）
   ↓
Step 7   执行 manifest.json 模板（依赖上述所有字段的最终值）
   ↓
Step 8   return app
```

### Step 2：Auth 初始化——将明文密码擦除为哈希

```go
// 仅当 len(Auth.Users) > 0 时执行
secretBytes := base64.StdEncoding.DecodeString(config.Auth.SecretKey)  // 阶段一已保证非空
// 校验长度 == AUTH_SECRET_KEY_LENGTH (64 字节)

for username := range config.Auth.Users {
    user := config.Auth.Users[username]
    if user.PasswordHashString != "" {
        user.PasswordHash = []byte(user.PasswordHashString)
        user.PasswordHashString = ""          // 擦除中间字段
    } else {
        user.PasswordHash = bcrypt.Hash(user.Password)
        user.Password = ""                    // ⚠️ 擦除明文密码
    }
}
```

**状态变化**：`user.Password` 和 `user.PasswordHashString` 被置空，敏感信息转移到 `yaml:"-"` 的 `PasswordHash` 字段。

### Step 3-4：Theme 初始化——先 Merge 预设，再逐个 init()

```go
// Step 3: 构造内置预设 default-dark / default-light，与用户 presets Merge
config.Theme.Presets = *themePresets.Merge(&config.Theme.Presets)

// 遍历所有预设（包括被用户覆盖的内置预设）
for key, properties := range config.Theme.Presets.Items() {
    properties.Key = key
    properties.init()          // 计算 CSS, PreviewHTML, BackgroundColorAsHex
}

// Step 4: 默认主题也执行 init()
config.Theme.Key = "default"
config.Theme.init()            // ⚠️ 这里计算出的 BackgroundColorAsHex 将在 Step 6g 被引用
```

[themeProperties.init()](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/theme.go#L56-L76) 的输出：
- `CSS`: 由主题模板编译而成的样式
- `PreviewHTML`: 主题选择器卡片的预览 HTML
- `BackgroundColorAsHex`: 背景色的 HEX 字符串（若 BackgroundColor 为 nil 则兜底 `"#151519"`）

### Step 5：Pages 初始化——归一化、继承、派生的顺序陷阱

对每个 page 按以下子步骤顺序执行，**不能打乱**：

```
5a  page.PrimaryColumnIndex = -1         // 复位为"未找到"哨兵值
5b  if page.Slug == "" → page.Slug = titleToSlug(page.Title)    // 派生
5c  检查 page.Slug ∈ {"login","logout"} → 报错（保留字冲突）
5d  if page.Width == "default" → page.Width = ""                // 归一化
5e  if page.DesktopNavigationWidth == "" && != "default"
       → page.DesktopNavigationWidth = page.Width               // ⚠️ 继承：依赖 5d 的结果
5f  注册 head widgets 到 app.widgetByID; widget.setProviders(providers)
5g  遍历 columns:
       if PrimaryColumnIndex == -1 && column.Size == "full"
           → PrimaryColumnIndex = int8(c)    // 命中第一个 full 列后不再改变
       注册 column widgets 到 app.widgetByID; widget.setProviders
```

**关键依赖**：
- `5e → 5d`：DesktopNavigationWidth 的继承必须在 Width 归一化之后执行。如果 YAML 写了 `width: default`，`5d` 先把它置为空字符串，`5e` 才会继承到正确的空值而不是字面量 `"default"`
- `5g → 5a`：PrimaryColumnIndex 在循环前先设为 `-1`，确保多页面时不会残留上一页的值

### Step 6：Branding 默认值——跨模块派生链

```
6a  BaseURL = TrimRight(BaseURL, "/")                            // 归一化
6b  CustomCSSFile / LogoURL = resolveUserDefinedAssetPath(...)   // "/assets/xxx" 加 BaseURL 前缀
6c  FaviconURL = 空 ? StaticAssetPath("favicon.svg") : resolve(...)   // 派生
6d  FaviconType = HasSuffix(FaviconURL, ".svg") ? "image/svg+xml" : "image/png"   // 推断（依赖 6c）
6e  AppName = 空 ? "Glance" : AppName                             // 派生
6f  AppIconURL = 空 ? StaticAssetPath("app-icon.png") : AppIconURL  // 派生
6g  AppBackgroundColor = 空 ? Theme.BackgroundColorAsHex : AppBackgroundColor  // 派生（⚠️ 依赖 Step 4 的 init() 结果）
```

**关键依赖**：
- `6d → 6c`：FaviconType 基于 FaviconURL 的后缀判断，必须在 6c 确定最终 URL 之后
- `6g → Step 4`：AppBackgroundColor 回退到主题背景色的 HEX 值，而 `BackgroundColorAsHex` 只有在 `theme.init()` 执行完才不为空。如果把 Step 6 放在 Step 4 之前，会拿到空字符串

### 主题预设合并（orderedYAMLMap.Merge）

[orderedYAMLMap.Merge](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L583-L600)

```
内置默认预设 (default-dark, default-light)
            ↓ Merge
      用户自定义 presets
```

合并策略：
- **键顺序**: 先保留 self（内置）的 keys，再追加 other（用户）中 self 没有的 key
- **值覆盖**: `maps.Copy(merged.data, other.data)` 后执行，用户值覆盖内置值
- **结果**: 用户可覆盖默认主题，也可新增自定义主题，顺序为内置在前

---

## 7. Widget 初始化（阶段一 Step 5 与阶段二 Step 5f/5g 的分工）

Widget 的初始化跨越两个阶段，分工不同：

### 阶段一：widget.initialize()——自洽初始化

在 [newConfigFromYAML#L112-L126](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L112-L126) 中执行：

```go
for p := range config.Pages {
    for w := range config.Pages[p].HeadWidgets {
        if err := config.Pages[p].HeadWidgets[w].initialize(); err != nil {
            return nil, formatWidgetInitError(err, config.Pages[p].HeadWidgets[w])
        }
    }
    // columns widgets 同理
}
```

`formatWidgetInitError` 将错误包装为 `<widget-type> widget: <原始错误>`，**但丢失行号**（YAML Node 已在反序列化完成后丢弃）。

**职责**：设置默认标题、默认缓存策略、校验字段必填性（如 `redditWidget.Subreddit` 不能为空）、预渲染 HTML。不依赖 application 上下文。

### 阶段二：widget.setProviders() + 索引注册

在 [newApplication#L175-L193](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/glance.go#L175-L193) 中执行：

```go
for i := range page.HeadWidgets {
    widget := page.HeadWidgets[i]
    app.widgetByID[widget.GetID()] = widget      // 注册到全局索引
    widget.setProviders(providers)                // 注入 assetResolver 依赖
}
// column widgets 同理
```

**职责**：注入依赖（`providers.assetResolver` 用于解析静态资产路径）、建立 ID→widget 的全局映射（供 API 路由使用）。必须在阶段二，因为需要 `app.StaticAssetPath`。

---

## 8. 错误处理与聚合策略

### 8.1 Fail-Fast（快速失败）

整个配置加载链路采用**遇到第一个错误立即返回**的策略，不做错误聚合：

| 阶段 | 错误处理方式 |
|------|-------------|
| Include 递归解析 | 闭包中设置 `includesLastErr`，下一次回调直接 return nil |
| 变量插值 | 闭包中设置 `err`，下一次回调直接 return nil |
| YAML 反序列化 | yaml.Unmarshal 遇到第一个错误返回 |
| 状态校验 | 按顺序校验，遇到第一个不满足项返回 |
| Widget 初始化 | 逐个 initialize，第一个失败立即返回 |
| 应用初始化 | 逐项初始化，第一个失败立即返回 |

### 8.2 错误包装链

使用 `fmt.Errorf("%w", err)` 逐层包装，形成可读的错误链：

```
creating application: initializing default theme: ...
validating config file: page 1 has no columns
parsing config: reading ./extra.yml: open ./extra.yml: no such file or directory
```

### 8.3 运行时热重载的错误容忍

[serveApp](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/main.go#L93-L181) 中的 `onChange` 回调：
- **首次启动**: 配置无效 → 直接退出（关闭 exitChannel）
- **运行中变更**: 配置无效 → 仅打日志，保留旧配置继续运行
- Widget 运行时更新失败 → 仅记录在 widget.Error / widget.Notice，不影响全局

### 8.4 文件监视器的错误传播

- `watcher.Errors` channel → 通过 `onErr` 回调打日志
- Watcher 初始化失败 → 降级为单次加载，不启用热重载
- Include 文件集合变动 → 动态 `Add`/`Remove` 监视路径

---

## 9. 文件热重载

**核心函数**: [configFilesWatcher](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L305-L445)

### 工作流程

```
fsnotify.Watcher
    ↓ Write/Rename/Remove 事件
debouncedParseAndCompareBeforeCallback (500ms 防抖)
    ↓
重新 parseYAMLIncludes()
    ↓
比较 includes 集合 → 有变化则更新监视文件列表
比较内容字节 → 有变化则触发 onChange()
```

### 关键点

- **防抖**: 500ms 内多次事件合并为一次处理，避免编辑器保存时的多次触发
- **Rename 处理**: Linux 下 rename 后文件不再被监视，会删除旧路径、最多轮询 2 秒等新文件创建，然后重新解析
- **集合比较**: `maps.Equal(lastIncludes, currentIncludes)` 检测 include 集合变化
- **内容比较**: `bytes.Equal(lastContents, currentContents)` 避免无意义的重载
- **线程安全**: `mu.Mutex` 保护 `lastContents` 和 `lastIncludes` 的并发读写
- **已知限制**: Windows 下 rename 行为与 Linux 不一致（见代码注释中 fsnotify issue #255）

---

## 10. 相关文件速查

| 文件 | 职责 |
|------|------|
| [config.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go) | 主配置结构体、加载入口、include/变量解析、状态校验、有序映射、文件监视 |
| [config-fields.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config-fields.go) | HSL 颜色、时长、图标、代理、查询参数的自定义 UnmarshalYAML |
| [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/glance.go) | newApplication：默认值合并、认证/主题/页面二次初始化 |
| [main.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/main.go) | CLI 入口、serveApp 热重载调度、校验/打印子命令 |
| [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go) | widgets 列表反序列化、widget 接口定义、基类 |
| [auth.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/auth.go) | AUTH_SECRET_KEY_LENGTH 常量定义、会话 Token 生成与校验 |
| [theme.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/theme.go) | themeProperties.init()、主题 CSS/HEX 计算、预设 SameAs 比较 |

---

## 11. 边界条件与两阶段校验/默认值深度分析

代码 TODO ([config.go#L447-L450](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L447-L450)) 已明确指出当前校验分散在两处。结合第 5、6 章的精确执行顺序分析，**两个阶段都会修改状态**：
- **阶段一** = [newConfigFromYAML](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L94-L129)：共 6 步，**4 步修改状态**（Step 1 字节流、Step 2 Port 默认值、Step 3 Unmarshal + 全局计数器、Step 5 widget.initialize），其中 Step 3 附带 **1 个全局副作用**（widgetIDCounter 递增）
- **阶段二** = [newApplication](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/glance.go#L47-L231) + [serveApp onChange](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/main.go#L101-L146)：共 8 步，全部修改状态或索引（归一化/继承/派生默认值、密码哈希、主题 CSS 计算、索引注册、热重载错误分流）

### 11.1 认证 secret-key 的两阶段校验

| 阶段 | 校验位置 | 校验内容 | 错误信息 | 原因说明 |
|------|---------|---------|---------|---------|
| 阶段一 | [isConfigStateValid#L456-L458](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L456-L458) | 当 `len(Auth.Users) > 0` 时，`SecretKey` **非空即可** | `"secret-key must be set when users are configured"` | 此阶段只做"存在性"预检，尚未解码，无法判断字节长度是否合法 |
| 阶段二 | [newApplication#L61-L69](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/glance.go#L61-L69) | ① Base64 解码成功；② 解码后字节数必须精确等于 `AUTH_SECRET_KEY_LENGTH`(64 字节) | `"decoding secret-key: ..."` 或 `"secret-key must be exactly 64 bytes"` | 常量定义：`AUTH_SECRET_KEY_LENGTH = AUTH_TOKEN_SECRET_LENGTH(32) + AUTH_USERNAME_HASH_LENGTH(32)`，来自 [auth.go#L27-L29](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/auth.go#L27-L29)。后续 `generateSessionToken`/`computeUsernameHash`/`verifySessionToken` 均依赖该长度做切片（如 `secret[AUTH_TOKEN_SECRET_LENGTH:]`），长度错误会导致越界 |

**易混淆点**：为什么不在阶段一就校验长度？
- 阶段一是纯 YAML 层面的逻辑校验，不解码、不依赖加密模块
- Base64 解码属于"数据变换"范畴，归入阶段二的初始化流程
- 两阶段分离使得 `glance --config-validate` CLI 命令（仅调用阶段一）可以不依赖 auth 子系统的全部逻辑

---

### 11.2 页面标识（Slug）与 full 列约束的两阶段处理

#### 11.2.1 Full 列约束

| 阶段 | 校验位置 | 内容 | 原因说明 |
|------|---------|------|---------|
| 阶段一 | [isConfigStateValid#L503-L533](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L503-L533) | ① slim 页面列数 ≤ 2，其他 ≤ 3；② 每列 size 只能是 small/full；③ **full 列数量必须为 1 或 2** | 这些是纯静态约束，与运行时逻辑无关，应在 YAML 层面拦截 |
| 阶段二 | [newApplication#L155 + L181-L186](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/glance.go#L155-L186) | ① `PrimaryColumnIndex` 初始化为 -1；② 遍历列时遇到 **第一个 size=full** 的列，将其下标赋值给 `PrimaryColumnIndex` | `PrimaryColumnIndex` 是运行时字段（`yaml:"-"`），不参与 YAML 序列化。之所以选"第一个 full 列"，是因为阶段一已经保证至少有 1 个 full 列，无需再判空 |

**易混淆点**：阶段一已保证 full ∈ {1,2}，阶段二为什么只取第一个？
- 前端渲染时"主列"只需一个锚点（通常放主要内容），第二个 full 列作为辅助
- 若需要多主列语义，应由布局 CSS 处理，此处 `PrimaryColumnIndex` 仅用于内部逻辑定位

#### 11.2.2 页面 Slug 处理

| 阶段 | 处理位置 | 内容 | 原因说明 |
|------|---------|------|---------|
| 阶段一 | 无 | 阶段一对 Slug **完全不校验** | Slug 是可选项（由 title 派生），且需要 title→slug 转换函数，归入初始化阶段更合适 |
| 阶段二 | [newApplication#L157-L165](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/glance.go#L157-L165) | ① Slug 为空时用 `titleToSlug(page.Title)` 自动生成；② 检查 Slug 是否命中保留字 `["login", "logout"]`（[glance.go#L28](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/glance.go#L28)）；③ 将 `""` 映射到第一页、各 slug 映射到对应 page，写入 `slugToPage` | 保留字冲突只能在"派生完成后"检测，因为用户没写 slug 时需要先从 title 算出来才知道是否冲突。`slugToPage[""]` 指向第一页，实现根路径 `/` 默认访问第一页的路由语义 |

**易混淆点**：阶段一校验了 page.Title 非空，为什么不同时校验 Slug？
- Slug 具有"可派生"属性——没有 Slug 并不一定是错误，可以从 Title 自动生成
- 保留字列表与路由系统耦合，放在阶段二（和路由注册一起）职责更内聚

---

### 11.3 热重载：首次启动 vs 运行中报错的分流逻辑

[serveApp](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/main.go#L93-L181) 用一个布尔标志 `hadValidConfigOnStartup` 控制错误分流。

#### 状态流转图

```
serveApp() 启动
   │
   ├─ parseYAMLIncludes() 失败 ──→ return error（进程直接退出，无日志分流）
   │
   ├─ configFilesWatcher() 初始化失败 ──→ 降级路径：
   │     ├─ newConfigFromYAML() 失败 → return error
   │     ├─ newApplication() 失败    → return error
   │     └─ startServer()            → 启动成功但无热重载
   │
   └─ configFilesWatcher() 初始化成功：
         └─ 立即触发 onChange(初始内容)
              │
              ├─ hadValidConfigOnStartup == false（首次）
              │    ├─ newConfigFromYAML/newApplication 失败 → close(exitChannel) → 进程退出
              │    └─ 成功 → hadValidConfigOnStartup = true，启动 server
              │
              └─ hadValidConfigOnStartup == true（运行中变更）
                   ├─ newConfigFromYAML/newApplication 失败 → log.Printf 打日志，return，保留旧 server
                   └─ 成功 → stopServer() → 启动新 server
```

#### 分流原因分析

| 场景 | 处理策略 | 原因 |
|------|---------|------|
| **首次启动加载失败** | 立即退出（关闭 exitChannel → `<-exitChannel` 返回 → serveApp return） | 启动时连合法配置都没有，服务无法提供任何功能，快速失败便于用户发现 |
| **运行中变更失败** | 仅打日志，保留旧配置 | 热重载的核心价值是"不中断服务"，旧配置仍然有效时应继续运行，让用户有机会修正错误 |
| **文件监视器初始化失败** | 降级为单次加载成功即启动 | fsnotify 在某些环境（容器、特定 FS）可能不可用，但配置本身合法，不应因此阻止服务启动 |
| **Include 解析首次失败** | 在 serveApp 开头直接 return error（不进入 onChange） | `parseYAMLIncludes` 在 watcher 启动前同步执行，还没有 `hadValidConfigOnStartup` 标志，统一走错误返回 |

**易混淆点**：为什么 watcher 初始化失败的降级路径里 `startServer, _ := app.server()` 忽略了 stopServer？
- 降级路径是同步启动（无 go func），服务阻塞在 `ListenAndServe` 上
- 降级意味着没有热重载，服务不会被中途停止，所以不需要 stop 函数
- 正常路径（有 watcher）是异步 goroutine 启动，需要 stopServer 句柄用于配置变更时优雅重启

---

### 11.4 Widget 反序列化与初始化报错定位精确分析

Widget 的整个加载流程共 **6 个报错分支**（反序列化 5 个 + initialize 1 个），每个分支的定位信息来源、手动添加方式各不相同。

先回顾 [widgets.UnmarshalYAML](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L95-L124) 的完整代码骨架：

```go
func (w *widgets) UnmarshalYAML(node *yaml.Node) error {
    var nodes []yaml.Node
    // ─── 分支 A ───
    if err := node.Decode(&nodes); err != nil { return err }

    for _, node := range nodes {
        meta := struct { Type string `yaml:"type"` }{}
        // ─── 分支 B ───
        if err := node.Decode(&meta); err != nil { return err }

        // ─── 分支 C ───
        widget, err := newWidget(meta.Type)
        if err != nil { return fmt.Errorf("line %d: %w", node.Line, err) }

        // ─── 分支 D ───
        if err = node.Decode(widget); err != nil { return err }

        *w = append(*w, widget)
    }
    return nil
}
```

以及 `newWidget` 内部的两个错误返回 ([widget.go#L20-L91](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L20-L91))：

```go
func newWidget(widgetType string) (widget, error) {
    // ─── 分支 C-1 ───
    if widgetType == "" {
        return nil, errors.New("widget 'type' property is empty or not specified")
    }
    switch widgetType {
    case "calendar", "clock", "weather", ...: w = &xxxWidget{}
    // ─── 分支 C-2 ───
    default: return nil, fmt.Errorf("unknown widget type: %s", widgetType)
    }
    w.setID(widgetIDCounter.Add(1))  // 即使后面出错，计数器已递增
    return w, nil
}
```

#### 6 个报错分支逐一分

| 分支 | 代码位置 | 触发场景 | 定位信息来源 | 是否手动添加 | 错误信息示例 | 补充定位方案 |
|------|---------|---------|------------|-------------|-------------|------------|
| **A** | [widget.go#L98-L100](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L98-L100) | `widgets` 根节点不是数组（如写成 map/string） | yaml.v3 内部 token 位置 | ❌ 直接 return err | `yaml: line 42: cannot unmarshal !!map into []yaml.Node` | 可包装 `fmt.Errorf("line %d: widgets list: %w", node.Line, err)` 显式补充根节点行号 |
| **B** | [widget.go#L107-L109](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L107-L109) | 单个 widget 节点无法解析出 `type` 字段（如 type 写成 int） | yaml.v3 内部 token 位置 | ❌ 直接 return err | `yaml: line 45: cannot unmarshal !!int into string` | 可包装 `fmt.Errorf("line %d: parsing widget type: %w", node.Line, err)` |
| **C-1** | [widget.go#L21-L23](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L21-L23) | `type` 字段为空字符串或未配置 | 外层手动 `fmt.Errorf("line %d: %w", node.Line, err)` | ✅ 手动加 `node.Line` | `line 47: widget 'type' property is empty or not specified` | 当前已正确处理；`node.Line` 是 `- ` 列表项起始行 |
| **C-2** | [widget.go#L84-L86](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L84-L86) | `type` 值不在 switch 白名单中 | 同上 | ✅ 手动加 `node.Line` | `line 47: unknown widget type: foobar` | 同上 |
| **D** | [widget.go#L116-L118](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L116-L118) | 具体 widget 字段类型不匹配（如 `limit: "abc"`） | yaml.v3 内部 token 位置 | ❌ 直接 return err | `yaml: line 50: cannot unmarshal !!str `abc` into int` | 可包装 `fmt.Errorf("line %d: widget %q fields: %w", node.Line, meta.Type, err)` 补充 type 上下文 |
| **E** | [config.go#L112-L126](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L112-L126) | `widget.initialize()` 失败（必填字段空、格式不合法等） | `formatWidgetInitError` 包装 widget type，**丢失行号** | ❌ 无行号，仅加 type | `reddit widget: subreddit must be specified` | 需在反序列化时将 `node.Line` 存入 widget（如加 `YAMLLine` 字段），或在遍历中维护索引 |

#### 关键定位信息的来源辨析

**yaml.v3 自带行号 vs 手动拼接行号**

- yaml.v3 的 `Node.Decode()` 返回的错误通常包含 `yaml.TypeError`，其 `Errors []string` 中每项形如 `line N: ...`，由库内部在 token 解析时自动记录
- Go 代码逻辑产生的错误（如 `newWidget` 的 switch default、`initialize()` 的字段校验）与 YAML token 无关，必须手动拼接

**`node.Line` 的准确含义**

在分支 C 中，`node.Line` 指的是：
> 当前列表项节点的**起始行**，即 `- type: weather` 中 `- ` 字符所在的行号。

若 widget 配置跨多行（如 YAML 中 `- type: weather` 后换行再写其他字段），`node.Line` 是该 widget **第一个 token** 的行号，不一定是出错字段所在行。分支 D 中 yaml.v3 自带的行号则是**出错字段**的精确行号。

#### `widgetIDCounter` 的全局副作用陷阱

[newWidget#L88](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L88) 在 **switch 成功后立即执行** `widgetIDCounter.Add(1)`：
- 即使后续分支 D `node.Decode(widget)` 失败、整个反序列化报错退出，计数器**已经递增**
- 热重载时每次重新解析配置，计数器从上一次的值继续累加，**不会复位**
- 这意味着 widget 的 `ID` 字段是"进程生命周期内单调递增"的，不是"每次配置加载从 1 开始"

#### initialize() 阶段无法定位行号的根因

[formatWidgetInitError](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L236-L238) 仅能拿到 `widget.GetType()`：

```go
func formatWidgetInitError(err error, widget widget) error {
    return fmt.Errorf("%s widget: %w", widget.GetType(), err)
}
```

丢失行号的原因是信息传递链条断裂：
1. `widgets.UnmarshalYAML` 中有 `node.Line`
2. 但 `widget` 接口**没有 `SetYAMLLine()` / `GetYAMLLine()` 方法**
3. `newConfigFromYAML` 的遍历循环中也没有维护"当前 widget 行号"的外部索引

若要修复，方案有二：
- 在 `widgetBase` 中加 `YAMLLine int` 字段，在 UnmarshalYAML 中赋值
- 或在 `newWidget` 返回成功后、`Decode` 之前，把 `node.Line` 写入 widget

---

### 11.5 两阶段执行步骤与边界条件汇总表

#### 阶段一：newConfigFromYAML（6 步，4 步修改状态 + 1 步全局副作用）

| 步骤 | 操作 | 是否改状态 | 关键细节 / 陷阱 |
|------|------|-----------|----------------|
| 1 | `parseConfigVariables(contents)` | ✅（字节流） | `${env}` / `${secret:}` / `${readFileFromEnv:}` 插值；注释中的变量也会被替换 |
| 2 | `config.Server.Port = 8080` | ✅（config 对象） | **Unmarshal 前写入**——YAML 有值则覆盖，无值则保留 8080；此阶段唯一硬编码默认值 |
| 3 | `yaml.Unmarshal(contents, config)` | ✅（config 对象 + 全局） | 递归调用各自定义 UnmarshalYAML；⚠️ `widgetBase.ID`（`yaml:"-"`）通过 `newWidget → widgetIDCounter.Add(1)` 被赋值为全局自增 ID，并非零值；⚠️ `proxyOptionsField` 内部 `*http.Client` 已构造 |
| 4 | `isConfigStateValid(config)` | ❌（只读） | 列数、full 列数量、secret-key 非空、用户名/密码长度等静态约束 |
| 5 | `widget.initialize()` 遍历 | ✅（widget 内部） | 默认标题/缓存策略/字段必填校验/预渲染 HTML；**不依赖 application** |
| 6 | `return config` | — | 出口状态：密码仍明文、Theme CSS 未计算、PrimaryColumnIndex 为 0（Go 零值）；widget ID 已全局唯一 |

#### 阶段二：newApplication + serveApp onChange（8 步，全部改状态或索引）

| 步骤 | 操作 | 默认值类型 | 关键依赖链 |
|------|------|-----------|-----------|
| 1 | Config 复制到 app.Config | — | 取别名 `config = &app.Config`，后续操作均修改 app 内部字段 |
| 2 | Auth 初始化：Base64 解码 secret-key、bcrypt 哈希密码 | 派生（变换） | 依赖阶段一保证了 SecretKey 非空；明文密码被擦除 |
| 3 | 主题预设 Merge + presets.init() | 派生（计算） | 内置 dark/light 与用户 presets 合并；生成 CSS/PreviewHTML/BackgroundColorAsHex |
| 4 | 默认主题 `config.Theme.init()` | 派生（计算） | ⚠️ **输出 `BackgroundColorAsHex`**，后续 Step 6g 依赖此值 |
| 5 | Pages 初始化：<br>5a PrimaryColumnIndex=-1<br>5b Slug=titleToSlug(Title)<br>5c Slug 保留字校验<br>5d Width "default"→""<br>5e DesktopNavWidth 继承 Width<br>5f/5g Widget 注册 + setProviders | 归一化/继承/派生 | ⚠️ **5e 必须在 5d 之后**（Width 先归一化再被继承）；⚠️ **5g 必须在 5a 之后**（复位后再定位第一个 full 列） |
| 6 | Branding / URL 默认值：<br>6a BaseURL TrimRight `/`<br>6b Logo/CSS 资产路径解析<br>6c FaviconURL 默认值<br>6d FaviconType 后缀推断<br>6e AppName="Glance"<br>6f AppIconURL 默认值<br>6g AppBackgroundColor=Theme.BackgroundColorAsHex | 归一化/派生 | ⚠️ **6d 依赖 6c**（URL 确定后才能判断后缀）；⚠️ **6g 依赖 Step 4**（theme.init() 必须先算完 BackgroundColorAsHex） |
| 7 | `manifest.json` 模板执行 | 派生 | 依赖上述所有字段的最终值 |
| 8 | 热重载错误分流 | 运行时控制 | `hadValidConfigOnStartup` 标志：首次失败 → 退出进程；运行中失败 → 打日志保留旧 server |
