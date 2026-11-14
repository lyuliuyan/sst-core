# 基于 SST Core 的企业 CRM / MES 系统数据交换与共享技术方案

## 1. 方案概述

### 1.1 背景与挑战

企业 CRM（客户关系管理）与 MES（制造执行系统）之间的数据交换面临以下核心挑战：

| 挑战维度 | 具体问题 |
|---------|---------|
| **数据模型异构** | CRM 以订单/客户/线索为核心，MES 以工单/设备/工序为核心，两者数据模型差异大 |
| **数据关联复杂** | 订单→工单→工序之间存在多层级映射关系，传统 ETL 难以维护引用完整性 |
| **版本追溯需求** | 工业场景要求数据变更可审计（谁在何时修改了什么），传统数据库缺乏原生版本控制 |
| **实时性要求** | 生产现场需要近实时的数据同步，批量定时同步存在延迟风险 |
| **安全与权限** | 跨系统数据共享需要细粒度的访问控制 |

### 1.2 方案定位

本方案利用 **SST Core**（Semantic STEP Technology Core）作为统一数据交换中间层，将 CRM 和 MES 的业务数据建模为 **RDF 语义图**，通过 SST Core 提供的：

- **统一图数据模型** — 消除异构数据模型差异
- **类 Git 版本控制** — 实现全量变更审计
- **gRPC 远程仓库 + SyncFrom 同步** — 实现近实时数据交换
- **OIDC/OAuth2 + RBAC** — 实现细粒度访问控制
- **Turtle/TriG 标准序列化** — 实现与外部系统的开放互操作

构建一套 **语义驱动、版本可控、实时同步** 的 CRM-MES 数据交换平台。

---

## 2. 整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        企业数据交换平台                               │
│                                                                     │
│  ┌──────────┐    ┌──────────────────────────────┐    ┌──────────┐  │
│  │  CRM 系统 │    │       SST Core 交换中枢       │    │  MES 系统 │  │
│  │          │    │                              │    │          │  │
│  │ ┌──────┐ │    │  ┌────────┐  ┌────────────┐  │    │ ┌──────┐ │  │
│  │ │订单   │ │    │  │ CRM    │  │ 共享图仓库  │  │    │ │工单   │ │  │
│  │ │客户   │◄├────┤  │ 仓库   │  │(SuperRepo) │  ├───►│ │设备   │ │  │
│  │ │线索   │ │    │  │        │  │            │  │    │ │工序   │ │  │
│  │ └──────┘ │    │  └────────┘  └────────────┘  │    │ └──────┘ │  │
│  │          │    │         ▲           ▲         │    │          │  │
│  │ ┌──────┐ │    │  ┌──────┴───────────┴──────┐  │    │ ┌──────┐ │  │
│  │ │映射   │◄├────┤  │      MES 仓库           │◄───┤ │映射   │ │  │
│  │ │适配器 │ │    │  │  (LocalFullRepository)  │  │    │ │适配器 │ │  │
│  │ └──────┘ │    │  └─────────────────────────┘  │    │ └──────┘ │  │
│  └──────────┘    │                              │    └──────────┘  │
│                  │  ┌─────────────────────────┐  │                  │
│                  │  │   gRPC Server + OIDC    │  │                  │
│                  │  │   Bleve 全文索引         │  │                  │
│                  │  │   MCP Server (AI 接口)   │  │                  │
│                  │  └─────────────────────────┘  │                  │
│                  └──────────────────────────────┘                  │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                    监控 / 审计 / 告警层                         │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.1 架构分层说明

| 层次 | 组件 | 职责 |
|------|------|------|
| **数据源层** | CRM 系统、MES 系统 | 各自业务数据的原始来源 |
| **适配层** | 映射适配器（Go 服务） | 将 CRM/MES 关系型数据转换为 RDF 三元组，写入 SST Stage |
| **交换中枢** | SST Core SuperRepository | 统一管理多个 Repository，提供版本控制、分支、同步能力 |
| **通信层** | gRPC + TLS + OIDC | 远程仓库间通信，保障传输安全与身份认证 |
| **服务层** | MCP Server / Bleve 索引 | 提供 AI 可消费的工具接口、全文检索能力 |
| **监控层** | Repository.Log + SyncFrom Log | 审计数据变更与同步历史 |

