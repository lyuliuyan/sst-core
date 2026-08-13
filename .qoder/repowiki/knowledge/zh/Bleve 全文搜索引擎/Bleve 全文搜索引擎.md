---
kind: external_dependency
name: Bleve 全文搜索引擎
slug: bleve
category: external_dependency
category_hints:
    - framework_behavior
scope:
    - '**'
source_files:
    - go.mod
    - bleveproto/index.proto
    - bleveproto/remote.go
---

### Bleve 全文搜索引擎
- 角色：为 SST Repository 提供基于 NamedGraph 内容的倒排索引与搜索能力，通过独立的 gRPC 服务暴露（`IndexService/Search`）。
- 集成点：`bleveproto/` 定义了 `index.proto` 及对应的 gRPC stub；`sst` 包在本地仓库中维护 Bleve index，并通过 `bleveproto/remote.go` 以远程方式访问。
- 稳定用法要点：搜索走独立的 gRPC 接口，不是直接调用 Bleve Go SDK；服务端需启动 `IndexService`，客户端通过 `OpenRemoteRepository` 后使用同一连接发起搜索请求。具体方法签名与参数以 `bleveproto/index.proto` 为准。
- 依赖来源：`go.mod` 声明 `github.com/blevesearch/bleve/v2` 及其 index API、scorch/zapx 等间接依赖。