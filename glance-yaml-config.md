# Glance YAML 配置加载与校验流程分析

## 一、整体流程概览

```
文件系统
    ↓ (cli.go / main.go 入口)
parseYAMLIncludes(mainFilePath)
    └── recursiveParseYAMLIncludes()  —— 递归处理 $include: 指令
    ↓
合并后的 YAML 字节流
    ↓
parseConfigVariables()  —— 变量插值 ${ENV_VAR}, ${secret:...}, ${readFileFromEnv:...}
    ↓
变量替换后的 YAML 字节流
    ↓
newConfigFromYAML()
    ├── yaml.Unmarshal()  —— 解析到 config 结构体
    ├── isConfigStateValid()  —— 逻辑校验（非空、数量约束等）
    └── 遍历所有 widget，调用 initialize()  —— widget 级校验与初始化
    ↓
*config 对象
    ↓
newApplication()  —— 二次校验 + 默认值合并 + 运行时字段计算
    ↓
可用的 application 对象
```

---

## 二、配置文件包含机制（Includes）

### 2.1 语法

支持两种写法，在 YAML 中通过 `!include:` 或 `$include:` 引入外部文件：

```yaml
# 作为列表项
pages:
  - !include: pages/home.yml
  - $include: pages/markets.yml

# 或缩进匹配（保留原缩进层级）
columns:
  - size: full
    widgets:
      !include: widgets/clock.yml
```

### 2.2 核心实现