---

## 3. 数据建模方案

### 3.1 核心思路：RDF 统一数据模型

SST Core 以 RDF（Resource Description Framework）三元组 `<主体, 谓语, 宾语>` 为基本数据单元。CRM 和 MES 的异构数据统一表达为 RDF 图：

```
CRM 订单数据                          MES 工单数据
┌────────────────────┐               ┌────────────────────┐
│ :Order_10001       │               │ :WorkOrder_WO2001  │
│   rdf:type :Order  │               │   rdf:type :WorkOrder│
│   :orderNo "10001" │               │   :woNo "WO2001"   │
│   :customer :C_501 │   ──映射──►   │   :product :P_301  │
│   :product :P_301  │               │   :quantity 500    │
│   :quantity 500    │               │   :status "生产中"  │
│   :status "已确认"  │               │   :relatedOrder    │
│                    │               │     :Order_10001    │
└────────────────────┘               └────────────────────┘
```

### 3.2 本体设计（OWL 词汇表）

建议定义以下核心本体词汇（使用 SST 早绑定机制）：

```go
// 共享词汇表 — 定义 CRM 与 MES 的公共概念
package vocabulary

import "github.com/semanticstep/sst-core/sst"

// 公共概念
var (
    Order       = sst.IRI("http://example.com/onto#Order")
    WorkOrder   = sst.IRI("http://example.com/onto#WorkOrder")
    Customer    = sst.IRI("http://example.com/onto#Customer")
    Product     = sst.IRI("http://example.com/onto#Product")
    Equipment   = sst.IRI("http://example.com/onto#Equipment")
    Process     = sst.IRI("http://example.com/onto#Process")

    // 关系属性
    HasCustomer  = sst.IRI("http://example.com/onto#hasCustomer")
    HasProduct   = sst.IRI("http://example.com/onto#hasProduct")
    HasOrder     = sst.IRI("http://example.com/onto#hasOrder")
    HasWorkOrder = sst.IRI("http://example.com/onto#hasWorkOrder")
    HasProcess   = sst.IRI("http://example.com/onto#hasProcess")
    UsesEquipment = sst.IRI("http://example.com/onto#usesEquipment")
    RelatedOrder  = sst.IRI("http://example.com/onto#relatedOrder")

    // 数据属性
    OrderNo   = sst.IRI("http://example.com/onto#orderNo")
    WoNo      = sst.IRI("http://example.com/onto#woNo")
    Quantity  = sst.IRI("http://example.com/onto#quantity")
    Status    = sst.IRI("http://example.com/onto#status")
    StartDate = sst.IRI("http://example.com/onto#startDate")
    EndDate   = sst.IRI("http://example.com/onto#endDate")
)
```

### 3.3 NamedGraph 划分策略

利用 SST 的 NamedGraph 机制按业务域划分数据图，并通过 `owl:imports` 建立引用关系：

```
SuperRepository "enterprise-exchange"
├── Repository "crm-data"            // CRM 数据仓库
│   ├── Dataset: 客户主数据          // NamedGraph: customers
│   ├── Dataset: 订单数据            // NamedGraph: orders → imports customers
│   └── Dataset: 产品目录            // NamedGraph: products
│
├── Repository "mes-data"            // MES 数据仓库
│   ├── Dataset: 工单数据            // NamedGraph: work-orders → imports orders, products
│   ├── Dataset: 设备数据            // NamedGraph: equipment
│   └── Dataset: 工序数据            // NamedGraph: processes → imports work-orders, equipment
│
└── Repository "shared-ontology"     // 共享本体仓库
    └── Dataset: 企业本体            // NamedGraph: enterprise-onto
```

---

## 4. 核心实现方案

### 4.1 数据适配器：CRM 数据写入 SST

