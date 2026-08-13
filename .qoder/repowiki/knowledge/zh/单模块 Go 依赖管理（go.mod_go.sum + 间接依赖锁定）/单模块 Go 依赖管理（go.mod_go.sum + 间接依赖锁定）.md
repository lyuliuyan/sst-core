---
kind: dependency_management
name: 单模块 Go 依赖管理（go.mod/go.sum + 间接依赖锁定）
category: dependency_management
scope:
    - '**'
source_files:
    - go.mod
    - go.sum
---

## 1. 使用的系统与工具

- **Go Modules**：整个仓库是单一 Go module，module path 为 `github.com/semanticstep/sst-core`，Go 版本要求为 `go 1.25.0`。
- **依赖声明与锁定**：所有直接依赖集中在根目录的 `go.mod` 中，通过 `require` 块显式声明；间接依赖由 `go mod tidy` 在 `go.sum` 中锁定校验和。
- **无 vendor 目录**：仓库未使用 `vendor/` 进行依赖本地化，构建时从远端或 GOPROXY 拉取。
- **无私有代理配置**：未发现 `.golangci.yaml`、`.gitignore` 或其他文件中对 `GOPRIVATE`、`GONOSUMDB`、`GONOPROXY` 的设置，也未发现自定义 go proxy 配置。

## 2. 关键文件

- `go.mod`：唯一依赖清单，定义 module、Go 版本以及全部直接依赖（含 gRPC、Bleve、BBolt、Cobra、OIDC、MCP SDK、测试库等）。
- `go.sum`：所有依赖（直接+间接）的哈希锁，保证可重复构建。
- `.golangci.yaml`：仅用于 lint 配置，不声明依赖。

## 3. 架构与约定

- **单模块多包组织**：`sst`、`cli`、`mcp`、`bboltproto`、`bleveproto`、`sstauth`、`step`、`tools`、`vocabularies`、`examples`、`defaultderive`、`singleton`、`sst_test` 均为同一 module 下的子包，彼此通过相对 import path 引用，不存在跨 module 依赖。
- **依赖分层清晰**：
  - 核心库依赖：`go.etcd.io/bbolt`（持久化）、`github.com/blevesearch/bleve/v2`（全文索引）、`google.golang.org/grpc` + `google.golang.org/protobuf`（远程通信）、`github.com/coreos/go-oidc/v3` + `golang.org/x/oauth2`（认证）。
  - CLI/MCP 层额外引入 `spf13/cobra`、`chzyer/readline`、`modelcontextprotocol/go-sdk` 等。
  - 测试相关依赖（如 `stretchr/testify`、`google/go-cmp`）集中放在顶层 require 中，供 `sst_test` 等测试包复用。
- **间接依赖显式列出**：`go.mod` 中大量第三方依赖以 `// indirect` 标记（如 bleve 生态、RoaringBitmap、golang.org/x/*），说明项目倾向于将常用间接依赖也提升到顶层，便于统一升级与审计。
- **协议代码自生成**：`bboltproto`、`bleveproto` 中的 `.pb.go`、`_grpc.pb.go`、`repository.pb.fm.go` 等由 protoc / xtoproto / yeqown/protoc-gen-fieldmask 生成，属于“被依赖”的代码而非依赖来源。

## 4. 约定与约束

- **所有第三方依赖必须通过 `go get` 加入并写入 `go.mod`/`go.sum`**：仓库没有 vendoring、没有 `replace` 指令、没有私有模块路径，任何新增依赖都应走标准 Go Modules 流程。
- **Go 版本锁定**：`go.mod` 首行固定 `go 1.25.0`，意味着所有依赖必须兼容该版本，升级 Go 版本需同步评估依赖兼容性。
- **无 replace 或私有替换**：未发现 `replace` 语句，所有依赖均指向公开版本（包括 `semanticstep/sst-ontologies v1.0.1` 等内部语义包也以公开 tag 形式引用）。
- **依赖更新策略**：当前未见自动化脚本或 CI 步骤强制更新依赖；实践中应通过 `go get -u ./...` 配合 `go mod tidy` 维护，并检查 `go.sum` 变更。
- **测试依赖与生产依赖混用**：`stretchr/testify`、`google/go-cmp` 等测试库出现在顶层 require 中，而非仅在 `sst_test` 的独立 module 下，这是单模块仓库的常见做法，但需注意避免误引入到生产二进制。

综上，该项目采用最简化的 Go Modules 方案：单一 `go.mod` + `go.sum` 锁定全部依赖，无 vendor、无私有代理、无 replace，所有子包共享同一依赖图。