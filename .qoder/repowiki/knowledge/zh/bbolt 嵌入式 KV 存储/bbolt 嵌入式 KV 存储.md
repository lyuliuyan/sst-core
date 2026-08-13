---
kind: external_dependency
name: bbolt 嵌入式 KV 存储
slug: bbolt
category: external_dependency
category_hints:
    - vendor_identity
scope:
    - '**'
source_files:
    - go.mod
    - sst/bbolt_structure.go
    - sst_test/bbolt_test.go
---

### bbolt 嵌入式 KV 存储
- 角色：作为 SST 本地仓库的底层键值存储引擎，用于持久化 commit、ref、stage 等元数据。
- 集成点：`sst/bbolt_structure.go` 定义 bbolt bucket 结构；测试文件 `sst_test/bbolt_test.go`、`sst_test/syncfrom_bbolt_test.go` 验证其版本兼容性与同步行为。
- 稳定用法要点：SST 自定义了 bucket 布局与版本号，升级 bbolt 时需关注 bucket schema 兼容性（见 `bbolt_version_update_test.go`）。