```go
// crm_adapter/main.go — CRM 数据适配服务
package main

import (
    "context"
    "fmt"

    "github.com/semanticstep/sst-core/sst"
    _ "github.com/semanticstep/sst-core/vocabularies/dict"
    "github.com/semanticstep/sst-core/vocabularies/rdf"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

func main() {
    // 1. 连接远程 SST 交换中枢
    transportCreds := grpc.WithTransportCredentials(insecure.NewCredentials())
    repo, err := sst.OpenRemoteRepository(
        context.TODO(),
        "sst-server:443#crm-data",  // SuperRepository 中的 CRM 仓库
        transportCreds,
    )
    if err != nil {
        panic(err)
    }
    defer repo.Close()

    // 2. 打开 Stage 工作区
    stage := repo.OpenStage(sst.DefaultTriplexMode)

    // 3. 创建或获取 NamedGraph
    ordersNG := stage.CreateNamedGraph(sst.IRI("http://example.com/graphs/orders"))

    // 4. 将 CRM 订单数据转换为 RDF 三元组写入
    order := ordersNG.CreateIRINode("Order_10001")
    order.AddStatement(rdf.Type, sst.IRI("http://example.com/onto#Order"))
    order.AddStatement(
        sst.IRI("http://example.com/onto#orderNo"),
        sst.LiteralString("10001"),
    )
    order.AddStatement(
        sst.IRI("http://example.com/onto#quantity"),
        sst.LiteralInt(500),
    )
    order.AddStatement(
        sst.IRI("http://example.com/onto#status"),
        sst.LiteralString("已确认"),
    )

    // 关联客户
    customer := ordersNG.CreateIRINode("Customer_501")
    customer.AddStatement(rdf.Type, sst.IRI("http://example.com/onto#Customer"))
    customer.AddStatement(
        sst.IRI("http://example.com/onto#customerName"),
        sst.LiteralString("某制造有限公司"),
    )
    order.AddStatement(
        sst.IRI("http://example.com/onto#hasCustomer"),
        customer,
    )

    // 5. 提交到仓库（带版本控制）
    commitHash, _, err := stage.Commit(context.TODO(), "CRM同步: 更新订单10001状态为已确认", "master")
    if err != nil {
        panic(err)
    }
    fmt.Printf("提交成功: %s\n", commitHash.String())
}
```

### 4.2 数据适配器：MES 数据写入 SST

```go
// mes_adapter/main.go — MES 数据适配服务
package main

import (
    "context"

    "github.com/semanticstep/sst-core/sst"
    _ "github.com/semanticstep/sst-core/vocabularies/dict"
    "github.com/semanticstep/sst-core/vocabularies/rdf"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

func main() {
    transportCreds := grpc.WithTransportCredentials(insecure.NewCredentials())
    repo, err := sst.OpenRemoteRepository(
        context.TODO(),
        "sst-server:443#mes-data",
        transportCreds,
    )
    if err != nil {
        panic(err)
    }
    defer repo.Close()

    stage := repo.OpenStage(sst.DefaultTriplexMode)
    woNG := stage.CreateNamedGraph(sst.IRI("http://example.com/graphs/work-orders"))

    // MES 工单 — 通过 IRI 引用 CRM 订单（跨仓库引用）
    wo := woNG.CreateIRINode("WO_2001")
    wo.AddStatement(rdf.Type, sst.IRI("http://example.com/onto#WorkOrder"))
    wo.AddStatement(
        sst.IRI("http://example.com/onto#woNo"),
        sst.LiteralString("WO2001"),
    )
    wo.AddStatement(
        sst.IRI("http://example.com/onto#quantity"),
        sst.LiteralInt(500),
    )
    // 关键：通过 IRI 直接引用 CRM 订单节点，建立跨系统数据关联
    wo.AddStatement(
        sst.IRI("http://example.com/onto#relatedOrder"),
        sst.IRI("http://example.com/data/Order_10001"),
    )

    _, _, err = stage.Commit(context.TODO(), "MES同步: 创建工单WO2001关联订单10001", "master")
    if err != nil {
        panic(err)
    }
}
```

### 4.3 仓库间数据同步（核心交换机制）

SST Core 内置了 `SyncFrom` 机制，支持仓库间的增量数据同步，这是 CRM-MES 数据交换的核心：

