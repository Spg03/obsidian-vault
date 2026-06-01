---
type: implementation
status: codex-expanded
tags:
  - gitnexus
  - group
  - cross-repo
---

# Group 实现原理

Group 解决的是多仓库场景：一个系统由多个仓库组成，单仓库图谱无法回答跨服务影响面。GitNexus 的做法不是把所有仓库硬合成一个超大图，而是让每个仓库独立索引，再用契约桥接。

源码入口：

```text
gitnexus/src/cli/group.ts
gitnexus/src/core/group/sync.ts
gitnexus/src/core/group/matching.ts
gitnexus/src/core/group/bridge-db.ts
gitnexus/src/core/group/cross-impact.ts
gitnexus/src/core/group/extractors/
```

## 总览图

![[gitnexus-group-data-flow-overview.png]]

## 为什么不直接合并所有仓库图谱

直接合并看起来简单，但工程上问题很多：

- 仓库可能属于不同团队，更新节奏不同。
- 多仓库符号名容易冲突。
- 单个仓库重建不应该强迫所有仓库重建。
- 跨仓库真正稳定的边界通常不是函数名，而是服务契约。

所以 Group 采用“单仓库独立索引 + 跨仓库契约桥”的方式。

## 三个核心文件

| 文件 | 作用 |
|---|---|
| `group.yaml` | 用户定义有哪些 repo、显式 links、detect 和 matching 配置 |
| `contracts.json` | Contract Registry，记录抽取出的契约和匹配结果 |
| `bridge.lbug` | 桥接数据库，保存 Contract 节点和 ContractLink 边 |

## syncGroup 主流程

`syncGroup(config, opts)` 做四件大事：

```text
read registry
  -> resolve each repo handle
  -> open each repo LadybugDB
  -> run extractors
  -> discover workspace links
  -> process manifest links
  -> exact match
  -> wildcard match
  -> dedupe cross-links
  -> write contracts.json
  -> write bridge.lbug
```

## 契约抽取

sync 会根据配置运行多种 extractor：

- HttpRouteExtractor
- GrpcExtractor
- ThriftExtractor
- TopicExtractor
- IncludeExtractor
- Workspace dependency extractor
- ManifestExtractor

每个 extractor 输出 `StoredContract`，包含：

- repo。
- service。
- role：provider 或 consumer。
- type：http、grpc、thrift、topic 等。
- contractId。
- symbolUid / symbolRef。
- confidence。

## matching 如何工作

匹配核心在 `matching.ts`。

### normalizeContractId

先把契约 ID 规范化。例如：

- HTTP method 大写，path 去尾部斜杠。
- gRPC / Thrift service 小写化，方法名保留大小写。
- topic trim + lowercase。
- include 路径统一斜杠和小写。

### provider index

`buildProviderIndex` 会把 provider contract 按规范化 ID 建索引。这样 consumer 查找 provider 时不用全量扫描。

### exact match

`runExactMatch` 会找 contractId 完全匹配的 provider，并生成 CrossLink。

### wildcard match

`runWildcardMatch` 主要处理 gRPC / Thrift 的 service wildcard，例如：

```text
grpc::UserService/*
```

它会匹配 provider 中同 service 的 method-level contract。

## manifest link 为什么重要

自动抽取不可能覆盖所有跨仓库关系，所以 `group.yaml` 里的显式 links 很重要。`ManifestExtractor` 会把人工声明的 link 也转成 contract/crossLink。

sync 里还会优先保留 manifest cross-link，因为它代表用户意图，可信度通常高于自动匹配。

## Bridge DB 的角色

`contracts.json` 是 canonical source of truth，`bridge.lbug` 是查询加速和跨仓库影响分析用的桥接库。

如果 writeBridge 失败，sync 不会让整个 contracts.json 写入失败，而是 warning。因为下次 sync 可以恢复 bridge。

## 跨仓库 impact

跨仓库 impact 的基本思路：

```text
local impact in current repo
  -> query bridge.lbug
  -> find cross-repo consumers
  -> run impact in consumer repo
  -> merge result
```

这让 GitNexus 可以回答“我改 api-gateway 的 login handler，会不会影响 user-service 或 web-app”。

## 边界

- 自动契约抽取依赖语言和框架支持。
- Exact/wildcard 匹配比语义匹配可靠，但覆盖面有限。
- 健康检查、param-only path 等容易产生噪声，需要过滤。
- Manifest link 是重要补充，不要把 Group 讲成全自动万能跨仓库理解。

## 相关笔记

- [[Contract Extraction]]
- [[Bridge DB]]
- [[Cross Repo Impact]]
- [[边界与取舍]]
