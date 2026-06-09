# Glance Docker 运行时约定

本文档从代码层面梳理 Glance 的 Docker 部署参数、运行入口、配置挂载、权限模型与多架构构建约定。

---

## 1. 容器启动流程

### 1.1 ENTRYPOINT 与 Docker 参数拼接

Glance 的两个 Dockerfile 均使用 **exec 形式**的 ENTRYPOINT 定义：

- [Dockerfile](Dockerfile#L13-L13)
- [Dockerfile.goreleaser](Dockerfile.goreleaser#L7-L7)

```dockerfile
ENTRYPOINT ["/app/glance", "--config", "/app/config/glance.yml"]
```

**Docker exec 形式 ENTRYPOINT 的关键行为**：当通过 `docker run glanceapp/glance <user-args>` 追加参数时，`<user-args>` 会被**追加**到 ENTRYPOINT 数组的尾部，而非覆盖。即最终进程的 `os.Args` 为：

```
["/app/glance", "--config", "/app/config/glance.yml", ...<user-args>
```

这意味着：
- 不追加任何参数时 → `os.Args[1:] = ["--config", "/app/config/glance.yml"]`
- 追加 `config:validate` → `os.Args[1:] = ["--config", "/app/config/glance.yml", "config:validate"]`
- 追加 `--version` → `os.Args[1:] = ["--config", "/app/config/glance.yml", "--version"]`
- 追加 `--config /other.yml serve` → `os.Args[1:] = ["--config", "/app/config/glance.yml", "--config", "/other.yml", "serve"]`

由于使用 `ENTRYPOINT` 而非 `CMD`，无法通过 `docker run` 参数覆盖掉内置的 `--config /app/config/glance.yml`；如需使用自定义配置路径，必须在追加参数中再次指定 `--config`（flag 包会取最后一次出现的值，见下节）。

---

### 1.2 CLI 参数解析机制详解

CLI 参数解析位于 [cli.go:parseCliOptions](internal/glance/cli.go#L33-L107)，整个解析分为三个阶段，存在多个需要注意的交互行为。

#### 阶段一：`--version` 短路判断（第 36-41 行）

```go
args = os.Args[1:]
if len(args) == 1 && (args[0] == "--version" || args[0] == "-v" || args[0] == "version") {
    return &cliOptions{intent: cliIntentVersionPrint}, nil
}
```

**该判断在 flag 解析之前执行，且仅当 `len(args) == 1` 时触发。**

| 调用方式 | `os.Args[1:]` | len | 是否触发短路 |
|---|---|---|---|
| 二进制直接 `glance --version` | `["--version"]` | 1 | ✅ 触发 |
| 二进制直接 `glance -v` | `["-v"]` | 1 | ✅ 触发 |
| 二进制直接 `glance version` | `["version"]` | 1 | ✅ 触发 |
| 二进制直接 `glance --version --config x.yml` | `["--version", "--config", "x.yml"]` | 3 | ❌ 不触发 |
| **Docker** `docker run ... --version` | `["--config", "/app/config/glance.yml", "--version"]` | 3 | ❌ 不触发 |
| **Docker** `docker run ... -v` | `["--config", "/app/config/glance.yml", "-v"]` | 3 | ❌ 不触发 |

> **重要结论**：在 Docker 环境下，由于 ENTRYPOINT 已内置了 `--config` 参数，`len(args)` 至少为 2，**`--version` / `-v` / `version` 的短路判断永远不会命中**。此时这些参数会进入后续的 flag 解析流程。

#### 阶段二：flag 包解析（第 43-64 行）

```go
flags := flag.NewFlagSet("", flag.ExitOnError)
configPath := flags.String("config", "glance.yml", "Set config path")
err := flags.Parse(os.Args[1:])
```

使用 Go 标准库 `flag` 包，行为特点：

1. **遇到第一个非 flag 参数即停止解析**，剩余参数通过 `flags.Args()` 返回
2. 同一 flag 多次出现时，**最后一个值生效**
3. 解析失败时直接 `os.Exit(2)`（由 `flag.ExitOnError` 决定）

| 调用方式（Docker 环境） | flag 解析结果 | `flags.Args()`（剩余非 flag 参数） |
|---|---|---|
| 无追加参数 | `configPath = "/app/config/glance.yml"` | `[]`（空） |
| 追加 `config:validate` | `configPath = "/app/config/glance.yml"` | `["config:validate"]` |
| 追加 `--config /other.yml config:validate` | `configPath = "/other.yml"`（最后一个生效） | `["config:validate"]` |
| 追加 `config:validate --config /other.yml` | `configPath = "/app/config/glance.yml"`（解析在 config:validate 处停止） | `["config:validate", "--config", "/other.yml"]` |
| 追加 `--version` | flag 包将 `--version` 视为未知 flag → `flag.ExitOnError` 触发退出，打印 `flag provided but not defined: -version` | - |
| 追加 `-v` | flag 包将 `-v` 视为未知 flag → 同上，退出码 2 | - |

> **关键发现**：在 Docker 环境下执行 `docker run ... --version` 会**直接报错退出**，而非打印版本号。这是 `--version` 短路判断与 Docker ENTRYPOINT 共同作用下的设计缺陷。

#### 阶段三：子命令分发（第 66-100 行）

根据 `flags.Args()` 返回的剩余参数数量进行分支匹配：

```go
args = flags.Args()

if len(args) == 0 {
    intent = cliIntentServe          // 默认：启动 Web 服务
} else if len(args) == 1 {
    // 匹配 config:validate / config:print / sensors:print / diagnose / secret:make
} else if len(args) == 2 {
    // 匹配 password:hash
} else if len(args) == 2 {        // ⚠️ 死代码，永远不会执行
    // 匹配 mountpoint:info
} else {
    return nil, unknownCommandErr
}
```

#### 完整子命令对照表（二进制直接调用场景）：

| 调用形式 | 剩余 args len | 解析结果 | 对应 intent |
|---|---|---|---|
| `glance`（无参数） | 0 | intent = `cliIntentServe`，configPath = `glance.yml` | 启动 Web 服务 |
| `glance --config /path/config.yml` | 0 | intent = `cliIntentServe`，configPath = `/path/config.yml` | 启动 Web 服务（指定配置） |
| `glance --version` / `-v` / `version` | — | 阶段一短路，直接返回 | 打印版本号 |
| `glance config:validate` | 1 | intent = `cliIntentConfigValidate` | 验证配置文件 |
| `glance config:print` | 1 | intent = `cliIntentConfigPrint` | 打印展开 include 后的配置 |
| `glance secret:make` | 1 | intent = `cliIntentSecretMake` | 生成认证密钥 |
| `glance sensors:print` | 1 | intent = `cliIntentSensorsPrint` | 列出传感器 |
| `glance diagnose` | 1 | intent = `cliIntentDiagnose` | 运行诊断 |
| `glance password:hash <pwd>` | 2 | intent = `cliIntentPasswordHash`，args[1] = `<pwd>` | 生成密码哈希 |
| `glance mountpoint:info <path>` | 2 | **见下节分析** | — |

启动入口在 [main.go:Main](internal/glance/main.go#L15-L91)，根据 intent 分发：

```
ENTRYPOINT → glance.Main() → parseCliOptions() → cliIntentServe → serveApp(configPath)
```

---

### 1.3 `mountpoint:info` 子命令可达性分析

**结论：`mountpoint:info` 子命令在当前代码中是不可达的（死代码）。**

问题出在 [cli.go:86-97](internal/glance/cli.go#L86-L97) 的条件分支结构：

```go
} else if len(args) == 2 {
    if args[0] == "password:hash" {
        intent = cliIntentPasswordHash
    } else {
        return nil, unknownCommandErr
    }
} else if len(args) == 2 {   // ← 与上一个分支条件完全相同，永远不会进入
    if args[0] == "mountpoint:info" {
        intent = cliIntentMountpointInfo
    } else {
        return nil, unknownCommandErr
    }
}
```

两个连续的 `else if len(args) == 2` 分支条件完全相同。当 `len(args) == 2` 时，控制权必然进入第一个分支：
- 若 `args[0] == "password:hash"` → 设置 intent 为 `cliIntentPasswordHash`，跳出分支
- 否则 → 返回 `unknown command` 错误，函数提前返回

第二个 `else if len(args) == 2` 分支在语法上合法但逻辑上永远不会被执行。

**实际运行表现**：

| 调用方式 | 预期行为 | 实际行为 |
|---|---|---|
| `glance mountpoint:info /app` | 打印挂载点信息 | 输出 `unknown command: mountpoint:info /app`，退出码 1 |
| Docker 环境下同上 | 同上 | 同上 |

虽然 `cliIntentMountpointInfo` 在 [main.go:56-57](internal/glance/main.go#L56-L57) 的 switch 中有对应分支，且 `cliMountpointInfo()` 函数本身实现完整，但由于解析阶段永远不会产生该 intent，这段代码同样不可达。

---

### 1.4 服务启动流程

核心启动逻辑在 [main.go:serveApp](internal/glance/main.go#L93-L181)：

1. **解析配置**：调用 `parseYAMLIncludes(configPath)` 递归展开所有 `!include:` / `$include:` 指令，最多递归 20 层（见 [config.go:CONFIG_INCLUDE_RECURSION_DEPTH_LIMIT](internal/glance/config.go#L22-L22)）
2. **启动文件监听器**：`configFilesWatcher()` 使用 fsnotify 监控主配置及所有被 include 的文件，500ms 防抖（见 [config.go:configFilesWatcher](internal/glance/config.go#L305-L445)）
3. **首次加载**：`onChange` 回调被触发，调用 `newConfigFromYAML()` 验证配置并调用 `newApplication()` 构建应用实例
4. **启动 HTTP 服务器**：`app.server()` 返回 start/stop 闭包，监听地址由 `server.host` 和 `server.port` 决定（默认端口 8080，见 [config.go:newConfigFromYAML](internal/glance/config.go#L101-L101)）
5. **配置热重载**：文件变更后停止旧 server、重建 application、启动新 server

若监听器启动失败（例如某些文件系统不支持 inotify），则退化为一次性加载模式，配置变更需手动重启容器。

---

## 2. 配置挂载约定

### 2.1 卷挂载路径

官方推荐的 docker-compose 挂载（见 [README.md](README.md#L209-L212)）：

```yaml
volumes:
  - ./config:/app/config
```

容器内约定：

| 容器内路径 | 用途 | 是否必须 |
|---|---|---|
| `/app/config/glance.yml` | 主配置文件（由 ENTRYPOINT 的 `--config` 指定） | 是 |
| `/app/config/*.yml` | 被主配置 `!include:` 的子配置文件 | 按需 |
| `/app/assets/` | 用户自定义资源目录（由配置 `server.assets-path` 指定） | 否 |
| `/run/secrets/<name>` | Docker secrets 挂载目录，由 `${secret:name}` 语法读取 | 否 |

### 2.2 配置文件查找

- 默认查找路径由 `--config` 显式指定，**不会**自动搜索当前工作目录（ENTRYPOINT 已锁定路径）
- 若主配置不存在，Docker 环境下会触发 v0.7.0 升级提示页面（见 [main.go:serveUpdateNoticeIfConfigLocationNotMigrated](internal/glance/main.go#L183-L219)），监听 8080 端口返回 503 和升级说明

### 2.3 配置变量注入

在 [config.go:parseConfigVariables](internal/glance/config.go#L142-L187) 中支持三种变量语法：

| 语法 | 解析规则 | 典型用途 |
|---|---|---|
| `${API_KEY}` | 读取同名环境变量，变量名须匹配 `^[A-Z0-9_]+$` | docker-compose environment 传参 |
| `${secret:db_password}` | 从 `/run/secrets/db_password` 读取文件内容并去除首尾空白 | Docker Swarm / Compose secrets |
| `${readFileFromEnv:SECRET_PATH}` | 从环境变量 `SECRET_PATH` 获取绝对路径，再读取该文件内容 | 自定义 secrets 挂载位置 |

若变量解析失败（环境变量不存在 / 文件不可读），应用会启动失败并打印错误。

### 2.4 配置热重载细节

文件监听基于 `fsnotify`：

- **监听范围**：主配置文件 + 所有通过 `!include:` 递归引入的文件
- **触发事件**：`Write`、`Rename`、`Remove`
- **防抖**：500ms 内多次变更合并为一次重载（[config.go:debounceDuration](internal/glance/config.go#L373-L373)）
- **重载失败策略**：启动后首次加载失败 → 容器退出；运行中重载失败 → 保留旧配置并打印日志，不中断服务

> **注意**：Windows 与 Linux 的 rename 语义存在差异，代码中已针对 Linux 的 rename-remove 模式做了重试补偿（[config.go](internal/glance/config.go#L400-L422)）。

### 2.5 健康检查端点

应用暴露 `/api/healthz` 端点（见 [glance.go](internal/glance/glance.go#L449-L451)），始终返回 200 OK，可用于 docker-compose healthcheck：

```yaml
healthcheck:
  test: ["CMD", "wget", "--spider", "-q", "http://localhost:8080/api/healthz"]
  interval: 30s
  timeout: 5s
  retries: 3
```

---

## 3. 非 root 权限

### 3.1 现状

当前两个 Dockerfile 均 **未创建非 root 用户**，也无 `USER` 指令：

- [Dockerfile](Dockerfile)
- [Dockerfile.goreleaser](Dockerfile.goreleaser)

因此容器默认以 `root` (uid=0) 身份运行，WORKDIR 为 `/app`。

### 3.2 以非 root 用户运行的可行方案

由于 Go 二进制为静态编译（`CGO_ENABLED=0`，见 [Dockerfile#L5](Dockerfile#L5-L5) 和 [.goreleaser.yaml#L9](.goreleaser.yaml#L9-L9)），不依赖系统库，可直接以任意 uid 运行。

**docker-compose 方式**：

```yaml
services:
  glance:
    image: glanceapp/glance
    user: "1000:1000"
    volumes:
      - ./config:/app/config:ro
    ports:
      - 8080:8080
```

**需注意**：
- 绑定 8080 端口无需特权（非特权端口 > 1024）
- 若配置了 `server.assets-path` 且需写入资源，则对应目录需对运行 uid 可写
- 配置卷建议以只读方式挂载（`:ro`），避免容器内进程意外修改配置

### 3.3 安全建议

| 措施 | 说明 |
|---|---|
| 使用 `user: <uid>:<gid>` | 避免以 root 运行 |
| 配置卷只读挂载 `:ro` | 防止配置被篡改 |
| `read_only: true` | 将容器根文件系统设为只读（需确认无临时文件写入） |
| `security_opt: [no-new-privileges:true]` | 禁止 setuid 提权 |
| `cap_drop: [ALL]` | 丢弃所有 Linux capabilities |

---

## 4. 多架构构建约定

### 4.1 GoReleaser 构建矩阵

镜像发布由 [.goreleaser.yaml](.goreleaser.yaml) 驱动，在 `builds` 节定义了 Go 交叉编译矩阵：

| goos | goarch | goarm | 说明 |
|---|---|---|---|
| linux | amd64 | - | x86_64 |
| linux | arm64 | - | ARM64 / aarch64 |
| linux | arm | 7 | ARMv7（树莓派 2/3） |
| openbsd/freebsd/windows/darwin | 多架构 | - | 仅发布二进制 tarball/zip，不构建 Docker 镜像 |

二进制编译参数：`CGO_ENABLED=0`，ldflags `-s -w` 去除符号表并注入版本号。

### 4.2 Docker 镜像构建

`.goreleaser.yaml` 的 `dockers` 节定义了三个独立架构镜像，均使用 [Dockerfile.goreleaser](Dockerfile.goreleaser) 作为构建模板，使用 `buildx`：

| 镜像 tag 模板 | 平台 | Go 架构 |
|---|---|---|
| `glanceapp/glance:<tag>-amd64` | `linux/amd64` | amd64 |
| `glanceapp/glance:<tag>-arm64` | `linux/arm64` | arm64 |
| `glanceapp/glance:<tag>-armv7` | `linux/arm/v7` | arm (goarm=7) |

`Dockerfile.goreleaser` 由 GoReleaser 在构建时将预编译的二进制复制进 alpine:3.22 基础镜像，不再执行 Go 编译。

### 4.3 Manifest 清单

`docker_manifests` 节将三个架构镜像组装为多架构清单：

- `glanceapp/glance:<tag>`：版本化 tag，始终推送
- `glanceapp/glance:latest`：latest tag，仅在推送正式 release tag 时推送（`skip_push: auto`）

### 4.4 CI 触发流程

发布流水线位于 [.github/workflows/release.yaml](.github/workflows/release.yaml)：

```
推送 v* tag → checkout → docker/login-action → setup-go → docker/setup-buildx-action → goreleaser/goreleaser-action release
```

GoReleaser 在单次运行中同时完成：
1. 多架构二进制编译与归档
2. 三个架构 Docker 镜像的 buildx 构建与推送
3. 多架构 manifest 的创建与推送
4. GitHub Release 创建与附件上传

### 4.5 本地手动构建

若不使用 GoReleaser，可直接使用项目根目录的 [Dockerfile](Dockerfile)：

```bash
# 构建当前平台镜像
docker build -t glanceapp/glance:local .

# 使用 buildx 多架构构建并推送
docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -t glanceapp/glance:local --push .
```

多阶段构建流程（[Dockerfile](Dockerfile)）：
1. **builder 阶段**：`golang:1.26.3-alpine3.22`，在工作目录下执行 `CGO_ENABLED=0 go build .`
2. **最终阶段**：`alpine:3.22`，仅拷贝二进制文件，WORKDIR=/app，EXPOSE 8080/tcp

### 4.6 .dockerignore 约定

[.dockerignore](.dockerignore) 使用黑名单模式：默认忽略全部，仅显式放行构建所需路径：

```
*
!/build/
!/internal/
!/pkg/
!/go.mod
!/go.sum
!main.go
```

Dockerfile 本身始终被隐式包含。这样可确保文档、CI 配置等大体积非代码文件不进入构建上下文。

---

## 5. 快速参考

### Docker 环境下各子命令调用方式

由于 Docker ENTRYPOINT 已内置 `--config /app/config/glance.yml`，需注意以下调用差异：

| 目的 | 二进制直接调用 | Docker 环境调用 | 是否可用 |
|---|---|---|---|
| 启动服务 | `glance` 或 `glance --config x.yml` | `docker run ... glanceapp/glance` | ✅ |
| 指定自定义配置路径 | `glance --config /other.yml` | `docker run ... glanceapp/glance --config /other.yml` | ✅（flag 最后值生效） |
| 查看版本 | `glance --version` | **不可直接使用**（会被 flag 包报错） | ❌（Docker 环境下） |
| 验证配置 | `glance config:validate` | `docker run --rm -v ./config:/app/config glanceapp/glance config:validate` | ✅ |
| 打印配置 | `glance config:print` | `docker run --rm -v ./config:/app/config glanceapp/glance config:print` | ✅ |
| 生成密钥 | `glance secret:make` | `docker run --rm glanceapp/glance secret:make` | ✅ |
| 哈希密码 | `glance password:hash mypass` | `docker run --rm glanceapp/glance password:hash mypass` | ✅ |
| 列出传感器 | `glance sensors:print` | `docker run --rm glanceapp/glance sensors:print` | ✅（容器内可见传感器有限） |
| 运行诊断 | `glance diagnose` | `docker run --rm glanceapp/glance diagnose` | ✅ |
| 挂载点信息 | `glance mountpoint:info /path` | **代码不可达**（当前版本 bug） | ❌ |

### 常用 docker run 命令

```bash
# 最简启动（需 ./config/glance.yml 存在）
docker run -d -p 8080:8080 -v ./config:/app/config glanceapp/glance

# 以非 root 运行 + 只读配置
docker run -d -p 8080:8080 -u 1000:1000 \
  -v ./config:/app/config:ro glanceapp/glance

# 验证配置文件（不启动服务）
docker run --rm -v ./config:/app/config glanceapp/glance config:validate

# 打印展开后的完整配置
docker run --rm -v ./config:/app/config glanceapp/glance config:print

# 生成认证密钥
docker run --rm glanceapp/glance secret:make

# 生成密码哈希
docker run --rm glanceapp/glance password:hash 'my-password'
```

### 关键路径速查表

| 项目 | 值 |
|---|---|
| 二进制路径 | `/app/glance` |
| 默认配置路径（ENTRYPOINT 锁定） | `/app/config/glance.yml` |
| 默认监听端口 | `8080/tcp` |
| 健康检查 | `GET /api/healthz` |
| Secrets 目录 | `/run/secrets/` |
| 基础镜像 | `alpine:3.22` |
| 工作目录 | `/app` |
| 支持的 Docker 架构 | `linux/amd64`, `linux/arm64`, `linux/arm/v7` |