```go
// sync_service/main.go — 数据交换同步服务
package main

import (
    "context"
    "log"
    "time"

    "github.com/semanticstep/sst-core/sst"
    "github.com/semanticstep/sst-core/sstauth"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

func main() {
    transportCreds := grpc.WithTransportCredentials(insecure.NewCredentials())

    // 打开 SuperRepository
    sr, err := sst.OpenRemoteSuperRepository(context.TODO(), "sst-server:443", transportCreds)
    if err != nil {
        log.Fatal(err)
    }

    ctx := sstauth.ContextWithAuthProvider(context.TODO(), /* OIDC provider */ nil)

    // 获取 CRM 和 MES 仓库
    crmRepo, _ := sr.Get(ctx, "crm-data")
    mesRepo, _ := sr.Get(ctx, "mes-data")

    // 方案 A：MES 仓库同步 CRM 数据（拉取模式）
    // 将 CRM 仓库的 master 分支全量同步到 MES 仓库
    err = mesRepo.SyncFrom(ctx, crmRepo, sst.WithBranch("master"))
    if err != nil {
        log.Printf("CRM→MES 同步失败: %v", err)
    }

    // 方案 B：选择性同步 — 仅同步特定数据集
    err = mesRepo.SyncFrom(ctx, crmRepo,
        sst.WithBranch("master"),
        sst.WithDatasetIRIs("urn:uuid:订单数据集UUID"),
    )
    if err != nil {
        log.Printf("选择性同步失败: %v", err)
    }

    // 方案 C：定时轮询同步（生产环境建议用事件驱动替代）
    ticker := time.NewTicker(30 * time.Second)
    for range ticker.C {
        if err := mesRepo.SyncFrom(ctx, crmRepo); err != nil {
            log.Printf("定时同步失败: %v", err)
        }
    }
}
```

**SyncFrom 同步原理：**

```
源仓库 (crm-data)                    目标仓库 (mes-data)
┌─────────────────┐                 ┌─────────────────┐
│ SyncTo gRPC 流   │ ──── gRPC ────► │ SyncFrom gRPC 流 │
│                 │                 │                 │
│ 1. 导出 Bucket  │   双向流式传输   │ 1. 接收 Bucket  │
│    数据          │ ◄─────────────► │    数据          │
│                 │                 │                 │
│ 2. 统计同步量   │   SyncComplete   │ 2. 写入本地     │
│    - NG Revisions│ ──────────────► │    BBolt        │
│    - DS Revisions│                 │                 │
│    - Commits    │                 │ 3. 更新 Bleve   │
│    - Documents  │                 │    索引          │
└─────────────────┘                 └─────────────────┘
```

同步内容包括：
- **NamedGraph 修订** — 图数据的增量版本
- **Dataset 修订** — 数据集版本指针
- **Commit 记录** — 完整的提交历史
- **Document 文件** — 关联的文档附件

### 4.4 数据查询与导出

#### 4.4.1 Bleve 全文检索

```go
// 在 MES 仓库中搜索包含特定关键词的工单
index := mesRepo.Bleve()
if index != nil {
    req := bleve.NewSearchRequest(bleve.NewMatchQuery("WO2001"))
    req.Fields = []string{"*"}
    result, err := index.Search(req)
    // result.Hits 包含匹配的文档
}
```

#### 4.4.2 Turtle/TriG 标准格式导出

```go
// 将 Stage 中的数据导出为 TriG 格式（W3C 标准）
var buf bytes.Buffer
err := stage.RdfWrite(&buf, sst.RdfFormatTriG)
// buf 中包含标准 TriG 格式数据，可被任何 RDF 工具消费

// 或导出单个 NamedGraph 为 Turtle 格式
var ttlBuf bytes.Buffer
err := namedGraph.RdfWrite(&ttlBuf, sst.RdfFormatTurtle)
```

#### 4.4.3 图遍历查询

```go
// 遍历工单关联的所有数据
woNode := namedGraph.IRINode("WO_2001")
err := woNode.ForAllTriples(func(pred sst.IBNode, obj sst.Term) error {
    fmt.Printf("  %s -> %v\n", pred.Fragment(), obj)
    return nil
})
```

