# sst-core 系统架构图

> 模块路径：`github.com/semanticstep/sst-core` ｜ Go 1.25 ｜ 单 Go Module 多包结构
> 本文档基于仓库当前代码结构（`sst/`、`cli/`、`mcp/`、`bboltproto/`、`bleveproto/`、`sstauth/`、`step/`、`tools/`、`vocabularies/`、`defaultderive/`）绘制。

---

## 1. 全景分层架构

```mermaid
graph TB
    subgraph L6["⑥ 接入层 / Entry Points"]
        CLI["SST CLI<br/>cli/main.go + cli/cmd<br/>Cobra + readline 交互会话"]
        MCP["MCP Server<br/>mcp/main.go<br/>25 个工具，stdio 协议"]
        APP["业务应用<br/>examples/* · CircuitCAM · CoCollaWorking"]
        SRV["仓库服务端进程<br/>sst.NewServer / sst.NewSuperServer"]
    end

    subgraph L5["⑤ 认证与授权横切层"]
        AUTH["sstauth<br/>OIDC/OAuth2 校验 · AccessMode · RBAC 拦截器<br/>Unary/StreamRBACInterceptor · Principal 透传"]
    end

    subgraph L4["④ 协议与传输层"]
        BPROTO["bboltproto<br/>repository.proto → gRPC 存根"]
        LPROTO["bleveproto<br/>index.proto → 远程检索存根"]
        SSTF["SST-1.0 二进制格式<br/>sstfile*.go · sstwriter.go · sstreaddiff.go"]
    end

    subgraph L3["③ 仓库抽象层 Repository API"]
        REPI["Repository 接口<br/>OpenStage · Dataset(s) · CommitDetails<br/>CheckoutCommit · RegisterIndexHandler · Bleve · Info"]
        SUPERI["SuperRepository 接口<br/>Get/Create/Delete/List · Quota 体系"]
        IMPL["四种实现<br/>LocalFull · LocalBasic · LocalFlat(FileSystem) · Remote"]
    end

    subgraph L2["② 工作区与图模型层"]
        STAGE["Stage 暂存区<br/>Create/Commit/MergeAndMerge"]
        DS["Dataset · NamedGraph<br/>三元组容器与引用图"]
        IB["IBNode / Term 模型<br/>IRI · BlankNode · Literal · Collection · TermCollection"]
        RDF["RDF 读写管线<br/>Turtle/TriG/N-Triples 词法+语法解析"]
    end

    subgraph L1["① 持久化与索引层"]
        SFS["StageFS<br/>wrfs 虚拟文件系统，UUID 为路径"]
        BBOLT["BBolt KV<br/>ngr / dsr / ds / c / dl 桶"]
        BLEVE["Bleve 全文索引<br/>defaultderive 派生映射 · 异步批量更新"]
        VAULT["Document Vault<br/>哈希分片的二进制文档保险库"]
        FS["扁平文件系统<br/>base64url-IRI.sst"]
    end

    subgraph L0["⓪ 本体与数据源层"]
        VOCAB["vocabularies/*<br/>rdf rdfs xsd owl sso lci dict sh skos rep ..."]
        STEP["step/p21 · ap242xml<br/>STEP Part21 / AP242 XML / SVG 转换"]
        ONT["sst-ontologies (外部依赖)<br/>Turtle 本体源"]
    end

    CLI --> APP
    MCP --> APP
    APP --> REPI
    APP --> SUPERI
    SRV --> REPI
    SRV -.鉴权拦截.-> AUTH
    APP -.携带 Token 的 ctx.-> AUTH

    REPI --> IMPL
    SUPERI --> IMPL
    IMPL --> SFS
    IMPL --> SSTF
    IMPL --> BBOLT
    IMPL --> BLEVE
    IMPL --> VAULT
    IMPL --> FS

    SFS --> STAGE
    STAGE --> DS
    DS --> IB
    RDF --> STAGE
    STEP --> RDF
    VOCAB --> DS
    ONT --> VOCAB

    IMPL -.远程调用.-> BPROTO
    BLEVE -.远程检索.-> LPROTO
```

---

## 2. 核心对象模型（内存数据结构）

