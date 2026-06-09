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