### 4.5 版本控制与审计

SST Core 提供类 Git 的版本控制能力，每次 Commit 都生成不可篡改的版本记录：

```go
// 查看 CRM 仓库的提交历史
logEntries, err := crmRepo.Log(ctx, nil, nil)
for _, entry := range logEntries {
    fmt.Printf("[%s] %s: %s\n",
        entry.Timestamp.Format(time.RFC3339),
        entry.Author,
        entry.Message,
    )
}

// 查看特定数据集的分支与提交
dataset, _ := crmRepo.Dataset(ctx, ordersDatasetIRI)
branches, _ := dataset.Branches(ctx)
for name, hash := range branches {
    fmt.Printf("分支 %s -> %s\n", name, hash.String())
}

// 查看某次提交的详细信息
details, _ := dataset.CommitDetailsByBranch(ctx, "master")
fmt.Printf("提交者: %s, 时间: %s, 修改的图: %d\n",
    details.Author, details.Timestamp, len(details.NamedGraphRevisions))
```

**审计追踪示例：**

```
[2026-08-28T10:30:00Z] crm-adapter@enterprise: CRM同步: 创建订单10001
[2026-08-28T10:30:05Z] mes-adapter@enterprise: MES同步: 创建工单WO2001关联订单10001
[2026-08-28T10:31:00Z] sync-service@enterprise: sync from repository (affected: 3 datasets)
[2026-08-28T11:00:00Z] crm-adapter@enterprise: CRM同步: 更新订单10001状态为已确认
```

---

## 5. 部署架构

### 5.1 推荐部署拓扑

```
                    ┌──────────────────────┐
                    │   SST Server (gRPC)  │
                    │   :443 TLS           │
                    │                      │
                    │  SuperRepository     │
                    │  ├─ crm-data         │
                    │  ├─ mes-data         │
                    │  └─ shared-ontology  │
                    │                      │
                    │  OIDC Provider       │
                    │  (Keycloak/自定义)    │
                    └──────┬───────────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
     ┌────────▼──┐  ┌─────▼─────┐  ┌──▼────────┐
     │ CRM 适配器 │  │ 同步服务   │  │ MES 适配器 │
     │ (Go 微服务)│  │ (Go 服务) │  │ (Go 微服务)│
     └────────┬──┘  └───────────┘  └──┬────────┘
              │                       │
     ┌────────▼──┐            ┌───────▼───┐
     │ CRM 数据库 │            │ MES 数据库 │
     │ (MySQL等) │            │ (Oracle等) │
     └───────────┘            └───────────┘
```

### 5.2 仓库实现选型

| 场景 | 推荐仓库实现 | 理由 |
|------|------------|------|
| **中央交换服务器** | LocalFullRepository | 完整版本控制 + Bleve 检索 + 事务保证 |
| **本地轻量缓存** | LocalBasicRepository | 无需版本历史，仅做数据缓存 |
| **离线文件交换** | LocalFlatRepository | 每个 Dataset 一个 .sst 文件，可直接拷贝传输 |
| **客户端应用** | RemoteRepository | 通过 gRPC 连接中央服务器 |

### 5.3 SuperRepository 服务端配置

```go
// 启动中央交换服务器
config := sst.SuperRepositoryServerConfig{
    Addr:       ":443",
    RepoDir:    "/data/sst-enterprise",
    TLSCert:    "/certs/server.crt",
    TLSKey:     "/certs/server.key",
    Issuer:     "https://keycloak.example.com/realms/enterprise",
    ClientID:   "sst-exchange",
}

srv, err := sst.NewSuperRepositoryServer(config)
if err != nil {
    log.Fatal(err)
}
log.Fatal(srv.Start())
```

---

## 6. 高级特性应用

### 6.1 分支策略实现数据隔离

利用 SST 的分支机制实现不同环境/租户的数据隔离：

```
master (已验证的生产数据)
  │
  ├── staging (预发布/测试数据)
  │     └── CRM 测试环境同步
  │
  └── dev (开发调试数据)
        └── 开发环境同步
```