```mermaid
classDiagram
    class Repository {
        <<interface>>
        +URL() string
        +SuperRepository() SuperRepository
        +OpenStage(TriplexMode) Stage
        +ForDatasets(fn) error
        +ForDocuments(fn) error
        +Datasets() []IRI
        +Dataset(IRI) Dataset
        +CommitDetails([]Hash)
        +CheckoutCommit(Hash, TriplexMode) Stage
        +RegisterIndexHandler(SSTDeriveInfo) error
        +Bleve() bleve.Index
        +RebuildBleveIndex() error
        +Info(branch) RepositoryInfo
        +Close() error
    }
    class SuperRepository {
        <<interface>>
        +Get/Create/Delete/List()
        +GetQuota/SetQuota()
        +GetTotalQuota/SetTotalQuota()
        +GetMaxRepositoryCount()
    }
    class Stage {
        <<interface>>
        +CreateNamedGraph(IRI) NamedGraph
        +NamedGraph(IRI) NamedGraph
        +NamedGraphs() slice
        +ReferencedGraph(IRI) NamedGraph
        +ForNamedGraphs(fn) error
        +Repository() Repository
        +Commit(ctx, msg, branch) Hash
        +MoveAndMerge(ctx, from) Report
        +IsValid() bool
    }
    class Dataset {
        <<interface>>
        +IRI() IRI
        +Branches(ctx) map
        +LeafCommits(ctx) slice
        +CommitDetailsByHash(ctx, hash)
        +CommitDetailsByBranch(ctx, branch)
        +SetBranchCommit(ctx, hash, branch) error
        +RemoveBranch(ctx, branch) error
        +CheckoutBranch(ctx, branch, mode) Stage
        +CheckoutCommit(ctx, hash, mode) Stage
        +CheckoutRevision(ctx, rev, mode) Stage
        +CommitForRevision(ctx, rev) Hash
        +FindCommonParentRevision(ctx, r1, r2) Hash
        +Repository() Repository
    }
    class NamedGraph {
        <<interface>>
        +IRI() IRI
        +ID() uuid
        +IsReferenced() bool
        +IsModified() bool
        +CreateIRINode(fragment, types) IBNode
        +CreateBlankNode(types) IBNode
        +CreateCollection(members) TermCollection
        +ForAllIBNodes(fn) error
        +GetSortedIBNodes() slice
        +AddImport(ng) error
        +DirectImports() slice
        +Diff(other) DiffTriples
        +RdfWrite(w, format) error
        +SstWrite(w) error
        +Stage() Stage
    }
    class Term模型 {
        <<IRI / BlankNode / Literal / Collection>>
    }
    class IBNode {
        <<三元组索引节点>>
    }

    SuperRepository "1" o-- "n" Repository : 聚合多个仓库条目
    Repository "1" --> "n" Stage : OpenStage 创建
    Repository "1" o-- "n" Dataset : 持久化
    Stage "1" o-- "n" NamedGraph : 本地 + 引用图
    Dataset "1" o-- "n" NamedGraph : 默认图 + 导入图
    NamedGraph "1" o-- "n" IBNode
    IBNode "1" o-- "3..*" Term : subject/predicate/object
```

**关键语义约束**

| 概念 | 说明 |
|---|---|
| Dataset 标识 | 由 Dataset IRI 派生的 UUIDv5，仓库内唯一 |
| NamedGraph 状态 | `local`（本 Stage 创建/修改）vs `referenced`（仅被引用未加载） |
| TriplexMode | 三元组/四元组模式，决定 NamedGraph 是否被隐式划分为 sub-graph |
| 版本链 | NamedGraphRevision → DatasetRevision → Commit，三者以 Hash 串联 |

---

## 3. 存储层架构与能力矩阵

```mermaid
graph LR
    subgraph API["Repository 接口（唯一耦合点）"]
        A["上层应用仅依赖接口<br/>实现可透明替换"]
    end

    subgraph LOCAL["本地实现"]
        LF["LocalFullRepository<br/>repolocalfull.go"]
        LB["LocalBasicRepository<br/>repolocalbasic.go"]
        LFL["LocalFlatRepository<br/>repolocalflat(.filesystem).go"]
    end

    subgraph REMOTE["远程实现"]
        RR["RemoteRepository<br/>reporemote*.go"]
        SRR["SuperRemoteRepository<br/>superremoterepository.go"]
    end

    subgraph STORE["物理存储"]
        B1[("BBolt<br/>ngr dsr ds c dl")]
        B2[("Bleve Index<br/>+ healthAwareIndex")]
        B3[("目录 + .sst 文件")]
        B4[("Document Vault")]
    end

    subgraph NET["远端服务"]
        GS["gRPC Server<br/>Repository Server / Super Server"]
    end

    A --> LF
    A --> LB
    A --> LFL
    A --> RR
    A --> SRR
    LF --> B1
    LF --> B2
    LF --> B4
    LB --> B1
    LB --> B2
    LFL --> B3
    RR --> GS
    SRR --> GS
    GS --> B1
    GS --> B2
    GS --> B4
```

