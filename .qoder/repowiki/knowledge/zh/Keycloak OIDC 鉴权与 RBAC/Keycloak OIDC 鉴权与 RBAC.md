---
kind: external_dependency
name: Keycloak OIDC 鉴权与 RBAC
slug: keycloak-oidc
category: external_dependency
category_hints:
    - auth_protocol
    - client_constraint
scope:
    - '**'
source_files:
    - sstauth/ACCESS_RIGHTS.md
    - go.mod
---

### Keycloak OIDC 鉴权与 RBAC
- 角色：SST 服务器端通过 OIDC ID token 完成认证，并依据 Keycloak client roles 映射到内部 `AccessMode`（none/read-only/read-write/admin/super-admin），实现 gRPC 方法级 RBAC。
- 集成点：`sstauth` 包中的 `UnaryRBACInterceptor`、`StreamRBACInterceptor` 从 gRPC metadata 提取 bearer token，校验签名、过期时间、audience，再解析 `resource_access.<clientID>.roles` 计算权限；客户端通过 `sstauth.ContextWithAuthProvider` 注入 token provider。
- 稳定用法要点：
  - 单仓库模式 (`PerRepoAuth=false`)：token 的 `aud` 必须等于服务器配置的 `ClientID`。
  - SuperRepository 模式 (`PerRepoAuth=true`)：普通方法要求 `aud = <superClientID>#<repoName>`，`RepoManagerService` 要求 super client 的 audience。
  - streaming RPC 的第一条消息到达前无法确定目标 repo，因此 handler 需在读取首条消息后调用 `CheckRepoAccess` 做精确校验。
  - 支持 Keycloak service account + Client Credentials grant 进行机器间访问。
- 依赖来源：`go.mod` 引入 `coreos/go-oidc/v3`、`golang.org/x/oauth2`。