```go
// MES 仅同步 master 分支的已验证数据
err := mesRepo.SyncFrom(ctx, crmRepo, sst.WithBranch("master"))

// 测试环境同步 staging 分支
err := testMesRepo.SyncFrom(ctx, testCrmRepo, sst.WithBranch("staging"))
```

### 6.2 Diff 机制实现变更检测

```go
// 比较两个版本之间的数据差异
diffResult, err := sst.SstReadDiff(reader, sst.DefaultTriplexMode)
// diffResult 包含：
//   - 新增的三元组
//   - 删除的三元组
//   - 修改的三元组
```

### 6.3 文档管理集成

SST 支持在仓库中存储关联文档（如订单附件、工艺文件等）：

```go
// 上传订单附件到仓库
file, _ := os.Open("order_10001_contract.pdf")
hash, err := repo.DocumentSet(ctx, "application/pdf", bufio.NewReader(file))

// 下载文档
var out bytes.Buffer
info, err := repo.Document(ctx, hash, &out)
fmt.Printf("文档类型: %s, 大小: %d\n", info.MIMEType, info.Size)
```

### 6.4 MCP 接口赋能 AI 应用

通过 SST MCP Server，AI 大模型可以直接操作交换平台的数据：

```
AI 助手 ←→ MCP Protocol ←→ SST MCP Server ←→ SST Core
                                    │
                                    ├── open_remote_repository  (打开仓库)
                                    ├── repository_datasets       (列出数据集)
                                    ├── dataset_checkout_branch   (检出分支)
                                    ├── stage_named_graphs        (查看图)
                                    ├── stage_trig                (导出 RDF)
                                    ├── repository_sync_from      (触发同步)
                                    └── stage_commit              (提交变更)
```

这使得 AI 助手能够：
- 查询 CRM 订单状态并关联 MES 工单进度
- 自动生成数据交换报告
- 辅助排查数据不一致问题

---

## 7. 安全与权限

### 7.1 认证架构

```
CRM 适配器 ──┐                    ┌── SST Server
             │   OIDC Token       │
MES 适配器 ──┼───────────────────►│
             │   gRPC + TLS       │  验证 Token
同步服务  ───┘                    │  检查 RBAC
                                  │
                            Keycloak
                           (OIDC Provider)
```

### 7.2 RBAC 权限矩阵

| 角色 | CRM 仓库 | MES 仓库 | 本体仓库 | 同步操作 |
|------|---------|---------|---------|---------|
| CRM 适配器 | 读写 | 只读 | 只读 | 不允许 |
| MES 适配器 | 只读 | 读写 | 只读 | 不允许 |
| 同步服务 | 读写 | 读写 | 只读 | 允许 |
| 审计员 | 只读 | 只读 | 只读 | 只读 |
| 管理员 | 读写 | 读写 | 读写 | 允许 |

---

## 8. 性能与可靠性

### 8.1 性能优化策略

| 策略 | 实现方式 |
|------|---------|
| **增量同步** | SyncFrom 仅传输新增/修改的 Bucket 数据，避免全量拷贝 |
| **选择性同步** | `WithDatasetIRIs()` 仅同步特定数据集，减少传输量 |
| **Bleve 异步索引** | 索引更新与数据写入解耦，不阻塞主流程 |
| **gRPC 流式传输** | 双向流式传输大数据集，避免内存溢出 |
| **BBolt 事务** | 本地写入使用事务保证一致性 |
| **连接复用** | RemoteRepository 复用 gRPC 连接 |

### 8.2 可靠性保障

| 机制 | 说明 |
|------|------|
| **事务保证** | LocalFullRepository 基于 BBolt 事务，确保写入原子性 |
| **幂等同步** | SyncFrom 基于 Hash 比对，重复同步不会产生重复数据 |
| **版本回退** | 支持 CheckoutCommit 回到任意历史版本 |
| **同步日志** | 每次 SyncFrom 自动记录同步日志，包含影响的数据集列表 |
| **Pre-Commit 条件** | 支持注册 `preCommitCondition` 钩子，在提交前执行校验 |

---

## 9. 实施路线图

### 阶段一：基础搭建（2-3 周）