| 能力 | LocalFull | LocalBasic | LocalFlat | Remote |
|---|---|---|---|---|
| 底层存储 | BBolt + Bleve | BBolt + Bleve | 文件系统 `.sst` | 服务端 BBolt + Bleve |
| 提交/版本历史 | ✅ | ❌ | ❌ | ✅ |
| Branch / CheckoutCommit | ✅ | ❌ | ❌ | ✅ |
| Diff / 增量同步 | ✅ | 部分 | ✅（文件哈希） | ✅ |
| Bleve 全文检索 | ✅ | ✅ | ❌ `ErrNotAvailable` | ✅（远程客户端） |
| `RegisterIndexHandler` | ✅ | ✅ | ❌ | ❌（部署期决定） |
| 事务 / 并发控制 | `stateControl` | `stateControl` | ❌ | 服务端保证 |
| 认证授权 | n/a | n/a | n/a | OIDC + RBAC |
| 工厂方法 | `OpenLocalRepository`（自动识别） | `OpenLocalRepository`（自动识别） | `CreateLocalFlatRepository` / `OpenLocalFlatRepository` | `OpenRemoteRepository(ctx, URL, tlsOption)`；Super 变体见 `sst/superremoterepository.go` |

> 创建入口统一为 `sst.CreateLocalRepository(repoDir, email, name, revisionHistory bool)`：`revisionHistory=true` 生成 LocalFull，`false` 生成 LocalBasic；`OpenLocalRepository` 则通过检测 bbolt 中是否存在 basic 专用根桶（`rootRepositoryBucketName`，字面值 `repository`，见 `sst/repolocalbasic.go`）自动选择实现类。

### BBolt 桶结构（`sst/bbolt_structure.go`）

```mermaid
graph TD
    ROOT[("StorageDB")]
    ROOT --> NGR["ngr : NamedGraphRevisions<br/>key = NG-SST 二进制哈希<br/>value = 独立 NG-SST 文件体"]
    ROOT --> DSR["dsr : DatasetRevisions<br/>key = Dataset 修订哈希"]
    ROOT --> DS["ds : Datasets<br/>key = DS-UUID"]
    ROOT --> C["c : Commits<br/>value = 提交对象 + ref 前缀"]
    ROOT --> DL["dl : log<br/>仓库级操作日志序列"]
    DSR --> D0["key 0x00 → 默认 NG-Revision-Hash"]
    DSR --> D1["key 0x00 + DS-UUID → 导入的 Dataset 引用"]
    C --> CR["refBranchPrefix：branch / tag / leaf 指向修订"]
    DS --> C2["DS-Revision-Hash → 回链 dsr"]
```

---

## 4. 远程部署与 gRPC 通信架构

```mermaid
graph TB
    subgraph CLIENT["客户端进程"]
        CAPP["应用 / CLI / MCP"]
        CR["RemoteRepository<br/>reporemote.go"]
        CSR["SuperRemoteRepository"]
        CT["OAuth2 TokenSource<br/>client_credentials / refresh"]
    end

    subgraph TRANSPORT["传输"]
        CH["gRPC Channel<br/>TLS 可选 · grpc-middleware auth"]
    end

    subgraph SERVER["服务端进程"]
        INT["UnaryRBACInterceptor / StreamRBACInterceptor"]
        OIDCV["OIDC IDTokenVerifier<br/>Keycloak Claims → AccessMode"]
        RS["RepositoryServer<br/>repositoryService · commitService · documentService"]
        SS["SuperRepositoryServer<br/>repoManagerService"]
        IDX["indexServiceServer<br/>Bleve 检索"]
    end

    subgraph BACKEND["后端存储"]
        BB[("BBolt")]
        BL[("Bleve")]
        VL[("Document Vault")]
    end

    subgraph IDP["身份提供方"]
        KC["Keycloak / OIDC Issuer"]
    end

    CAPP --> CR
    CAPP --> CSR
    CR --> CT
    CSR --> CT
    CT -.发现/验签.-> KC
    CR --> CH
    CSR --> CH
    CH --> INT
    INT -.拉取 JWKS.-> KC
    INT --> RS
    INT --> SS
    INT --> IDX
    SS --> RS
    RS --> BB
    RS --> VL
    IDX --> BL
    RS --> BL
```

