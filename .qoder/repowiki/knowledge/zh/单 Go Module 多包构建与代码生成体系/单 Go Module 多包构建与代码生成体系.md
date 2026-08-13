---
kind: build_system
name: 单 Go Module 多包构建与代码生成体系
category: build_system
scope:
    - '**'
source_files:
    - go.mod
    - cli/cmd/version.go
    - .golangci.yaml
    - bboltproto/remote.go
    - bleveproto/remote.go
    - step/p21/p21.go
    - tools/compiler/README.md
    - cli/main.go
    - mcp/main.go
---

## 1. 构建系统概览

该项目采用 **单一 Go module（`github.com/semanticstep/sst-core`）** 组织所有子包，使用 `go build` / `go run` / `go test` 作为核心构建、运行与测试命令。仓库中不存在 Makefile、Dockerfile、CI 流水线或 release 脚本；版本注入通过 `go build -ldflags -X` 完成。跨平台编译依赖 Go 原生交叉编译能力（代码中多处显式检查 `runtime.GOOS == "windows"`）。

## 2. 关键文件与工具

- `go.mod`：声明模块路径与全部依赖，Go 版本锁定为 `go 1.25.0`。
- `cli/cmd/version.go`：定义 `version`、`buildTime`、`commit` 三个变量，由 `-ldflags -X` 注入，默认值为 `dev`。
- `.golangci.yaml`：集中配置 golangci-lint v2 规则，启用 errorlint、exhaustive、forbidigo、godot、godox、revive、unconvert、unparam，并强制使用 gofumpt 格式化。
- `bboltproto/remote.go`、`bleveproto/remote.go`：通过 `//go:generate` 调用 `protoc` + `protoc-gen-go-grpc` + `protoc-gen-fieldmask` 生成 gRPC 存根。
- `step/p21/p21.go`：通过 `//go:generate golex` / `goyacc` 从 `.l` / `.y` / `.ebnf` 源生成词法分析器与语法分析器。
- `tools/compiler/README.md`：文档化本体编译器，将 Turtle 本体编译为 `vocabularies/` 下的 Go 词汇包。

## 3. 架构与约定

### 3.1 多入口可执行文件
每个顶层目录中的 `main.go` 构成独立可执行程序：
- `cli/main.go` → SST CLI（基于 Cobra）
- `mcp/main.go` → MCP 服务器
- `tools/compiler`、`tools/sst_to_svg`、`tools/svg_to_sst`、`tools/validate`、`tools/bboltngr`、`tools/hextob58`、`tools/b32tob58`、`tools/b64tob32` → 辅助工具
- `examples/*` → 示例程序

构建方式统一为 `go build -o <目标> ./<包路径>`，CLI 文档也明确给出 `go build -o cli/sst ./cli/main.go` 的用法。

### 3.2 版本注入约定
`cli/cmd/version.go` 注释中给出了标准注入模板：
```bash
go build -ldflags "-X 'github.com/semanticstep/sst-core/cli/cmd.version=$(git describe --tags --abbrev=0)' -X 'github.com/semanticstep/sst-core/cli/cmd.buildTime=$(date -u +%Y%m%d%H%M%S)' -X 'github.com/semanticstep/sst-core/cli/cmd.commit=$(git rev-parse --short=12 HEAD)'" -o cli/sst ./cli
```
未注入时 `version` 默认为 `dev`，`versionString()` 会拼接 `version-buildTime-commit` 三段。

### 3.3 代码生成管线
- **gRPC 协议**：`bboltproto/repository.proto` 与 `bleveproto/index.proto` 是源码，`*.pb.go`、`*_grpc.pb.go`、`repository.pb.fm.go` 均为生成产物，提交到仓库并由 `//go:generate` 重新生成。
- **STEP 解析器**：`step/p21/iso10303p21y2016.l`、`.y`、`.ebnf` 经 `golex` / `goyacc` 生成 `iso10303p21y2016lexer.go`、`iso10303p21y2016parser.go`。
- **词汇表编译**：`tools/compiler` 读取 `sst-ontologies` 的 Turtle 本体，生成 `vocabularies/<name>/vocabulary.go`、`dictionary.go` 以及 `vocabularies/dict/vocabularymap.go` 与预序列化 `.sst` 字典。

### 3.4 静态分析与质量门禁
`.golangci.yaml` 强制：
- 禁止在库代码中使用 `fmt.Printf` 与标准 `log.Print*`（仅允许在 `cli/`、`tools/`、`examples/`、`sst_test/`、`step/`、`singleton/` 等路径下使用）。
- 对 `_test.go` 放宽 forbidigo 限制。
- 忽略生成的第三方与 builtin 路径。
- 使用 gofumpt 进行唯一格式输出。

## 4. 约定与约束

- **单一模块**：所有子包共享 `go.mod`，不存在子模块拆分。
- **无外部构建编排**：没有 Makefile、Dockerfile、GitHub Actions 或其他 CI 配置文件；构建完全依赖 `go` 命令行。
- **版本来源**：版本号来自 git tag，构建时间来自 `date -u`，提交哈希来自 `git rev-parse --short=12 HEAD`，三者均通过 ldflags 注入。
- **跨平台兼容**：核心库通过 `runtime.GOOS` 分支处理 Windows 差异（如路径分隔符），不依赖 CGO。
- **代码生成即产物**：`.proto`、`.l`、`.y`、`.ebnf` 等源码与生成的 `.go` 文件一同提交，开发者需先安装 protoc/golex/goyacc 再执行 `go generate`。
- **依赖锁定**：所有依赖（含间接依赖）均在 `go.sum` 中锁定，确保可重现构建。

## 5. 适用性说明

该仓库属于纯 Go 项目，构建系统简单直接——以 `go build` 为核心，辅以 `go generate` 驱动的多阶段代码生成。由于缺少 Makefile/Dockerfile/CI 脚本，本卡片聚焦于实际存在的构建约定而非缺失的外部流程。