正则表达式定义在 [config.go:240](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/config.go#L240)：

```go
var configIncludePattern = regexp.MustCompile(`(?m)^([ \t]*)(?:-[ \t]*)?(?:!|\$)include:[ \t]*(.+)$`)
```

匹配分组：
- Group 1: 缩进空白字符
- Group 2: 包含的文件路径

### 2.3 递归解析流程

函数 [recursiveParseYAMLIncludes](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/config.go#L246-L303)：

```
recursiveParseYAMLIncludes(mainFilePath, includes, depth):
    1. 检查 depth > 20 → 返回递归深度超限错误
    2. 读取主文件内容
    3. 获取主文件绝对路径和目录（用于解析相对路径 include）
    4. 用 configIncludePattern 遍历所有匹配：
       a. 提取缩进 indent 和文件路径 includeFilePath
       b. 相对路径 → 基于主文件目录拼接为绝对路径
       c. 将路径加入 includes map（用于文件监听）
       d. 递归调用自身解析被包含文件，depth+1
       e. 用 indent 给被包含文件的每一行加前缀
       f. 替换原匹配位置
    5. 返回合并后的内容 + 所有已包含文件路径集合
```

关键设计：
- **缩进保留**：`prefixStringLines(indent, content)` 确保被包含内容的缩进层级正确
- **递归深度限制**：`CONFIG_INCLUDE_RECURSION_DEPTH_LIMIT = 20` 防止循环引用
- **路径追踪**：`includes map[string]struct{}` 记录所有被包含的文件，供后续文件监听使用

### 2.4 文件热重载

[configFilesWatcher](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/config.go#L305-L445) 实现配置变更自动重载：

- 使用 `fsnotify` 监听主文件和所有 include 文件
- 500ms 防抖（debounce）避免频繁触发
- 变更后重新执行 `parseYAMLIncludes`，对比内容和 include 集合
- 动态增删监听的文件（include 列表可能改变）
- 处理 Rename/Remove 事件的平台差异（Linux 重命名后文件不再被 watch）

---

## 三、变量插值机制

### 3.1 支持的变量类型

| 语法 | 类型 | 说明 |
|------|------|------|
| `${API_KEY}` | env | 从环境变量读取，变量名须匹配 `^[A-Z0-9_]+$` |
| `${secret:api_key}` | secret | 从 `/run/secrets/api_key` 文件读取（Docker Secrets） |
| `${readFileFromEnv:PATH_VAR}` | readFileFromEnv | 从环境变量 `PATH_VAR` 指定的绝对路径文件读取 |
| `\${API_KEY}` | 转义 | 输出字面量 `${API_KEY}`（去掉反斜杠） |

### 3.2 正则匹配

定义在 [config.go:132](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/config.go#L132)：

```go
var configVariablePattern = regexp.MustCompile(`(^|.)\$\{(?:([a-zA-Z]+):)?([a-zA-Z0-9_-]+)\}`)
```

匹配分组：
- Group 1: 前缀字符（用于检测 `\` 转义）
- Group 2: 变量类型（可选，如 `secret:`, `readFileFromEnv:`）
- Group 3: 变量名称

### 3.3 解析流程

[parseConfigVariables](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/config.go#L142-L187)：

```
对每个正则匹配：
    1. 如果前缀是 `\` → 去掉反斜杠，返回原变量字符串（转义）
    2. 根据类型前缀确定 variableType：
       - 无前缀 → configVarTypeEnv
       - 有前缀 → 使用该前缀作为类型
    3. 调用 parseConfigVariableOfType(type, name) 获取值
    4. 返回 prefix + 解析后的值替换原匹配
```

[parseConfigVariableOfType](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/config.go#L190-L234) 按类型处理：

- **env**: 验证变量名格式 → `os.LookupEnv()` 读取
- **secret**: 拼接 `/run/secrets/<name>` → `os.ReadFile()` → `TrimSpace`
- **readFileFromEnv**: 读环境变量获取路径 → 校验绝对路径 → 读文件 → `TrimSpace`
- **未知类型**: 返回 `returnOriginal=true`，保留原字符串不变

### 3.4 设计要点

- **变量插值在 YAML 解析之前执行**：变量可以出现在 YAML 的任何位置，包括结构本身（如键名、缩进等）
- **错误终止**：任何变量解析失败立即终止整个流程，返回错误
- **白名单模式**：环境变量名必须全大写+数字+下划线，避免误匹配普通文本

---

## 四、YAML 解析与自定义字段

### 4.1 主配置结构体

[config.go:30-69](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/config.go#L30-L69) 定义了顶层配置：

```go
type config struct {
    Server    struct{...}   `yaml:"server"`
    Auth      struct{...}   `yaml:"auth"`
    Document  struct{...}   `yaml:"document"`
    Theme     struct{
        themeProperties `yaml:",inline"`   // 内嵌，字段平铺
        Presets orderedYAMLMap[string, *themeProperties] `yaml:"presets"`
    } `yaml:"theme"`
    Branding  struct{...}   `yaml:"branding"`
    Pages     []page        `yaml:"pages"`
}
```

### 4.2 Widgets 的动态解析

[widget.go:95-124](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/widget.go#L95-L124) 实现了 `widgets` 类型的自定义 `UnmarshalYAML`：

```go
type widgets []widget

func (w *widgets) UnmarshalYAML(node *yaml.Node) error {
    var nodes []yaml.Node
    node.Decode(&nodes)

    for _, node := range nodes {
        // 第一步：先只读 type 字段
        meta := struct{ Type string `yaml:"type"` }{}
        node.Decode(&meta)

        // 第二步：根据 type 创建具体 widget 实例
        widget, err := newWidget(meta.Type)  // 工厂函数，switch-case 匹配类型

        // 第三步：将整个节点解码到具体 widget 实例
        node.Decode(widget)

        *w = append(*w, widget)
    }
    return nil
}
```

这是典型的**多态反序列化**模式：先读取 discriminator 字段（`type`），再通过工厂创建对应类型，最后完整解码。

### 4.3 自定义 YAML 字段类型

在 [config-fields.go](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/config-fields.go) 中实现了多个实现 `UnmarshalYAML` 的自定义类型：

#### hslColorField - HSL 颜色解析

```go
type hslColorField struct { H, S, L float64 }
```

支持格式：`hsl(240, 13%, 95%)`、`240 13% 95%`、`240, 13, 95` 等
- 正则提取 H/S/L 三个数值
- 校验范围：H∈[0,360], S∈[0,100], L∈[0,100]

#### durationField - 时长解析

```go
type durationField time.Duration
```

支持格式：`30s`、`5m`、`2h`、`1d`
- 正则提取数值和单位（s/m/h/d）
- 转换为 `time.Duration`

#### customIconField - 图标 URL 解析

```go
type customIconField struct {
    URL        template.URL
    AutoInvert bool
}
```

支持多种图标源前缀：
- `si:github` → Simple Icons CDN + 自动反色
- `di:docker` → Homarr Dashboard Icons CDN
- `mdi:home` → Material Design Icons CDN + 自动反色
- `sh:something` → Selfhst Icons CDN
- `auto-invert <url>` → 指定 URL 并启用自动反色
- 普通 URL → 直接使用

#### proxyOptionsField - 代理配置（双模式）

```go
type proxyOptionsField struct {
    URL           string        `yaml:"url"`
    AllowInsecure bool          `yaml:"allow-insecure"`
    Timeout       durationField `yaml:"timeout"`
    client        *http.Client  `yaml:"-"`
}
```

支持两种配置方式：
```yaml
# 简洁模式：直接写 URL 字符串
proxy: http://proxy:8080

# 完整模式：对象配置
proxy:
  url: http://proxy:8080
  allow-insecure: true
  timeout: 30s
```

实现技巧：先尝试解码为 string，失败再尝试解码为 struct alias。

#### queryParametersField - 查询参数（多类型值）

```go
type queryParametersField map[string][]string
```

支持值类型：string、数字、bool、字符串数组、混合数组
- 自动将标量转为单元素数组
- 统一转为 `map[string][]string`，便于 `url.Values` 使用

#### orderedYAMLMap - 保序映射

[config.go:540-637](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/config.go#L540-L637) 是一个泛型保序映射：

```go
type orderedYAMLMap[K comparable, V any] struct {
    keys []K          // 保持插入顺序
    data map[K]V
}
```

- 自定义 `UnmarshalYAML` 手动遍历 `yaml.Node.Content`，保留 key 顺序
- 提供 `Items()` 返回 `iter.Seq2` 用于 `range` 遍历
- `Merge()` 方法合并两个保序映射，保留原顺序，后加入的 key 追加到末尾
- 用于主题 Presets，确保主题选择器中主题顺序与配置文件一致

---

## 五、默认值合并策略

### 5.1 分层默认值

默认值在多个阶段设置，按优先级从低到高：

#### 阶段一：结构体零值 + 硬编码默认

在 [newConfigFromYAML](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/config.go#L94-L129) 中：

```go
config := &config{}
config.Server.Port = 8080   // 唯一在 yaml.Unmarshal 之前设置的默认值
```

其他字段依赖 Go 零值（false、""、nil、0）。

#### 阶段二：Widget 级默认

在各 widget 的 `initialize()` 方法中设置，例如 [widget-clock.go:22-44](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/widget-clock.go#L22-L44)：

```go
func (widget *clockWidget) initialize() error {
    widget.withTitle("Clock").withError(nil)  // 默认标题
    if widget.HourFormat == "" {
        widget.HourFormat = "24h"             // 默认 24 小时制
    }
    // ...
}
```

使用 builder 模式的 `withXxx()` 方法（定义在 [widget.go:243-291](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/widget.go#L243-L291)），这些方法遵循"仅当未设置时才赋值"原则：

```go
func (w *widgetBase) withTitle(title string) *widgetBase {
    if w.Title == "" {
        w.Title = title
    }
    return w
}
```

#### 阶段三：应用级默认与规范化

在 [newApplication](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/glance.go#L47-L231) 中：

| 字段 | 默认值逻辑 |
|------|-----------|
| `page.Slug` | `titleToSlug(page.Title)` （用户未配置时） |
| `page.Width` | `"default"` → 置空字符串 |
| `page.DesktopNavigationWidth` | 未设置时继承 `page.Width` |
| `config.Branding.AppName` | `"Glance"` |
| `config.Branding.AppIconURL` | 默认 app-icon.png |
| `config.Branding.AppBackgroundColor` | 主题背景色的 HEX 值 |
| `config.Branding.FaviconURL` | 默认 favicon.svg |
| `splitColumnWidget.MaxColumns` | `< 2` 时设为 `2` |

#### 阶段四：主题默认值

在 `newApplication` 的主题初始化部分：
- 自动注入 `default-dark` 和 `default-light` 两个内置预设
- 用户自定义 Presets 通过 `orderedYAMLMap.Merge()` 合并（用户配置覆盖内置同名主题）
- `themeProperties.init()` 生成 CSS 和预览 HTML

### 5.2 容器类 Widget 的子 Widget 默认传递

[containerWidgetBase](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/widget-container.go)（被 group、split-column 等容器 widget 内嵌）：

```go
func (widget *containerWidgetBase) _initializeWidgets() error {
    for i := range widget.Widgets {
        if err := widget.Widgets[i].initialize(); err != nil {  // 递归调用子 widget 初始化
            return formatWidgetInitError(err, widget.Widgets[i])
        }
    }
    return nil
}
```

---

## 六、错误聚合机制

### 6.1 错误处理阶段

整个配置加载过程中，错误在以下层次被捕获和包装：

| 阶段 | 函数 | 错误包装方式 |
|------|------|-------------|
| Include 解析 | `recursiveParseYAMLIncludes` | `fmt.Errorf("reading %s: %w", path, err)` 用 `%w` 包装保留原始错误链 |
| 变量插值 | `parseConfigVariables` | `fmt.Errorf("parsing variable: %v", localErr)` 记录具体变量错误 |
| YAML 语法 | `yaml.Unmarshal` | 直接返回 yaml 库错误（含行号） |
| 配置逻辑校验 | `isConfigStateValid` | 立即返回首个错误，不聚合 |
| Widget 初始化 | `newConfigFromYAML` | `formatWidgetInitError(err, w)` → `"{type} widget: {err}"` |
| 应用初始化 | `newApplication` | 每个子系统加前缀，如 `"decoding secret-key: ..."`, `"initializing preset theme %s: ..."` |

### 6.2 formatWidgetInitError

[config.go:236-238](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/config.go#L236-L238)：

```go
func formatWidgetInitError(err error, w widget) error {
    return fmt.Errorf("%s widget: %v", w.GetType(), err)
}
```

为 widget 初始化错误添加类型前缀，便于定位。

### 6.3 错误处理策略

**快速失败（Fail Fast）**：所有阶段遇到错误立即终止并返回，不继续处理后续内容。这意味着：
- 不会报告多个错误，只返回第一个遇到的错误
- 优点是实现简单、不会级联错误
- 缺点是用户可能需要多次修复才能看到所有问题

唯一的例外是 widget 渲染阶段的错误容错（见 [widget.go:217-241](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/widget.go#L217-L241)）：

```go
func (w *widgetBase) renderTemplate(data any, t *template.Template) template.HTML {
    err := t.Execute(&w.templateBuffer, data)
    if err != nil {
        // 重置 buffer，尝试再次渲染（此时 widget.Error 已设置，模板显示错误 UI）
        w.templateBuffer.Reset()
        t.Execute(&w.templateBuffer, data)
        // 如果再次失败，buffer 可能为空或半渲染
    }
    return template.HTML(w.templateBuffer.String())
}
```

渲染失败时尝试二次渲染以展示错误信息，避免单个 widget 崩溃导致整个页面布局损坏。

### 6.4 isConfigStateValid 校验清单

[config.go:451-537](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/config.go#L451-L537) 中执行的校验：

**全局校验：**
- 至少配置 1 个 page
- 配置了 users 时必须设置 secret-key
- 用户名非空且 ≥ 3 字符
- 每个用户必须有 password 或 password-hash，密码 ≥ 6 字符
- assets-path 目录必须存在

**Page 级校验：**
- page.name 非空
- page.width ∈ {"wide", "slim", "default", ""}
- page.desktop-navigation-width ∈ {"wide", "slim", "default", ""}
- 至少配置 1 个 column
- slim 页面最多 2 列，其他最多 3 列
- full-size 列数量必须是 1 或 2（不能为 0 或 > 2）

**Column 级校验：**
- column.size ∈ {"small", "full"}

---

## 七、配置加载完整调用链

以 CLI 启动为例的完整调用顺序：

```
main.go → cli.go
    ↓
loadConfig() (cli.go 中)
    ↓
parseYAMLIncludes(configFilePath)
    └── recursiveParseYAMLIncludes() × N
    ↓
parseConfigVariables(mergedContents)
    ↓
newConfigFromYAML(processedContents)
    ├── config.Server.Port = 8080
    ├── yaml.Unmarshal(contents, config)
    ├── isConfigStateValid(config)
    └── 遍历 pages → headWidgets / columns → widgets
        └── widget.initialize() （含默认值设置）
    ↓
newApplication(config)
    ├── Auth 初始化（密码哈希、密钥解码）
    ├── Theme 初始化（预设合并、CSS 生成）
    ├── Pages 预处理（slug、宽度、主列索引、provider 注入）
    ├── Branding 默认值填充
    └── manifest.json 预渲染
    ↓
server() → 注册路由 → ListenAndServe
```

---

## 八、配置热重载机制

### 8.1 整体架构

Glance 支持配置文件（含所有 include 的文件）变更时自动热重载，无需重启进程。核心由三部分协作：

```
┌───────────────────────────────────────────────────────────────┐
│  serveApp()  [main.go]                                         │
│                                                               │
│  parseYAMLIncludes() ── 首次解析，获取初始内容和 include 集合   │
│          │                                                    │
│          ▼                                                    │
│  configFilesWatcher() ── 启动 fsnotify 监听所有相关文件        │
│          │                                                    │
│          ├── onChange(newContents)                            │
│          │     └── 成功 → 停旧服务 → 创建新 app → 启新服务     │
│          │     └── 失败 → 记录日志 → 维持旧服务运行（见第九节） │
│          │                                                    │
│          └── onErr(err)                                       │
│                └── 仅记录日志，不中断服务                      │
└───────────────────────────────────────────────────────────────┘
```

### 8.2 serveApp 启动流程

[internal/glance/main.go:93-181](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/main.go#L93-L181) 是服务启动和热重载的总控函数：

```go
func serveApp(configPath string) error {
    exitChannel := make(chan struct{})     // 进程退出信号
    hadValidConfigOnStartup := false       // ← 关键状态标志
    var stopServer func() error            // 当前运行服务的停止函数

    // ── onChange：配置变更回调 ──
    onChange := func(newContents []byte) {
        // 详见第九节
    }

    // ── onErr：文件监听错误回调 ──
    onErr := func(err error) {
        log.Printf("Error watching config files: %v", err)
    }

    // ── 首次解析配置 ──
    configContents, configIncludes, err := parseYAMLIncludes(configPath)
    if err != nil {
        return fmt.Errorf("parsing config: %w", err)
    }

    // ── 启动文件监听器 ──
    stopWatching, err := configFilesWatcher(configPath, configContents, configIncludes, onChange, onErr)
    if err == nil {
        defer stopWatching()               // 正常：通过热重载启动
    } else {
        // 降级路径：监听器启动失败（如 fsnotify 不支持）
        // 跳过热重载，直接加载配置启动一次
        log.Printf("Error starting file watcher, config file changes will require a manual restart. (%v)", err)

        config, err := newConfigFromYAML(configContents)
        if err != nil {
            return fmt.Errorf("validating config file: %w", err)
        }
        app, err := newApplication(config)
        if err != nil {
            return fmt.Errorf("creating application: %w", err)
        }
        startServer, _ := app.server()
        if err := startServer(); err != nil {
            return fmt.Errorf("starting server: %w", err)
        }
    }

    <-exitChannel   // 阻塞等待退出信号
    return nil
}
```

### 8.3 configFilesWatcher 文件监听详解

[config.go:305-445](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/config.go#L305-L445) 实现了复杂的文件监听逻辑：

#### 核心状态

```go
func configFilesWatcher(
    mainFilePath string,
    lastContents []byte,           // 上次成功的合并内容（用于内容对比）
    lastIncludes map[string]struct{},  // 上次监听的文件集合（用于动态增删监听）
    onChange func(newContents []byte),
    onErr func(error),
) (func() error, error)
```

#### 防抖机制

```go
const debounceDuration = 500 * time.Millisecond
var debounceTimer *time.Timer

debouncedParseAndCompareBeforeCallback := func() {
    if debounceTimer != nil {
        debounceTimer.Stop()
        debounceTimer.Reset(debounceDuration)  // 重置计时器
    } else {
        debounceTimer = time.AfterFunc(debounceDuration, parseAndCompareBeforeCallback)
    }
}
```

目的：编辑器保存文件时可能触发多次 Write 事件（如 atomic rename 会写临时文件再 rename），500ms 防抖避免重复解析。

#### 内容对比与监听集合动态更新

```go
parseAndCompareBeforeCallback := func() {
    // 重新解析所有 include，得到最新内容和文件集合
    currentContents, currentIncludes, err := parseYAMLIncludes(mainFilePath)
    // ...

    mu.Lock()
    defer mu.Unlock()

    // 如果 include 集合变了（新增或删除了 include 文件）
    if !maps.Equal(currentIncludes, lastIncludes) {
        updateWatchedFiles(lastIncludes, currentIncludes)  // 增删 watcher
        lastIncludes = currentIncludes
    }

    // 只有文件内容真正变化才触发 onChange
    if !bytes.Equal(lastContents, currentContents) {
        lastContents = currentContents
        onChange(currentContents)
    }
}
```

**二级对比策略**：
1. **Include 集合对比**：用户新增/删除 `$include` 引用时，动态更新 fsnotify 监听的文件列表
2. **内容字节对比**：避免 include 文件没变但主文件时间戳变化（或其他无意义变更）触发无效重载

#### 文件事件处理

```go
case event.Has(fsnotify.Write):
    debouncedParseAndCompareBeforeCallback()       // 防抖后解析

case event.Has(fsnotify.Rename):
    deleteLastInclude(event.Name)                  // 从 tracked set 移除旧路径
    // 等 2 秒（10 × 200ms）看文件会不会重新出现（编辑器 atomic save 的典型行为）
    for range 10 {
        if _, err := os.Stat(event.Name); err == nil { break }
        time.Sleep(200 * time.Millisecond)
    }
    debouncedParseAndCompareBeforeCallback()

case event.Has(fsnotify.Remove):
    deleteLastInclude(event.Name)
    debouncedParseAndCompareBeforeCallback()
```

**Rename 事件的特殊处理**：
- Linux 下，很多编辑器使用"写临时文件 + rename 覆盖"的原子保存方式
- Rename 后 fsnotify 在 Linux 上会丢失对原路径的监听
- 代码主动等待 2 秒看文件是否重新出现，然后触发重新解析（重新解析时会重新添加新文件到 watcher）

---

## 九、校验失败时维持旧状态的处理流程

### 9.1 核心设计目标

热重载过程中，如果用户提交了一份有错误的配置（语法错误、字段不合法、widget 初始化失败等），**不能让正在运行的服务崩溃或停止**。必须保证：

1. 首次启动就配置错误 → 进程直接退出（无可维持的旧状态）
2. 运行中配置变更出错 → 仅记录日志，旧服务继续运行，等用户修复后再次变更时再尝试重载

### 9.2 onChange 回调的状态机

[internal/glance/main.go:101-146](file:///d:/fz/0601/solo-dogfeeding/code/134-glance/internal/glance/main.go#L101-L146) 中 `onChange` 的完整逻辑：

```go
onChange := func(newContents []byte) {
    if stopServer != nil {
        log.Println("Config file changed, reloading...")
    }

    // ── 关卡 1：解析 + 校验配置 ──
    config, err := newConfigFromYAML(newContents)
    if err != nil {
        log.Printf("Config has errors: %v", err)

        if !hadValidConfigOnStartup {   // ← 关键判断
            close(exitChannel)           // 首次启动失败 → 退出进程
        }
        return                          // 运行中失败 → 直接 return，不碰旧服务
    }

    // ── 关卡 2：创建 application（二次校验 + 初始化） ──
    app, err := newApplication(config)
    if err != nil {
        log.Printf("Failed to create application: %v", err)

        if !hadValidConfigOnStartup {
            close(exitChannel)
        }
        return
    }

    // ── 成功：更新状态 ──
    if !hadValidConfigOnStartup {
        hadValidConfigOnStartup = true   // 标记首次成功，后续永远走"维持旧状态"分支
    }

    // ── 关卡 3：优雅切换 ──
    if stopServer != nil {
        if err := stopServer(); err != nil {   // 先停旧服务
            log.Printf("Error while trying to stop server: %v", err)
        }
    }

    // 启动新服务（goroutine，因为 ListenAndServe 是阻塞的）
    go func() {
        var startServer func() error
        startServer, stopServer = app.server()   // 更新 stopServer 到新服务的

        if err := startServer(); err != nil {
            log.Printf("Failed to start server: %v", err)
        }
    }()
}
```

### 9.3 hadValidConfigOnStartup 标志的语义

这个布尔标志是整个容错机制的核心：

| 值 | 含义 | 校验失败时行为 |
|----|------|-------------|
| `false` | 进程启动后尚未有过任何一份有效配置成功运行 | `close(exitChannel)` → 进程退出，返回非零码 |
| `true` | 至少有一份有效配置已经成功启动过服务 | 仅 `log.Printf` 记录错误，`return` 跳过本次变更，旧服务继续运行 |

**状态转换**：`false` → `true` 是单向的，一旦变为 `true` 就永远不会回到 `false`。这意味着只要服务曾经成功启动过一次，之后任何配置错误都不会导致进程退出。

### 9.4 完整状态转换图

```
                        ┌───────────────┐
                        │  进程启动     │
                        └───────┬───────┘
                                │
                    hadValidConfigOnStartup = false
                                │
                                ▼
                  onChange 被首次触发（由 watcher 启动时调用）
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
              解析/校验成功             解析/校验失败
                    │                       │
     hadValidConfigOnStartup = true    close(exitChannel)
           startServer()                     进程退出
                    │
                    ▼
              ┌──────────┐
              │ 服务运行 │ ◄──────────┐
              └────┬─────┘            │
                   │                  │
         用户修改配置文件              │
                   │                  │
                   ▼                  │
        fsnotify 触发 onChange        │
                   │                  │
          ┌────────┴────────┐         │
          ▼                 ▼         │
      成功              校验失败       │
          │                 │         │
   stopServer()      log 错误 ────────┘
   创建新 app
   startServer()
          │
          ▼
     继续运行（新配置）
```

### 9.5 资源生命周期管理

切换过程中的资源清理：

```go
if stopServer != nil {
    if err := stopServer(); err != nil {
        log.Printf("Error while trying to stop server: %v", err)
    }
}
// 旧的 *application 对象失去引用，由 GC 回收
// 旧的 widget、http.Server、监听器等都随 stopServer() 关闭

go func() {
    startServer, stopServer = app.server()   // stopServer 变量被覆盖为新函数
    startServer()
}()
```

注意事项：
- `stopServer` 是闭包捕获的变量，始终指向当前运行服务的停止函数
- 旧服务 `stopServer()` 调用 `http.Server.Close()`，会关闭所有 listener 并等待活跃请求完成
- 旧 `application` 对象及其所有 widget 缓存、主题配置等变为不可达，由 Go GC 回收
- `stopWatching` 通过 `defer` 在 `serveApp` 返回时调用（即进程退出时）

### 9.6 降级路径下的容错

当 `configFilesWatcher` 本身启动失败（如某些不支持 inotify 的环境），代码走降级路径：

```go
} else {
    log.Printf("Error starting file watcher, config file changes will require a manual restart. (%v)", err)

    // 直接加载一次配置，不支持热重载
    config, err := newConfigFromYAML(configContents)
    if err != nil {
        return fmt.Errorf("validating config file: %w", err)  // 启动失败直接返回错误
    }
    app, err := newApplication(config)
    if err != nil {
        return fmt.Errorf("creating application: %w", err)
    }
    startServer, _ := app.server()
    if err := startServer(); err != nil {
        return fmt.Errorf("starting server: %w", err)
    }
}
```

降级路径下：
- 无热重载，`hadValidConfigOnStartup` 逻辑不生效
- 首次启动失败直接返回 error → 进程退出
- 一旦启动成功就一直运行，直到进程被外部终止