- [ ] 部署 SST Server（SuperRepository + gRPC + TLS）
- [ ] 部署 Keycloak OIDC Provider
- [ ] 定义企业本体词汇表（OWL/RDF）
- [ ] 创建 SuperRepository 及子仓库（crm-data、mes-data、shared-ontology）

### 阶段二：适配器开发（3-4 周）

- [ ] 开发 CRM 数据适配器（读取 CRM DB → 写入 SST RDF）
- [ ] 开发 MES 数据适配器（读取 MES DB → 写入 SST RDF）
- [ ] 实现基本的 Commit/Branch 工作流

### 阶段三：同步服务（2-3 周）

- [ ] 开发数据同步服务（基于 SyncFrom API）
- [ ] 配置选择性同步规则
- [ ] 实现同步监控与告警

### 阶段四：查询与展示（2-3 周）

- [ ] 配置 Bleve 全文索引
- [ ] 开发数据查询 API（REST/GraphQL 封装）
- [ ] 集成 MCP Server 赋能 AI 应用

### 阶段五：运维与优化（持续）

- [ ] 性能调优（索引策略、同步频率）
- [ ] 完善审计日志
- [ ] 灾备方案（仓库备份与恢复）

---

## 10. 方案优势总结

| 维度 | 传统 ETL/ESB 方案 | 基于 SST Core 的方案 |
|------|-------------------|---------------------|
| **数据模型** | 需要为每对系统开发映射规则 | 统一 RDF 图模型，天然支持跨域关联 |
| **版本控制** | 无原生支持，需额外开发 | 内置类 Git 版本控制，每次变更可追溯 |
| **数据同步** | 批量定时 / 消息队列 | gRPC 流式增量同步 + 选择性数据集同步 |
| **变更审计** | 依赖数据库审计表 | 原生 Commit Log + SyncFrom Log |
| **标准兼容** | 私有格式 | W3C RDF/Turtle/TriG 开放标准 |
| **全文检索** | 需额外集成搜索引擎 | 内置 Bleve 全文索引 |
| **AI 集成** | 需定制开发 | 内置 MCP Server，AI 可直接操作 |
| **安全** | 需定制开发 | 内置 OIDC/OAuth2 + RBAC |
| **数据关联** | 外键/中间表 | RDF IRI 直接引用，跨图跨仓库自然关联 |

---

## 附录 A：关键 API 速查

| API | 用途 | 所在包 |
|-----|------|--------|
| `sst.OpenRemoteRepository()` | 打开远程仓库 | `sst` |
| `sst.NewLocalSuperRepository()` | 创建本地 SuperRepository | `sst` |
| `repo.OpenStage()` | 打开 Stage 工作区 | `sst.Repository` |
| `stage.CreateNamedGraph()` | 创建命名图 | `sst.Stage` |
| `ng.CreateIRINode()` | 创建 IRI 节点 | `sst.NamedGraph` |
| `node.AddStatement()` | 添加 RDF 三元组 | `sst.IBNode` |
| `stage.Commit()` | 提交变更到仓库 | `sst.Stage` |
| `repo.SyncFrom()` | 从另一个仓库同步数据 | `sst.Repository` |
| `dataset.CheckoutBranch()` | 检出分支到 Stage | `sst.Dataset` |
| `repo.Bleve()` | 获取全文索引 | `sst.Repository` |
| `ng.RdfWrite()` | 导出为 Turtle/TriG | `sst.NamedGraph` |
| `sst.RdfRead()` | 从 Turtle/TriG 导入 | `sst` |
| `repo.DocumentSet()` | 上传文档 | `sst.Repository` |
| `repo.Log()` | 查看仓库日志 | `sst.Repository` |

## 附录 B：依赖项

```
github.com/semanticstep/sst-core     // SST Core 核心库
github.com/semanticstep/sst-core/sstauth  // 认证模块
google.golang.org/grpc               // gRPC 通信
go.etcd.io/bbolt                     // 嵌入式 KV 存储
github.com/blevesearch/bleve/v2      // 全文检索
github.com/modelcontextprotocol/go-sdk // MCP SDK
```
