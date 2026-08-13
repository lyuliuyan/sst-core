---
kind: external_dependency
name: Model Context Protocol (MCP) 服务
slug: mcp
category: external_dependency
category_hints:
    - sdk_real_api
scope:
    - '**'
source_files:
    - go.mod
    - mcp/main.go
    - mcp/cmd/server.go
---

### Model Context Protocol (MCP) 服务
- 角色：SST 提供 MCP server，使外部 AI 工具/LLM 可通过 MCP 协议访问 SST repository 的文档、diff、commit 等信息。
- 集成点：`mcp/` 目录实现 MCP server，依赖 `github.com/modelcontextprotocol/go-sdk`；`mcp/cmd/server.go` 注册 session/document 等资源。
- 稳定用法要点：MCP 是独立于 gRPC 的协议入口，与 SST core 通过 `sst` 包交互；新增资源应遵循现有 session/document 模式注册。