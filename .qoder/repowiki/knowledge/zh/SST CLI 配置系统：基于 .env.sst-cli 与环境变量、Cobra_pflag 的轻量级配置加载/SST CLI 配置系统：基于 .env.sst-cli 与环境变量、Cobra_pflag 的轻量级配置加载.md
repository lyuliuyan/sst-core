---
kind: configuration_system
name: SST CLI 配置系统：基于 .env.sst-cli 与环境变量、Cobra/pflag 的轻量级配置加载
category: configuration_system
scope:
    - '**'
source_files:
    - cli/cmd/utils/envsstcli.go
    - cli/cmd/utils/gettoken.go
    - cli/cmd/interactive/config.go
    - cli/cmd/root.go
    - cli/main.go
    - mcp/main.go
    - examples/helloworldreporemote/main.go
    - examples/helloworldsuperreporemote/main.go
    - tools/validate/run.go
---

## 1. 使用的系统与框架

SST Core 没有引入通用的配置库（如 Viper），而是采用**纯 Go + 标准库 + Cobra/pflag** 的轻量实现，集中在 `cli/cmd/utils` 与 `mcp` 两个入口中。

- **配置文件格式**：自定义 `.env.sst-cli` 文件，每行 `KEY=VALUE`，支持 `#` 注释、空行、单/双引号包裹的值。
- **加载方式**：启动时调用 `utils.LoadEnvSstCLI()`，按固定顺序搜索并合并到进程环境。
- **运行时参数**：CLI 通过 Cobra 命令树组织；示例程序使用 `spf13/pflag` 的 `FlagSet` 暴露命令行标志。
- **认证配置**：OIDC Keycloak 相关凭据全部来自环境变量，由 `gettoken.go` 在需要时读取。

## 2. 关键文件与包

| 文件 | 职责 |
|---|---|
| `cli/cmd/utils/envsstcli.go` | 定义 `.env.sst-cli` 文件格式、解析逻辑与搜索路径 |
| `cli/cmd/utils/gettoken.go` | 从环境变量加载 OIDC 配置、获取/刷新 JWT、构造 gRPC 认证上下文 |
| `cli/cmd/root.go` | Cobra 根命令入口，默认进入交互式模式 |
| `cli/cmd/interactive/config.go` | 交互式会话的内存状态（仓库别名、阶段、命名图等）——这是“会话级配置”而非持久化配置 |
| `cli/main.go` | 程序入口，最先调用 `LoadEnvSstCLI()` |
| `mcp/main.go` | MCP 服务器入口，同样先调用 `LoadEnvSstCLI()` |
| `examples/helloworldreporemote/main.go`、`examples/helloworldsuperreporemote/main.go` | 演示用 `pflag.FlagSet` 暴露 `--issuer`、`--client-id` 等标志 |
| `tools/validate/run.go` | 工具子程序也使用 `spf13/pflag` 解析标志 |

## 3. 架构与约定

### 3.1 配置文件搜索顺序与覆盖规则

`LoadEnvSstCLI()` 按以下顺序查找 `.env.sst-cli`：
1. 当前工作目录 (`cwd/.env.sst-cli`)
2. 用户主目录 (`$HOME/.env.sst-cli`)

找到第一个可读文件后停止。解析时遵循三条硬性约定：
- **不覆盖已有环境变量**：仅当 `os.Getenv(key) == ""` 时才写入，因此进程环境中已存在的变量优先级最高。
- **忽略注释与空行**：以 `#` 开头或空白行被跳过。
- **值去引号**：首尾同时为 `"` 或 `'` 时去除引号。

### 3.2 环境变量命名约定

所有可配置项统一使用 `SST_*` 前缀的大写环境变量：

