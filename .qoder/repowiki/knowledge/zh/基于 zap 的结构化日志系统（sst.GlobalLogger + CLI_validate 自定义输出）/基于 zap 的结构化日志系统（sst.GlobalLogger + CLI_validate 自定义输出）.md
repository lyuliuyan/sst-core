---
kind: logging_system
name: 基于 zap 的结构化日志系统（sst.GlobalLogger + CLI/validate 自定义输出）
category: logging_system
scope:
    - '**'
source_files:
    - sst/sstfile.go
    - cli/cmd/interactive/stage_handler.go
    - cli/cmd/interactive/interactive_handler.go
    - cli/cmd/utils/log.go
    - tools/validate/logger.go
    - defaultderive/defaultderive.go
    - sst/indexupdate.go
    - sst/ibnode.go
    - mcp/main.go
---

## 1. 使用的系统与框架

SST Core 的日志系统以 uber-go/zap 为核心，通过包级全局变量 GlobalLogger 在核心库中统一注入与消费。CLI、MCP 服务以及第三方依赖（如 ap242xmlimport）则使用各自独立的日志输出方式。

- 核心库 (sst 包)：go.uber.org/zap + zapcore，默认无操作 logger（zap.NewNop()），可通过 SetLogger / EnableDefaultLogger 替换。
- CLI 交互模式：在特定命令路径中临时将 sst.GlobalLogger 替换为仅输出 Error 级别的 production 配置。
- 工具集 (tools/validate)：定义自己的 LogLevel、Logger 接口和 TripleStringFormatter，不依赖 sst 的全局日志。
- 其他位置：部分代码仍使用 Go 标准库 log 或 fmt 直接输出（例如交互式会话中对 log.SetOutput 的重定向）。