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
[widgets.UnmarshalYAML](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L95-L124) 的关键流程：
1. 先解出每个 widget 的 `type` 字段
2. 根据 type 调用 `newWidget()` 通过 switch 工厂创建对应结构体实例
3. 将完整节点 Decode 到具体 widget 上
4. 错误信息携带 `node.Line` 行号，便于定位

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

## 5. 默认值合并

默认值的设置分散在**两个阶段**：

### 阶段 A：YAML 反序列化前（硬编码默认值）

[newConfigFromYAML](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L94-L129) 中：

```go
config := &config{}
config.Server.Port = 8080   // 唯一在此阶段设置的默认值
```

### 阶段 B：应用初始化时（派生/计算默认值）

[newApplication](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/glance.go#L47-L231) 中集中处理：

| 字段 | 默认值逻辑 |
|------|-----------|
| page.Slug | 为空则 `titleToSlug(page.Title)` |
| page.Width | `"default"` → 置空 |
| page.DesktopNavigationWidth | 为空则继承 page.Width |
| page.PrimaryColumnIndex | 第一个 size=full 的列下标 |
| Branding.AppName | 空 → `"Glance"` |
| Branding.FaviconURL | 空 → `/static/<hash>/favicon.svg` |
| Branding.FaviconType | 根据后缀推断 `image/svg+xml` / `image/png` |
| Branding.AppIconURL | 空 → `/static/<hash>/app-icon.png` |
| Branding.AppBackgroundColor | 空 → 主题背景色的 HEX 值 |
| Server.BaseURL | 自动去除末尾 `/` |

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

## 6. Widget 初始化

YAML 反序列化完成后，遍历所有页面的 head-widgets 和 columns widgets，逐个调用 `initialize()`。

[newConfigFromYAML](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L112-L126) 中：

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

`formatWidgetInitError` 将错误包装为 `<widget-type> widget: <原始错误>`，便于定位。

---

## 7. 错误处理与聚合策略

### 7.1 Fail-Fast（快速失败）

整个配置加载链路采用**遇到第一个错误立即返回**的策略，不做错误聚合：

| 阶段 | 错误处理方式 |
|------|-------------|
| Include 递归解析 | 闭包中设置 `includesLastErr`，下一次回调直接 return nil |
| 变量插值 | 闭包中设置 `err`，下一次回调直接 return nil |
| YAML 反序列化 | yaml.Unmarshal 遇到第一个错误返回 |
| 状态校验 | 按顺序校验，遇到第一个不满足项返回 |
| Widget 初始化 | 逐个 initialize，第一个失败立即返回 |
| 应用初始化 | 逐项初始化，第一个失败立即返回 |

### 7.2 错误包装链

使用 `fmt.Errorf("%w", err)` 逐层包装，形成可读的错误链：

```
creating application: initializing default theme: ...
validating config file: page 1 has no columns
parsing config: reading ./extra.yml: open ./extra.yml: no such file or directory
```

### 7.3 运行时热重载的错误容忍

[serveApp](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/main.go#L93-L181) 中的 `onChange` 回调：
- **首次启动**: 配置无效 → 直接退出（关闭 exitChannel）
- **运行中变更**: 配置无效 → 仅打日志，保留旧配置继续运行
- Widget 运行时更新失败 → 仅记录在 widget.Error / widget.Notice，不影响全局

### 7.4 文件监视器的错误传播

- `watcher.Errors` channel → 通过 `onErr` 回调打日志
- Watcher 初始化失败 → 降级为单次加载，不启用热重载
- Include 文件集合变动 → 动态 `Add`/`Remove` 监视路径

---

## 8. 文件热重载

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

## 9. 相关文件速查

| 文件 | 职责 |
|------|------|
| [config.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go) | 主配置结构体、加载入口、include/变量解析、状态校验、有序映射、文件监视 |
| [config-fields.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config-fields.go) | HSL 颜色、时长、图标、代理、查询参数的自定义 UnmarshalYAML |
| [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/glance.go) | newApplication：默认值合并、认证/主题/页面二次初始化 |
| [main.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/main.go) | CLI 入口、serveApp 热重载调度、校验/打印子命令 |
| [widget.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go) | widgets 列表反序列化、widget 接口定义、基类 |
| [auth.go](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/auth.go) | AUTH_SECRET_KEY_LENGTH 常量定义、会话 Token 生成与校验 |

---

## 10. 边界条件与两阶段校验/默认值深度分析

代码 TODO ([config.go#L447-L450](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L447-L450)) 已明确指出当前校验分散在两处，以下将易混淆的边界条件按**阶段一（配置加载与解析，只读校验）**与**阶段二（应用初始化与运行时，可修改数据）**进行拆分梳理。

### 10.1 认证 secret-key 的两阶段校验

| 阶段 | 校验位置 | 校验内容 | 错误信息 | 原因说明 |
|------|---------|---------|---------|---------|
| 阶段一 | [isConfigStateValid#L456-L458](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L456-L458) | 当 `len(Auth.Users) > 0` 时，`SecretKey` **非空即可** | `"secret-key must be set when users are configured"` | 此阶段只做"存在性"预检，尚未解码，无法判断字节长度是否合法 |
| 阶段二 | [newApplication#L61-L69](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/glance.go#L61-L69) | ① Base64 解码成功；② 解码后字节数必须精确等于 `AUTH_SECRET_KEY_LENGTH`(64 字节) | `"decoding secret-key: ..."` 或 `"secret-key must be exactly 64 bytes"` | 常量定义：`AUTH_SECRET_KEY_LENGTH = AUTH_TOKEN_SECRET_LENGTH(32) + AUTH_USERNAME_HASH_LENGTH(32)`，来自 [auth.go#L27-L29](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/auth.go#L27-L29)。后续 `generateSessionToken`/`computeUsernameHash`/`verifySessionToken` 均依赖该长度做切片（如 `secret[AUTH_TOKEN_SECRET_LENGTH:]`），长度错误会导致越界 |

**易混淆点**：为什么不在阶段一就校验长度？
- 阶段一是纯 YAML 层面的逻辑校验，不解码、不依赖加密模块
- Base64 解码属于"数据变换"范畴，归入阶段二的初始化流程
- 两阶段分离使得 `glance --config-validate` CLI 命令（仅调用阶段一）可以不依赖 auth 子系统的全部逻辑

---

### 10.2 页面标识（Slug）与 full 列约束的两阶段处理

#### 10.2.1 Full 列约束

| 阶段 | 校验位置 | 内容 | 原因说明 |
|------|---------|------|---------|
| 阶段一 | [isConfigStateValid#L503-L533](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L503-L533) | ① slim 页面列数 ≤ 2，其他 ≤ 3；② 每列 size 只能是 small/full；③ **full 列数量必须为 1 或 2** | 这些是纯静态约束，与运行时逻辑无关，应在 YAML 层面拦截 |
| 阶段二 | [newApplication#L155 + L181-L186](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/glance.go#L155-L186) | ① `PrimaryColumnIndex` 初始化为 -1；② 遍历列时遇到 **第一个 size=full** 的列，将其下标赋值给 `PrimaryColumnIndex` | `PrimaryColumnIndex` 是运行时字段（`yaml:"-"`），不参与 YAML 序列化。之所以选"第一个 full 列"，是因为阶段一已经保证至少有 1 个 full 列，无需再判空 |

**易混淆点**：阶段一已保证 full ∈ {1,2}，阶段二为什么只取第一个？
- 前端渲染时"主列"只需一个锚点（通常放主要内容），第二个 full 列作为辅助
- 若需要多主列语义，应由布局 CSS 处理，此处 `PrimaryColumnIndex` 仅用于内部逻辑定位

#### 10.2.2 页面 Slug 处理

| 阶段 | 处理位置 | 内容 | 原因说明 |
|------|---------|------|---------|
| 阶段一 | 无 | 阶段一对 Slug **完全不校验** | Slug 是可选项（由 title 派生），且需要 title→slug 转换函数，归入初始化阶段更合适 |
| 阶段二 | [newApplication#L157-L165](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/glance.go#L157-L165) | ① Slug 为空时用 `titleToSlug(page.Title)` 自动生成；② 检查 Slug 是否命中保留字 `["login", "logout"]`（[glance.go#L28](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/glance.go#L28)）；③ 将 `""` 映射到第一页、各 slug 映射到对应 page，写入 `slugToPage` | 保留字冲突只能在"派生完成后"检测，因为用户没写 slug 时需要先从 title 算出来才知道是否冲突。`slugToPage[""]` 指向第一页，实现根路径 `/` 默认访问第一页的路由语义 |

**易混淆点**：阶段一校验了 page.Title 非空，为什么不同时校验 Slug？
- Slug 具有"可派生"属性——没有 Slug 并不一定是错误，可以从 Title 自动生成
- 保留字列表与路由系统耦合，放在阶段二（和路由注册一起）职责更内聚

---

### 10.3 热重载：首次启动 vs 运行中报错的分流逻辑

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

### 10.4 Widget 反序列化报错定位精确分析

[widgets.UnmarshalYAML](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L95-L124) 有 4 个可能的错误返回点，行号信息的完整性各不相同：

| 序号 | 代码位置 | 错误场景 | 是否携带行号 | 错误示例 | 原因说明 |
|------|---------|---------|------------|---------|---------|
| ① | [L98-L100](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L98-L100) | `node.Decode(&nodes)` 失败（widgets 根节点不是数组） | 否（yaml 库自带位置） | `yaml: unmarshal errors: line 12: cannot unmarshal !!map into []yaml.Node` | 外层 YAML 解析器已记录 token 位置，通常自带行号 |
| ② | [L107-L109](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L107-L109) | 单个 widget 节点 Decode meta 失败（如 type 字段类型不对） | 否（yaml 库自带位置） | `yaml: unmarshal errors: line 15: cannot unmarshal !!int into string` | 同上，yaml.v3 内部错误已包含行号 |
| ③ | [L111-L114](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L111-L114) | `newWidget(meta.Type)` 失败（type 为空或未知） | **是，手动拼接** `fmt.Errorf("line %d: %w", node.Line, err)` | `line 17: unknown widget type: foobar` | `newWidget` 返回的是纯语义错误（无 YAML 位置），必须手动追加 `node.Line`。注意这个 `node.Line` 是**列表项节点的起始行**（即 `- type: xxx` 的 `-` 所在行），不一定是 `type:` 字段所在行 |
| ④ | [L116-L118](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/widget.go#L116-L118) | `node.Decode(widget)` 失败（具体 widget 字段类型错误） | 否（yaml 库自带位置） | `yaml: unmarshal errors: line 20: cannot unmarshal !!str into int` | 具体 widget 结构体解码由 yaml.v3 负责，错误会精确到字段行号 |

#### 阶段一后续：Widget initialize() 错误包装

反序列化成功后，在 [newConfigFromYAML#L112-L126](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L112-L126) 逐个调用 `widget.initialize()`：

```go
if err := config.Pages[p].HeadWidgets[w].initialize(); err != nil {
    return nil, formatWidgetInitError(err, config.Pages[p].HeadWidgets[w])
}
```

[formatWidgetInitError](file:///d:/fz/0601/solo-dogfeeding/code/133-glance/internal/glance/config.go#L236-L238) 将错误包装为 `<widget-type> widget: <原始错误>`，**但丢失了行号**——因为此时 widget 结构体已完成反序列化，不再持有原始 `yaml.Node` 的行号信息。

**易混淆点**：为什么③必须手动加行号而①②④不用？
- ①②④的错误来自 `yaml.Node.Decode()` 内部，yaml.v3 在解析 token 时已记录源位置，错误信息自动包含 `line N`
- ③的错误来自 Go 代码的 switch 分支（`newWidget` 工厂），与 YAML token 无关，必须通过闭包捕获外层 `node.Line` 手动拼接
- 这也解释了为什么 `initialize()` 阶段无法再给出行号——widget 接口里没有 `Line()` 方法，YAML 节点信息在反序列化完成后已丢弃

---

### 10.5 两阶段边界条件汇总表

| 关注点 | 阶段一（newConfigFromYAML + isConfigStateValid） | 阶段二（newApplication + serveApp onChange） |
|--------|------------------------------------------------|----------------------------------------------|
| **Auth secret-key** | ✅ 非空校验（有 users 时必须存在） | ✅ Base64 解码 + 精确 64 字节长度校验 + 密码哈希 |
| **Full 列约束** | ✅ 列数上限 + size 枚举 + full∈{1,2} | ✅ PrimaryColumnIndex = 第一个 full 列下标（-1 兜底） |
| **Page Slug** | ❌ 不处理 | ✅ 空则 titleToSlug 派生 + 保留字(login/logout)检查 + slugToPage 建索引 |
| **页面 Width** | ✅ 枚举校验（wide/slim/default/空） | ✅ "default" → 置空归一化 |
| **DesktopNavigationWidth** | ✅ 枚举校验（非空时） | ✅ 空则继承 page.Width |
| **热重载错误分流** | ❌ 不涉及运行时 | ✅ hadValidConfigOnStartup 标志：首次失败退出、运行时失败保留旧配置 |
| **Widget 反序列化** | ✅ 4 处错误点，仅 newWidget 失败手动加行号 | ✅ initialize() 错误包装为 type 前缀，但丢失行号 |
| **默认值 Port** | ✅ Server.Port = 8080（Unmarshal 前硬编码） | — |
| **默认值 Branding** | — | ✅ AppName/Favicon/AppIcon/BackgroundColor 派生默认值 |
| **主题预设** | — | ✅ 内置 default-dark/light 与用户 presets 合并，用户值覆盖内置值 |