**协议分层**

| Proto | 服务域 | 生成产物 |
|---|---|---|
| `bboltproto/repository.proto` | 仓库管理、Dataset/Ref/Commit、文档流式传输、双向同步 | `repository.pb.go`、`repository_grpc.pb.go`、`repository.pb.fm.go`（fieldmask） |
| `bleveproto/index.proto` | 远程索引搜索 | `index.pb.go`、`index_grpc.pb.go` |

代码生成命令由 `//go:generate` 驱动：`protoc` + `protoc-gen-go` + `protoc-gen-go-grpc` + `protoc-gen-fieldmask`。

---

## 5. 数据流：导入 → 编辑 → 提交 → 检索

```mermaid
sequenceDiagram
    autonumber
    participant SRC as 数据源<br/>Turtle/TriG · P21 · AP242 XML · SVG
    participant PARSER as RDF 词法/语法解析<br/>rdflex · rdftrig · rdfreaderstage
    participant STAGE as Stage（内存）
    participant NG as NamedGraph / IBNode
    participant ENC as SST 二进制编码<br/>sstwriter · sstfile
    participant FS as StageFS（wrfs）
    participant DB as BBolt
    participant IDX as BleveIndexUpdater（异步）

    SRC->>PARSER: 流式读入
    PARSER->>STAGE: 逐条 rdfTriple → Stage
    STAGE->>NG: 创建/合并 IRI·BlankNode·Literal 节点
    Note over STAGE,NG: 应用调用 vocabularies 生成的类型化 setter<br/>（编译期推理结果固化）
    STAGE->>ENC: Commit(ctx, msg, branch)
    ENC->>FS: 以 UUID 为路径写入 NG-SST
    FS->>DB: 生成 NamedGraphRevision → DatasetRevision → Commit
    DB-->>IDX: 提交事件（batch precondition）
    IDX->>IDX: defaultderive 派生索引文档，批量写入
    IDX-->>STAGE: 检索可用（Search / SearchInContext）
```

**读取方向**：`Repository.Dataset / CheckoutCommit` → BBolt 取修订 → SST 二进制解码 → Stage/NamedGraph → RDF 写回器（`rdfwrite.go`）→ Turtle/TriG 文本。

---

## 6. 本体管线：编译期推理 + 运行期判定

```mermaid
graph LR
    subgraph ONTSRC["本体源"]
        T["sst-ontologies<br/>*.ttl Turtle"]
    end
    subgraph COMPILE["编译期 · tools/compiler"]
        R["reasoner.go<br/>子类型图 + Floyd-Warshall 传递闭包"]
        G["代码生成<br/>AsKind_* · Element/ElementInformer"]
    end
    subgraph GEN["生成产物"]
        V["vocabularies/&lt;name&gt;/vocabulary.go<br/>dictionary.go · dict/vocabularymap.go"]
    end
    subgraph RUNTIME["运行期"]
        INF["ElementInformer<br/>IsClass / SubtypeOf / Domain / Range / InverseOf"]
        VAL["tools/validate<br/>SHACL：rdf_type · domain_range · functional_property"]
    end

    T --> R --> G --> V
    V --> INF
    V --> VAL
    INF --> APPRUN["O(1) 类型归属判定"]
    VAL --> APPRUN
```

> SST Core 不做通用运行时推理：类层次推导在编译期完成并固化进 Go 代码；运行期只做轻量判定与约束校验。

---

## 7. 包依赖关系