| 变量 | 必填 | 用途 | 来源 |
|---|---:|---|---|
| `SST_OIDC_REALM_URL` | 是 | Keycloak Realm 基础 URL（末尾 `/` 会被 trim） | `.env.sst-cli` / 进程环境 |
| `SST_OIDC_CLIENT_ID` | 是 | OIDC 客户端 ID | `.env.sst-cli` / 进程环境 |
| `SST_OIDC_CLIENT_SECRET` | 是 | OIDC 客户端密钥 | `.env.sst-cli` / 进程环境 |
| `SST_USERNAME` | 否 | 用户名（与密码一起提供非交互登录） | `.env.sst-cli` / 进程环境 |
| `SST_PASSWORD` | 否 | 密码（需与 `SST_USERNAME` 同时设置） | `.env.sst-cli` / 进程环境 |
| `GOTRACEBACK` | 否 | 控制 SST 核心 panic 恢复时的堆栈输出级别 | 进程环境 |
| `SST_TEST_REPO_PATH` | 否 | 集成测试指定远程仓库路径 | 测试环境 |

缺少必需 OIDC 变量时，`loadOIDCConfig()` 会返回错误消息，明确提示应将其设置在 `.env.sst-cli`（仓库根目录或 `$HOME`）。

### 3.3 交互式会话配置

`InteractiveConfig` 结构体维护交互式模式的**进程内会话状态**，包括仓库/超级仓库/数据集/阶段/命名图/IBNode 的别名映射、分支、提交哈希、源文件路径以及每个仓库的认证上下文。该结构体以包级单例 `interactiveConfig` 持有，随交互式会话生命周期存在，不属于持久化配置。

### 3.4 命令行标志

- CLI 主程序使用 Cobra，但根命令未绑定任何全局标志；默认行为是直接启动交互式模式。
- 示例程序（`helloworldreporemote`、`helloworldsuperreporemote`）和 `tools/validate` 使用 `spf13/pflag` 的 `FlagSet` 暴露 `--issuer`、`--client-id`、`--in`、`--out` 等标志，并通过 `flag.Parse()` 解析。
- 这些标志与 `.env.sst-cli` 中的 OIDC 配置并存，用于本地示例程序的快速切换。

### 3.5 认证上下文传播

`sstauth` 包负责将 OIDC 令牌注入 gRPC context；CLI 侧通过 `GetAuthContext(repo, authContexts)` 根据仓库选择对应的认证上下文，若不存在则回退到 `context.TODO()`，使本地仓库无需认证即可访问。

## 4. 约定与约束

- **配置文件位置**：`.env.sst-cli` 必须放在当前工作目录或用户主目录，文件名固定不可配置。
- **环境变量优先**：进程环境中的同名变量始终覆盖 `.env.sst-cli` 中的值，禁止覆盖是硬编码约束（`if os.Getenv(key) == ""`）。
- **OIDC 三件套必填**：`SST_OIDC_REALM_URL`、`SST_OIDC_CLIENT_ID`、`SST_OIDC_CLIENT_SECRET` 三个变量缺一不可，否则 `loadOIDCConfig()` 直接报错。
- **Keycloak 协议固定**：Token 端点路径硬编码为 `{realmURL}/protocol/openid-connect/token`，不支持自定义 endpoint。
- **JWT 过期检测**：通过解码 JWT payload 中的 `exp` 字段判断 access token 是否即将过期（提前 10 秒触发刷新）。
- **无 YAML/TOML/JSON 配置**：整个仓库未发现任何配置文件格式的加载器；所有外部配置均通过环境变量注入。
- **无集中式配置中心**：没有 etcd、Consul、Nacos 等远程配置服务的使用痕迹。
- **测试专用配置**：集成测试通过 `SST_TEST_REPO_PATH` 环境变量指定被测仓库路径，避免硬编码路径。

## 5. 适用性说明

本仓库的配置系统属于**轻量级、面向 CLI 的本地配置方案**，适用于单机开发者与 CI 场景。它不具备多环境分层（dev/staging/prod）、热重载、配置校验 schema 或加密存储能力；如需企业级配置管理，应在部署层（容器编排、配置中心）注入环境变量，再由本系统读取。