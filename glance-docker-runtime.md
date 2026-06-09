# Glance Docker 运行时约定

本文档从代码层面梳理 Glance 的 Docker 部署参数、运行入口、配置挂载、权限模型与多架构构建约定。

---

## 1. 容器启动流程

### 1.1 ENTRYPOINT 与默认参数

Glance 的两个 Dockerfile 均使用相同的 ENTRYPOINT 定义：

- [Dockerfile](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/Dockerfile#L13-L13)
- [Dockerfile.goreleaser](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/Dockerfile.goreleaser#L7-L7)

```dockerfile
ENTRYPOINT ["/app/glance", "--config", "/app/config/glance.yml"]
```

容器启动后会执行 `/app/glance` 二进制，并传入 `--config /app/config/glance.yml` 参数。由于使用的是 `ENTRYPOINT` 而非 `CMD`，通过 `docker run glanceapp/glance <args>` 追加的参数会被视为 **子命令**（而非覆盖默认参数）。

### 1.2 CLI 参数解析链路

CLI 参数解析位于 [cli.go:parseCliOptions](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/internal/glance/cli.go#L33-L107)。

| 调用形式 | 解析结果 | 对应 intent |
|---|---|---|
| `glance`（无参数，即 ENTRYPOINT 默认） | intent = `cliIntentServe`，configPath = `glance.yml`（被 --config 覆盖） | 启动 Web 服务 |
| `glance --config /path/config.yml` | intent = `cliIntentServe`，configPath = `/path/config.yml` | 启动 Web 服务（指定配置） |
| `glance --version` / `-v` / `version` | intent = `cliIntentVersionPrint` | 打印版本号 |
| `glance config:validate` | intent = `cliIntentConfigValidate` | 验证配置文件 |
| `glance config:print` | intent = `cliIntentConfigPrint` | 打印展开 include 后的配置 |
| `glance secret:make` | intent = `cliIntentSecretMake` | 生成认证密钥 |
| `glance password:hash <pwd>` | intent = `cliIntentPasswordHash` | 生成密码哈希 |
| `glance sensors:print` | intent = `cliIntentSensorsPrint` | 列出传感器 |
| `glance diagnose` | intent = `cliIntentDiagnose` | 运行诊断 |
| `glance mountpoint:info <path>` | intent = `cliIntentMountpointInfo` | 打印挂载点信息 |

启动入口在 [main.go:Main](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/internal/glance/main.go#L15-L91)，根据 intent 分发：

```
ENTRYPOINT → glance.Main() → parseCliOptions() → cliIntentServe → serveApp(configPath)
```

### 1.3 服务启动流程

核心启动逻辑在 [main.go:serveApp](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/internal/glance/main.go#L93-L181)：

1. **解析配置**：调用 `parseYAMLIncludes(configPath)` 递归展开所有 `!include:` / `$include:` 指令，最多递归 20 层（见 [config.go:CONFIG_INCLUDE_RECURSION_DEPTH_LIMIT](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/internal/glance/config.go#L22-L22)）
2. **启动文件监听器**：`configFilesWatcher()` 使用 fsnotify 监控主配置及所有被 include 的文件，500ms 防抖（见 [config.go:configFilesWatcher](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/internal/glance/config.go#L305-L445)）
3. **首次加载**：`onChange` 回调被触发，调用 `newConfigFromYAML()` 验证配置并调用 `newApplication()` 构建应用实例
4. **启动 HTTP 服务器**：`app.server()` 返回 start/stop 闭包，监听地址由 `server.host` 和 `server.port` 决定（默认端口 8080，见 [config.go:newConfigFromYAML](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/internal/glance/config.go#L101-L101)）
5. **配置热重载**：文件变更后停止旧 server、重建 application、启动新 server

若监听器启动失败（例如某些文件系统不支持 inotify），则退化为一次性加载模式，配置变更需手动重启容器。

---

## 2. 配置挂载约定

### 2.1 卷挂载路径

官方推荐的 docker-compose 挂载（见 [README.md](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/README.md#L209-L212)）：

```yaml
volumes:
  - ./config:/app/config
```

容器内约定：

| 路径 | 用途 | 是否必须 |
|---|---|---|
| `/app/config/glance.yml` | 主配置文件（由 ENTRYPOINT 的 `--config` 指定） | 是 |
| `/app/config/*.yml` | 被主配置 `!include:` 的子配置文件 | 按需 |
| `/app/assets/` | 用户自定义资源目录（由配置 `server.assets-path` 指定） | 否 |
| `/run/secrets/<name>` | Docker secrets 挂载目录，由 `${secret:name}` 语法读取 | 否 |

### 2.2 配置文件查找

- 默认查找路径由 `--config` 显式指定，**不会**自动搜索当前工作目录（ENTRYPOINT 已锁定路径）
- 若主配置不存在，Docker 环境下会触发 v0.7.0 升级提示页面（见 [main.go:serveUpdateNoticeIfConfigLocationNotMigrated](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/internal/glance/main.go#L183-L219)），监听 8080 端口返回 503 和升级说明

### 2.3 配置变量注入

在 [config.go:parseConfigVariables](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/internal/glance/config.go#L142-L187) 中支持三种变量语法：

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
- **防抖**：500ms 内多次变更合并为一次重载（[config.go:debounceDuration](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/internal/glance/config.go#L373-L373)）
- **重载失败策略**：启动后首次加载失败 → 容器退出；运行中重载失败 → 保留旧配置并打印日志，不中断服务

> **注意**：Windows 与 Linux 的 rename 语义存在差异，代码中已针对 Linux 的 rename-remove 模式做了重试补偿（[config.go](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/internal/glance/config.go#L400-L422)）。

### 2.5 健康检查端点

应用暴露 `/api/healthz` 端点（见 [glance.go](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/internal/glance/glance.go#L449-L451)），始终返回 200 OK，可用于 docker-compose healthcheck：

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

- [Dockerfile](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/Dockerfile)
- [Dockerfile.goreleaser](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/Dockerfile.goreleaser)

因此容器默认以 `root` (uid=0) 身份运行，WORKDIR 为 `/app`。

### 3.2 以非 root 用户运行的可行方案

由于 Go 二进制为静态编译（`CGO_ENABLED=0`，见 [Dockerfile:5](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/Dockerfile#L5-L5) 和 [.goreleaser.yaml:9](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/.goreleaser.yaml#L9-L9)），不依赖系统库，可直接以任意 uid 运行。

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

镜像发布由 [.goreleaser.yaml](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/.goreleaser.yaml) 驱动，在 `builds` 节定义了 Go 交叉编译矩阵：

| goos | goarch | goarm | 说明 |
|---|---|---|---|
| linux | amd64 | - | x86_64 |
| linux | arm64 | - | ARM64 / aarch64 |
| linux | arm | 7 | ARMv7 (树莓派 2/3) |
| openbsd/freebsd/windows/darwin | 多架构 | - | 仅发布二进制 tarball/zip，不构建 Docker 镜像 |

二进制编译参数：`CGO_ENABLED=0`，ldflags `-s -w` 去除符号表并注入版本号。

### 4.2 Docker 镜像构建

`.goreleaser.yaml` 的 `dockers` 节定义了三个独立架构镜像，均使用 [Dockerfile.goreleaser](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/Dockerfile.goreleaser) 作为构建模板，使用 `buildx`：

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

发布流水线位于 [.github/workflows/release.yaml](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/.github/workflows/release.yaml)：

```
推送 v* tag → checkout → docker/login-action → setup-go → docker/setup-buildx-action → goreleaser/goreleaser-action release
```

GoReleaser 在单次运行中同时完成：
1. 多架构二进制编译与归档
2. 三个架构 Docker 镜像的 buildx 构建与推送
3. 多架构 manifest 的创建与推送
4. GitHub Release 创建与附件上传

### 4.5 本地手动构建

若不使用 GoReleaser，可直接使用项目根目录的 [Dockerfile](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/Dockerfile)：

```bash
# 构建当前平台镜像
docker build -t glanceapp/glance:local .

# 使用 buildx 多架构构建并推送
docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -t glanceapp/glance:local --push .
```

多阶段构建流程（[Dockerfile](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/Dockerfile)）：
1. **builder 阶段**：`golang:1.26.3-alpine3.22`，在 `/app` 下执行 `CGO_ENABLED=0 go build .`
2. **最终阶段**：`alpine:3.22`，仅拷贝 `/app/glance` 二进制，WORKDIR=/app，EXPOSE 8080/tcp

### 4.6 .dockerignore 约定

[.dockerignore](file:///d:/fz/0601/solo-dogfeeding/code/148-glance/.dockerignore) 使用黑名单模式：默认忽略全部，仅显式放行构建所需路径：

```
*
!/build/
!/internal/
!/pkg/
!/go.mod
!/go.sum
!main.go
```

Dockerfile 本身始终被隐式包含。这样可确保 `docs/`、`.github/` 等大体积非代码文件不进入构建上下文。

---

## 5. 快速参考

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