```mermaid
graph TD
    SSTCORE["sst（核心库）"]
    AUTH["sstauth"]
    BBP["bboltproto"]
    BLP["bleveproto"]
    DD["defaultderive"]
    VOC["vocabularies/*"]
    STEP["step/*"]
    CLI["cli"]
    MCPS["mcp"]
    TOOLS["tools/*"]
    TESTS["sst_test"]
    SINGLE["singleton"]

    BBP --> SSTCORE
    BLP --> SSTCORE
    SSTCORE --> AUTH
    DD --> SSTCORE
    VOC --> SSTCORE
    STEP --> SSTCORE
    STEP --> VOC
    CLI --> SSTCORE
    CLI --> VOC
    CLI --> STEP
    CLI --> DD
    MCPS --> SSTCORE
    MCPS --> AUTH
    TOOLS --> SSTCORE
    TOOLS --> VOC
    SINGLE --> VOC
    TESTS --> SSTCORE
    TESTS --> CLI
```

依赖原则：**所有子包通过 Go import 协作，不存在进程间通信**；`sst` 是唯一向外暴露语义能力的核心，其余包均单向依赖它（`bboltproto`/`bleveproto` 被 `sst` 侧的远程实现消费）。

---

## 8. 周边工具与转换层

```mermaid
graph TB
    subgraph FORMATS["外部格式"]
        P21["STEP Part 21"]
        A242I["AP242 XML 导入"]
        A242E["AP242 XML 导出"]
        SVG["SVG 2D 几何"]
        TTL["Turtle / TriG"]
    end

    subgraph STEPPKG["step/*"]
        LEX["p21 词法/语法<br/>golex + goyacc 生成"]
        D2S["detailedToSimplified"]
        S2D["simplifiedToDetailed"]
    end

    subgraph TOOLSPKG["tools/*"]
        COMP["compiler<br/>本体 → Go 词汇"]
        VALID["validate<br/>SHACL / SST 规则校验"]
        FEX["filterextraction<br/>子图过滤抽取"]
        S2SVG["sst_to_svg"]
        SVG2S["svg_to_sst"]
        HASH["b32tob58 · b64tob32<br/>hextob58 · bboltngr"]
    end

    P21 --> LEX --> D2S
    A242I --> S2D
    D2S --> COREGRAPH["SST NamedGraph"]
    S2D --> COREGRAPH
    LEX --> COREGRAPH
    A242I --> COREGRAPH
    TTL <--> COREGRAPH
    COREGRAPH --> A242E
    COREGRAPH --> S2SVG --> SVG
    SVG --> SVG2S --> COREGRAPH
    COMP --> VOCABPKG["vocabularies/*"]
    COREGRAPH --> VALID
    COREGRAPH --> FEX
    HASH --> COREGRAPH
```

---

## 9. 接入层能力对照

| 入口 | 形态 | 主要能力 | 配置来源 |
|---|---|---|---|
| **CLI**（`cli/`，可执行 `sst`） | Cobra 根命令 + readline 交互会话 | `openlocalrepository` / `createlocalrepository` / `openlocalflatrepository` / `openremoterepository` / `open{local,remote}superrepository`；`datasets` `dataset` `openstage` `log` `commitinfo` `commitdiff` `checkoutcommit`；`rdfread` `sstread` `importp21` `importap242xml` `importsvg`；`document{set,get,info,delete}` `documents`；`syncfrom` `clone` `filterextract` `extractsstfile` `dump` `query` `listfield` `superrepository`；`utils` 提供 `.env.sst-cli` 加载、分页、彩色输出、bbolt dump、token 获取 | `.env.sst-cli` + 环境变量 + pflag；版本经 `-ldflags -X` 注入 |
| **MCP Server**（`mcp/`） | Model Context Protocol，stdio | 25 个工具：`status`、`open_local_repository`、`open_remote_repository`、`repository_{close,sync_from,datasets,dataset,open_stage}`、`dataset_{branches,list_commits,commit_details_by_hash,commit_details_by_branch,history,checkout_branch,checkout_commit}`、`stage_{info,named_graphs,validate,trig,commit}`、`repository_document_{set,get,info,delete}` | 远程仓库依赖 `.env.sst-cli` 中的 `SST_OIDC_*` 与 `SST_USERNAME`/`SST_PASSWORD` |
| **库嵌入**（`examples/*`） | Go package import | 直接调用 `sst` API：本地 basic/flat/full 仓库、远程仓库、SuperRepository、P21、PMI 分析 | 代码内联 / 命令行参数 |
| **服务端**（`sst.NewServer` / `sst.NewSuperServer`） | gRPC 进程 | 托管多仓库、配额、索引重建、文档流 | `RepositoryServerConfig`（TLS、issuer、client-id） |

---

## 10. 横切关注点

