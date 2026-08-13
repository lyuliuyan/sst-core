---
kind: error_handling
name: SST Core 错误处理体系：哨兵错误、panic 恢复与 CLI 人类可读化
category: error_handling
scope:
    - '**'
source_files:
    - sst/repository.go
    - sst/reporemote.go
    - sst/bbolt_structure.go
    - sst/errrecovery.go
    - cli/cmd/utils/clierror.go
    - sst/hash.go
    - sst/rdfwrite.go
    - sst/sstwriter.go
    - sst/sstwritediff.go
    - sst/sstwritestage.go
    - sst/vocabulary.go
    - step/p21/p21.go
---

## 1. 整体方案

SST Core 采用 **Go 标准库 `errors` + gRPC status + 自定义 panic 恢复** 的混合策略：
- 核心库（`sst` 包）通过包级 `var ErrXxx = errors.New(...)` 定义**哨兵错误**，调用方使用 `errors.Is` 进行精确分支判断。
- 远程 gRPC 层将业务语义映射为 `google.golang.org/grpc/codes`，并在客户端链中向上返回。
- CLI 层集中将底层错误翻译为用户可读的英文短句，统一输出格式 `Cannot <op>: <reason>.`。
- 对可能 panic 的远程/异步操作使用 `handleWithRecovery` 捕获并转为 `recoveredError`，由上层决定是否附带堆栈。

## 2. 关键文件与位置

| 文件 | 职责 |
|---|---|
| `sst/repository.go` | 仓库级哨兵错误集中声明：`ErrRepositoryNotFound`、`ErrDatasetNotFound`、`ErrDatasetAlreadyExists`、`ErrBranchNotFound`、`ErrEmptyCommitMessage`、`ErrQuotaExceeded`、`ErrNotSupported`、`ErrUnrecognizedOption` 等 |
| `sst/reporemote.go` | 远程仓库状态错误：`ErrRepoClosed`、`ErrStageExpired`、`ErrRepoNotFound`；以及 `wrapError(file, line, err)` 在错误上附加源码行号以便调试 |
| `sst/bbolt_structure.go` | 存储层哨兵错误：`ErrConcurrentModification`、`ErrDatasetRevisionNotFound`、`ErrNamedGraphRevisionNotFound`、`ErrUnsupportedRepository`、`ErrUnknownRefType` |
| `sst/hash.go` | `ErrIllegalHashLength` |
| `sst/rdfwrite.go` | `ErrWriterClosed` |
| `sst/sstwriter.go` | `ErrMissingImports` |
| `sst/sstwritediff.go` | `ErrGraphIRIsMismatch` |
| `sst/sstwritestage.go` | `ErrDirectoryExpectedAsBasePath` |
| `sst/vocabulary.go` | `ErrInvalidDictionaryStage`、`ErrElementInformerNotFound` |
| `sst/errrecovery.go` | `recoveredError` 类型与 `handleWithRecovery(ctx, ch, handler)` 工具，基于 `GOTRACEBACK` 环境变量控制是否包含堆栈 |
| `cli/cmd/utils/clierror.go` | CLI 错误解释器：`FormatCLIProblem` / `ExplainCLIError` / `PrintCLIProblem`，按优先级从 sentinel → gRPC chain → known messages → network heuristics → humanize 提取原因 |
| `cli/cmd/interactive/stage_handler.go` | 示例：用 `errors.Is(writeErr, sst.ErrDirectoryExpectedAsBasePath)` 做分支处理 |

## 3. 架构与约定

### 3.1 哨兵错误（Sentinel Errors）
- 所有跨包可被 `errors.Is` 匹配的错误均以包级 `var ErrXxx = errors.New("...")` 形式声明，集中在 `sst/repository.go`、`sst/reporemote.go`、`sst/bbolt_structure.go` 等文件中。
- 调用方一律通过 `errors.Is(err, sst.ErrXxx)` 判断，而不是字符串比较。CLI 层的 `reasonFromSentinel` 是这种模式的唯一集中消费点，把哨兵错误翻译成用户可见的短句。

### 3.2 gRPC 错误传播
- 远程调用失败时，gRPC `status` 的 `codes` 被映射到人类可读原因：`PermissionDenied` → "write access required"、`Unauthenticated` → "authentication failed"、`NotFound` 根据消息内容进一步区分 branch/dataset/document/commit/repository 等。
- `reasonFromGRPCChain` 遍历 `errors.Unwrap` 链查找第一个 gRPC status，确保即使错误被多层包装也能识别。

### 3.3 错误包装与调试信息
- `wrapError(filepath.Base(file), line, err)` 在错误前追加 `filename:line:` 前缀，便于定位来源；CLI 的 `cleanErrorMessage` 会剥离该前缀再显示给用户。
- 内部逻辑也大量使用 `fmt.Errorf("...", err)` 或 `errors.Wrap` 风格包装，保留原始错误链供 `errors.Is` 和 gRPC 解析。

### 3.4 Panic 恢复
- `handleWithRecovery` 将传入的 `handler(ctx) error` 包裹进 `recover()`，若发生 panic 则构造 `recoveredError` 并通过 channel 返回。
- `recoveredError` 支持 `IncludingTraceback()` / `ExcludingTraceback()` 切换是否在 `Error()` 中包含 `runtime.Stack` 输出的堆栈。
- 是否收集堆栈由 `GOTRACEBACK` 环境变量决定（none/single/all/system/crash），默认单帧。

### 3.5 CLI 错误呈现规范
- 统一入口 `FormatCLIProblem(operation, reason)` 生成 `Cannot {operation}: {reason}.` 的单行消息。
- `ExplainCLIError` 负责从任意错误树中提取最合适的 reason，优先顺序：sentinel → gRPC chain → known message 子串匹配 → 网络错误启发式 → 原始消息 humanize。
- 内部错误（包含 `.go:`、`rpc error`、`failed to resolve grpc` 等模式）会被过滤为通用 "operation failed"，避免向用户泄露实现细节。

## 4. 约定与约束

1. **核心库对外暴露的错误必须是包级哨兵变量**，禁止仅靠字符串匹配；新增业务错误需在 `sst/repository.go` 或对应子模块顶部集中声明。
2. **下游只允许使用 `errors.Is` 判断业务错误**，禁止直接比较 `err.Error()` 字符串（CLI 层除外，用于最终展示）。CLI 层新增错误分支需同步更新 `reasonFromSentinel` 与 `reasonFromKnownMessages`。
3. **远程 gRPC 错误必须保持 `codes` 语义**，客户端不得吞掉 status code；CLI 层依赖这些 code 做分类。
4. **任何可能 panic 的远程/异步调用必须经 `handleWithRecovery` 包裹**，并将结果以 error 形式返回给调用者。
5. **CLI 不直接打印底层错误**，必须通过 `PrintCLIProblem` / `ExplainCLIError` 输出，保证用户看到一致格式的英文短句。
6. **内部调试信息（文件名+行号）通过 `wrapError` 注入**，但必须在面向用户的消息中剥离（`cleanErrorMessage` 已处理）。
7. **堆栈输出受 `GOTRACEBACK` 控制**，生产环境默认仅单帧，可通过环境变量提升为全量堆栈。

## 5. 适用范围

该错误处理体系覆盖 SST Core 的核心库（`sst`）、CLI（`cli/cmd`）、MCP 服务器（`mcp/cmd`）、认证（`sstauth`）以及 STEP 转换（`step/p21`）等全部 Go 代码路径，是仓库统一的错误处理约定。