```mermaid
graph LR
    subgraph CROSS["横切能力"]
        LOG["日志：go.uber.org/zap<br/>sst.GlobalLogger；CLI/validate 自定义输出"]
        ERR["错误：哨兵 error + panic 恢复<br/>errrecovery.go · CLI 人类可读化"]
        QUOTA["配额：RepositoryQuota<br/>单仓库 / 总量 / 仓库数量上限"]
        HEALTH["索引健康：healthAwareIndex<br/>降级与重建 RebuildBleveIndex"]
        PLATFORM["跨平台：runtime.GOOS 分支<br/>不依赖 CGO"]
    end
    CROSS -.作用于.-> SSTCOREPKG["sst / cli / mcp / tools"]
```

---

## 11. 架构特征小结

| 特征 | 体现 |
|---|---|
| **接口单一、实现可插拔** | `Repository`/`SuperRepository` 是唯一耦合点，本地 BBolt、扁平文件、远程 gRPC 三者对应用透明等价 |
| **Stage 作为唯一写入边界** | 所有修改先入内存 Stage，`Commit` 才产生不可变修订；未链接仓库的 Stage 可作纯内存/临时暂存 |
| **内容寻址存储** | NamedGraph/Dataset 修订以 SHA-256 哈希（Base58 编码）为键，天然去重、可 diff、可增量同步 |
| **图即文件系统** | `StageFS` 用 `wrfs` 接口把 NamedGraph 映射为以 UUID 为路径的虚拟文件，统一 BBolt 与文件后端 |
| **推理前移** | 本体推理在编译期完成并固化为 Go 代码，运行期零推理开销，只做 O(1) 判定与 SHACL 校验 |
| **搜索可选可重建** | Bleve 索引通过 `defaultderive` 派生，异步批量更新并带健康度监控，不可用时不影响图读写 |
| **认证集中在传输层** | OIDC/RBAC 以 gRPC 拦截器注入，业务实现（`sst` 包内仓库逻辑）不感知身份细节 |
| **构建极简** | 无 Makefile/Dockerfile/CI，全靠 `go build` + `go generate`（protoc / golex / goyacc / 本体编译器） |

---

## 附：源码定位速查

| 关注点 | 文件 |
|---|---|
| Repository 接口与远程工厂 | `sst/repository.go` |
| SuperRepository 接口与本地实现 | `sst/superlocalrepository.go` |
| 本地 full / basic / flat 仓库 | `sst/repolocalfull.go` · `sst/repolocalbasic.go` · `sst/repolocalflat.go`、`sst/repolocalflatfilesystem.go` |
| 远程仓库客户端 | `sst/reporemote.go`、`sst/reporemoteds.go`、`sst/reporemoteref.go`、`sst/reporemotecommit.go` |
| 仓库服务端 | `sst/reporemoteapp.go`、`sst/superremoterepository.go` |
| Stage / Dataset / NamedGraph | `sst/stage.go` · `sst/dataset.go` · `sst/namedgraph.go` |
| 提交与差异 | `sst/commit.go` · `sst/sstwritediff.go` · `sst/sstreaddiff.go` · `sst/diff.go` |
| BBolt 桶结构 | `sst/bbolt_structure.go` |
| StageFS 虚拟文件系统 | `sst/stagefs.go`、`sst/stagefsdatasetsfs.go`、`sst/stagefslocal*.go` |
| SST 二进制格式 | `sst/sstfile.go`、`sst/sstfilestagefs.go`、`sst/sstfiledeltaspanread.go` |
| RDF 解析与序列化 | `sst/rdflex.go`、`sst/rdfttl.go`、`sst/rdftrig.go`、`sst/rdfreader*.go`、`sst/rdfwrite*.go` |
| Bleve 索引更新 | `sst/indexupdate.go`、`defaultderive/` |
| 文档保险库 | `sst/vault.go`、`sst/DocumentInfo.go` |
| 认证授权 | `sstauth/auth.go`、`sstauth/authoids.go`、`sstauth/accessmode.go` |
| gRPC 协议 | `bboltproto/repository.proto`、`bleveproto/index.proto` |
| 本体编译器 / 校验器 | `tools/compiler/reasoner.go`、`tools/validate/validate.go` |
| STEP 转换 | `step/p21/`、`step/ap242xml{import,export}/`、`step/{detailed,simplified}To*/` |
