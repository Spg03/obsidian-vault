---
type: reference
status: codex-expanded
tags:
  - gitnexus
  - schema
  - ladybugdb
---

# 图谱 Schema 速览

这篇基于 GitNexus 当前代码整理，主要参考：

```text
gitnexus/src/core/lbug/schema.ts
gitnexus-shared/src/lbug/schema-constants.ts
gitnexus-shared/src/graph/types.ts
gitnexus/src/mcp/resources.ts
```

GitNexus 的 LadybugDB schema 是“节点分表 + 统一关系表”的混合图模型：

```text
多个节点表：File / Function / Class / Method / ...
一个关系表：CodeRelation，通过 type 字段区分 CALLS / IMPORTS / DEFINES / ...
一个向量表：CodeEmbedding
```

## 一句话理解

GitNexus 不是把所有东西塞进一张大表，而是按代码实体类型建节点表，再用统一的 `CodeRelation` 表连接所有节点。这样 Cypher 可以自然表达：

```cypher
MATCH (caller)-[:CodeRelation {type: 'CALLS'}]->(callee:Function)
RETURN caller.name, callee.name
```

## Schema 常量

| 常量 | 值 | 来源 |
|---|---|---|
| `NODE_TABLES` | 31 个持久化节点表 | `gitnexus-shared/src/lbug/schema-constants.ts` |
| `REL_TABLE_NAME` | `CodeRelation` | `gitnexus-shared/src/lbug/schema-constants.ts` |
| `REL_TYPES` | 20 个关系类型 | `gitnexus-shared/src/lbug/schema-constants.ts` |
| `EMBEDDING_TABLE_NAME` | `CodeEmbedding` | `gitnexus-shared/src/lbug/schema-constants.ts` |
| `EMBEDDING_DIMS` | 默认 384，可由 `GITNEXUS_EMBEDDING_DIMS` 改 | `gitnexus/src/core/lbug/schema.ts` |
| `EMBEDDING_INDEX_NAME` | `code_embedding_idx` | `gitnexus/src/core/lbug/schema.ts` |

## 节点表总览

当前 LadybugDB 实际创建 31 个节点表：

```text
File, Folder, Function, Class, Interface, Method, CodeElement,
Community, Process, Section,
Struct, Enum, Macro, Typedef, Union, Namespace, Trait, Impl,
TypeAlias, Const, Static, Variable, Property, Record, Delegate,
Annotation, Constructor, Template, Module, Route, Tool
```

注意：`gitnexus-shared/src/graph/types.ts` 的 `NodeLabel` 还包含 `Project`、`Package`、`Decorator`、`Import`、`Type` 等更宽泛的图类型，但它们不在当前 LadybugDB `NODE_TABLES` 里，不是当前持久化节点表。

## 节点表字段明细

### File

```sql
CREATE NODE TABLE File (
  id STRING,
  name STRING,
  filePath STRING,
  content STRING,
  PRIMARY KEY (id)
)
```

用途：源码文件节点。`content` 可用于搜索、上下文展示和 Wiki 生成。

### Folder

```sql
CREATE NODE TABLE Folder (
  id STRING,
  name STRING,
  filePath STRING,
  PRIMARY KEY (id)
)
```

用途：目录节点。通常通过 `CONTAINS` 连接子目录或文件。

### Function

```sql
CREATE NODE TABLE Function (
  id STRING,
  name STRING,
  filePath STRING,
  startLine INT64,
  endLine INT64,
  isExported BOOLEAN,
  content STRING,
  description STRING,
  PRIMARY KEY (id)
)
```

用途：函数、箭头函数等可调用符号。

### Class

```sql
CREATE NODE TABLE Class (
  id STRING,
  name STRING,
  filePath STRING,
  startLine INT64,
  endLine INT64,
  isExported BOOLEAN,
  content STRING,
  description STRING,
  PRIMARY KEY (id)
)
```

用途：类定义。常通过 `HAS_METHOD`、`HAS_PROPERTY`、`EXTENDS`、`IMPLEMENTS` 连接其他节点。

### Interface

```sql
CREATE NODE TABLE Interface (
  id STRING,
  name STRING,
  filePath STRING,
  startLine INT64,
  endLine INT64,
  isExported BOOLEAN,
  content STRING,
  description STRING,
  PRIMARY KEY (id)
)
```

用途：接口或类型定义。可参与 `IMPLEMENTS`、`METHOD_IMPLEMENTS`、`HAS_METHOD` 等关系。

### Method

```sql
CREATE NODE TABLE Method (
  id STRING,
  name STRING,
  filePath STRING,
  startLine INT64,
  endLine INT64,
  isExported BOOLEAN,
  content STRING,
  description STRING,
  parameterCount INT32,
  returnType STRING,
  PRIMARY KEY (id)
)
```

用途：类、结构体、接口、trait、impl 等拥有的方法。

### CodeElement

```sql
CREATE NODE TABLE CodeElement (
  id STRING,
  name STRING,
  filePath STRING,
  startLine INT64,
  endLine INT64,
  isExported BOOLEAN,
  content STRING,
  description STRING,
  PRIMARY KEY (id)
)
```

用途：兜底型代码元素。当语言或框架结构无法归入更具体类型时使用。

### Community

```sql
CREATE NODE TABLE Community (
  id STRING,
  label STRING,
  heuristicLabel STRING,
  keywords STRING[],
  description STRING,
  enrichedBy STRING,
  cohesion DOUBLE,
  symbolCount INT32,
  PRIMARY KEY (id)
)
```

用途：Leiden 社区发现得到的功能模块。

字段含义：

| 字段 | 含义 |
|---|---|
| `label` | 原始社区标签 |
| `heuristicLabel` | 便于人读的功能区名称 |
| `keywords` | 聚类关键词 |
| `description` | 描述，可由启发式或 LLM 增强 |
| `enrichedBy` | `heuristic` 或 `llm` 等来源 |
| `cohesion` | 社区内聚度 |
| `symbolCount` | 社区内符号数量 |

### Process

```sql
CREATE NODE TABLE Process (
  id STRING,
  label STRING,
  heuristicLabel STRING,
  processType STRING,
  stepCount INT32,
  communities STRING[],
  entryPointId STRING,
  terminalId STRING,
  PRIMARY KEY (id)
)
```

用途：执行流节点。Process 不是代码本身，而是从入口点追踪出的调用链/流程。

字段含义：

| 字段 | 含义 |
|---|---|
| `heuristicLabel` | 人类可读流程名，例如 LoginFlow |
| `processType` | `intra_community` 或 `cross_community` 等 |
| `stepCount` | 流程步数 |
| `communities` | 流程涉及的社区 |
| `entryPointId` | 入口节点 ID |
| `terminalId` | 终止节点 ID |

### 多语言通用 Code Element 表

这些表都使用同一个基础模板：

```sql
CREATE NODE TABLE `<Name>` (
  id STRING,
  name STRING,
  filePath STRING,
  startLine INT64,
  endLine INT64,
  content STRING,
  description STRING,
  PRIMARY KEY (id)
)
```

覆盖的节点表如下：

| 节点表 | 常见来源 |
|---|---|
| `Struct` | C/C++/Rust/Go 等结构体 |
| `Enum` | 枚举 |
| `Macro` | 宏 |
| `Typedef` | C/C++ typedef |
| `Union` | C/C++ union |
| `Namespace` | C++/C# 等命名空间 |
| `Trait` | Rust trait |
| `Impl` | Rust impl block |
| `TypeAlias` | TS/Rust/Swift 等类型别名 |
| `Const` | 常量 |
| `Static` | 静态变量或静态项 |
| `Variable` | 变量 |
| `Record` | C# record 等 |
| `Delegate` | C# delegate |
| `Annotation` | Java annotation 等 |
| `Constructor` | 构造函数 |
| `Template` | C++ template 等 |
| `Module` | 模块 |

查询这些带特殊标识的表时，Cypher 里建议使用反引号：

```cypher
MATCH (s:`Struct`) RETURN s.name, s.filePath LIMIT 20
```

### Property

```sql
CREATE NODE TABLE `Property` (
  id STRING,
  name STRING,
  filePath STRING,
  startLine INT64,
  endLine INT64,
  content STRING,
  description STRING,
  declaredType STRING,
  PRIMARY KEY (id)
)
```

用途：字段、属性、成员变量。`declaredType` 用于字段访问链解析，例如 `user.address.city` 这类链式访问。

### Route

```sql
CREATE NODE TABLE Route (
  id STRING,
  name STRING,
  filePath STRING,
  responseKeys STRING[],
  errorKeys STRING[],
  middleware STRING[],
  PRIMARY KEY (id)
)
```

用途：HTTP API 路由，来自 Next.js、Express、PHP 等路由识别。

字段含义：

| 字段 | 含义 |
|---|---|
| `name` | 路由名称或路径标识 |
| `responseKeys` | handler 返回 JSON 中识别到的顶层字段 |
| `errorKeys` | 错误响应字段 |
| `middleware` | 保护该路由的 middleware/wrapper 链 |

### Tool

```sql
CREATE NODE TABLE Tool (
  id STRING,
  name STRING,
  filePath STRING,
  description STRING,
  PRIMARY KEY (id)
)
```

用途：MCP/RPC 工具定义节点。用于 `tool_map`、影响分析、工具实现追踪。

### Section

```sql
CREATE NODE TABLE Section (
  id STRING,
  name STRING,
  filePath STRING,
  startLine INT64,
  endLine INT64,
  level INT64,
  content STRING,
  description STRING,
  PRIMARY KEY (id)
)
```

用途：Markdown 文档标题段落。`level` 对应 Markdown heading level。

## 关系表 CodeRelation

GitNexus 只创建一个关系表：

```sql
CREATE REL TABLE CodeRelation (
  ... FROM/TO endpoint declarations ...,
  type STRING,
  confidence DOUBLE,
  reason STRING,
  step INT32
)
```

所有关系都通过 `CodeRelation.type` 区分，而不是为每种关系单独建表。

## 关系属性

| 字段 | 类型 | 含义 |
|---|---|---|
| `type` | STRING | 关系类型，例如 `CALLS`、`IMPORTS` |
| `confidence` | DOUBLE | 置信度，1.0 表示高确定性，低于 1 表示推断或弱证据 |
| `reason` | STRING | 关系来源或原因，例如 field read/write、解析策略说明 |
| `step` | INT32 | 流程步骤，主要用于 `STEP_IN_PROCESS` |

内存图里的 `GraphRelationship` 还可以带 `evidence`：

```ts
evidence?: readonly {
  readonly kind: string;
  readonly weight: number;
  readonly note?: string;
}[];
```

这主要服务 scope-resolution 产生的边，用于解释某条边为什么有这个置信度。注意当前 LadybugDB `CodeRelation` DDL 中没有单独的 `evidence` 列。

## 完整关系类型

当前 `REL_TYPES` 包含 20 个关系类型：

| 关系类型 | 含义 | 常见查询方向 |
|---|---|---|
| `CONTAINS` | Folder/File 包含子节点 | Folder -> File，Folder -> Folder，File -> Section 等 |
| `DEFINES` | File 定义符号 | File -> Function/Class/Method/... |
| `IMPORTS` | 文件或符号导入依赖 | File/Function/Method -> 目标符号或文件 |
| `CALLS` | 函数、方法、构造器等调用另一个可调用符号 | caller -> callee |
| `EXTENDS` | 继承父类、父接口、trait 等 | child -> parent |
| `IMPLEMENTS` | 类或结构实现接口/trait | concrete -> abstract |
| `HAS_METHOD` | 类型拥有方法 | Class/Struct/Interface/Trait/Impl -> Method |
| `HAS_PROPERTY` | 类型拥有属性 | Class/Struct/Interface/Trait/Impl -> Property |
| `ACCESSES` | 函数/方法读取或写入属性 | Function/Method -> Property，`reason` 可为 read/write |
| `METHOD_OVERRIDES` | 方法覆盖另一个方法 | overriding method -> overridden method |
| `OVERRIDES` | 旧版兼容别名 | legacy indexes |
| `METHOD_IMPLEMENTS` | 具体方法实现接口方法 | concrete method -> interface method |
| `MEMBER_OF` | 符号属于社区 | symbol -> Community |
| `STEP_IN_PROCESS` | 符号是某执行流的第 N 步 | symbol -> Process，`step` 表示顺序 |
| `HANDLES_ROUTE` | 文件/函数/方法处理路由 | handler -> Route |
| `FETCHES` | 前端或客户端消费 API | consumer -> Route/API-like target |
| `HANDLES_TOOL` | 文件/函数/方法处理 MCP/RPC 工具 | handler -> Tool |
| `ENTRY_POINT_OF` | 某入口属于流程入口 | entry symbol/route/tool -> Process |
| `WRAPS` | wrapper/middleware 包装关系 | wrapper -> wrapped |
| `QUERIES` | ORM 或数据查询关系 | code symbol -> queried entity/operation |

补充：`gitnexus-shared/src/graph/types.ts` 的 `RelationshipType` 还包含 `INHERITS`、`USES`、`DECORATES`。它们属于共享内存图类型口径，但不在当前 LadybugDB `REL_TYPES` 常量里；写 Cypher 时应优先参考 `REL_TYPES` 和 `CodeRelation.type`。

## CodeRelation 允许连接的节点类型

`schema.ts` 在 `CREATE REL TABLE CodeRelation` 中显式列出大量 `FROM X TO Y` 组合。它不是任意节点都能连任意节点，而是按 GitNexus 会产生的关系组合提前声明。

### 结构与定义类连接

| From | To | 典型关系 |
|---|---|---|
| `Folder` | `Folder`, `File` | `CONTAINS` |
| `File` | `File`, `Folder`, 各类符号节点, `Section`, `Route`, `Tool` | `CONTAINS`, `DEFINES`, `IMPORTS`, `HANDLES_ROUTE`, `HANDLES_TOOL` |

### 可调用符号连接

| From | To | 典型关系 |
|---|---|---|
| `Function` | `Function`, `Method`, `Class`, `Interface`, `Constructor`, `Property`, 多语言节点, `Community`, `Process` | `CALLS`, `ACCESSES`, `MEMBER_OF`, `STEP_IN_PROCESS` |
| `Method` | `Function`, `Method`, `Class`, `Interface`, `Constructor`, `Property`, 多语言节点, `Community`, `Process` | `CALLS`, `METHOD_OVERRIDES`, `METHOD_IMPLEMENTS`, `ACCESSES` |
| `Constructor` | `Constructor`, `Class`, `Struct`, `Function`, `Method`, 多语言节点, `Community`, `Process` | `CALLS`, `DEFINES`, `STEP_IN_PROCESS` |

### 类型结构连接

| From | To | 典型关系 |
|---|---|---|
| `Class` | `Method`, `Property`, `Class`, `Interface`, 多语言节点, `Community`, `Process` | `HAS_METHOD`, `HAS_PROPERTY`, `EXTENDS`, `IMPLEMENTS` |
| `Interface` | `Function`, `Method`, `Class`, `Interface`, `TypeAlias`, `Struct`, `Constructor`, `Property`, `Community`, `Process` | `HAS_METHOD`, `EXTENDS`, `METHOD_IMPLEMENTS` |
| `Struct` | `Struct`, `Trait`, `Class`, `Enum`, `Function`, `Method`, `Interface`, `Constructor`, `Property`, `Community`, `Process` | `HAS_METHOD`, `HAS_PROPERTY`, `IMPLEMENTS` |
| `Trait` | `Method`, `Constructor`, `Property`, `Community`, `Process` | `HAS_METHOD`, `HAS_PROPERTY` |
| `Impl` | `Method`, `Constructor`, `Property`, `Trait`, `Struct`, `Impl`, `Community`, `Process` | `HAS_METHOD`, `IMPLEMENTS`, `EXTENDS` |
| `Template` | `Template`, `Function`, `Method`, `Class`, `Struct`, `TypeAlias`, `Enum`, `Macro`, `Interface`, `Constructor`, `Community`, `Process` | 模板定义、调用、成员关系 |
| `Module` | `Module`, `Function`, `Method`, `Community`, `Process` | 模块包含、调用、流程步骤 |

### 其他语言节点连接

| From | To | 典型关系 |
|---|---|---|
| `Enum` | `Enum`, `Class`, `Interface`, `Community`, `Process` | 类型关系、社区归属、流程步骤 |
| `Macro` | `Function`, `Method`, `Community`, `Process` | 宏调用或宏归属 |
| `Namespace` | `Struct`, `Community`, `Process` | 命名空间包含/归属 |
| `TypeAlias` | `Trait`, `Class`, `Community`, `Process` | 类型别名关系 |
| `Typedef`, `Union`, `Const`, `Static`, `Variable`, `Property`, `Record`, `Delegate`, `Annotation` | `Community`, `Process`，部分还可连 `Method`/`Constructor`/`Property` | 归属、流程步骤、成员关系 |

### 入口节点连接

| From | To | 典型关系 |
|---|---|---|
| `File`, `Function`, `Method` | `Route` | `HANDLES_ROUTE` |
| `File`, `Function`, `Method` | `Tool` | `HANDLES_TOOL` |
| `Route`, `Tool` | `Process` | `STEP_IN_PROCESS`, `ENTRY_POINT_OF` |

### Process 连接

这些节点类型都可通过 `STEP_IN_PROCESS` 指向 `Process`：

```text
Function, Method, Class, Interface, Struct, Constructor, Module, Macro,
Impl, Typedef, TypeAlias, Enum, Union, Namespace, Trait, Const, Static,
Variable, Property, Record, Delegate, Annotation, Template, CodeElement,
Route, Tool
```

这说明 Process 是一条执行流的聚合视图，不只函数和方法能参与流程，路由、工具、类型节点也可以作为流程成员或入口。

## Embedding 表

Embedding 不放在普通节点表里，而是单独建 `CodeEmbedding`：

```sql
CREATE NODE TABLE CodeEmbedding (
  id STRING,
  nodeId STRING,
  chunkIndex INT32,
  startLine INT64,
  endLine INT64,
  embedding FLOAT[384],
  contentHash STRING,
  PRIMARY KEY (id)
)
```

字段含义：

| 字段 | 含义 |
|---|---|
| `id` | embedding chunk 的唯一 ID |
| `nodeId` | 对应代码节点 ID |
| `chunkIndex` | 一个节点被切成多个 chunk 时的序号 |
| `startLine` / `endLine` | chunk 覆盖的行号 |
| `embedding` | 向量，默认 384 维 |
| `contentHash` | 内容 hash，用于判断 embedding 是否过期 |

向量索引：

```sql
CALL CREATE_VECTOR_INDEX('CodeEmbedding', 'code_embedding_idx', 'embedding', metric := 'cosine')
```

默认模型维度来自 `snowflake-arctic-embed-xs` 的 384 维，但可通过 `GITNEXUS_EMBEDDING_DIMS` 修改。

## 常见 Cypher 查询

### 查函数调用方

```cypher
MATCH (caller)-[:CodeRelation {type: 'CALLS'}]->(f:Function {name: "validateUser"})
RETURN caller.name, caller.filePath
```

### 查函数调用了谁

```cypher
MATCH (f:Function {name: "validateUser"})-[:CodeRelation {type: 'CALLS'}]->(callee)
RETURN labels(callee)[0] AS type, callee.name, callee.filePath
```

### 查文件定义了哪些符号

```cypher
MATCH (file:File)-[:CodeRelation {type: 'DEFINES'}]->(symbol)
WHERE file.filePath CONTAINS "auth"
RETURN file.filePath, labels(symbol)[0] AS type, symbol.name
```

### 查类的方法

```cypher
MATCH (c:Class {name: "UserService"})-[:CodeRelation {type: 'HAS_METHOD'}]->(m:Method)
RETURN m.name, m.parameterCount, m.returnType
```

### 查字段访问

```cypher
MATCH (fn)-[r:CodeRelation {type: 'ACCESSES'}]->(p:Property)
WHERE p.name = "address"
RETURN labels(fn)[0] AS type, fn.name, r.reason, p.declaredType
```

### 查接口实现

```cypher
MATCH (impl)-[:CodeRelation {type: 'IMPLEMENTS'}]->(iface:Interface)
RETURN impl.name, labels(impl)[0] AS implType, iface.name
```

### 查方法重写

```cypher
MATCH (winner:Method)-[r:CodeRelation {type: 'METHOD_OVERRIDES'}]->(loser:Method)
RETURN winner.name, winner.filePath, loser.name, loser.filePath, r.reason
```

### 查社区成员

```cypher
MATCH (s)-[:CodeRelation {type: 'MEMBER_OF'}]->(c:Community)
WHERE c.heuristicLabel = "Auth"
RETURN labels(s)[0] AS type, s.name, s.filePath
```

### 追执行流步骤

```cypher
MATCH (s)-[r:CodeRelation {type: 'STEP_IN_PROCESS'}]->(p:Process)
WHERE p.heuristicLabel = "LoginFlow"
RETURN r.step, labels(s)[0] AS type, s.name, s.filePath
ORDER BY r.step
```

### 查 API 路由返回字段

```cypher
MATCH (handler)-[:CodeRelation {type: 'HANDLES_ROUTE'}]->(route:Route)
RETURN route.name, route.filePath, route.responseKeys, route.errorKeys, route.middleware
```

### 查 MCP/RPC 工具处理器

```cypher
MATCH (handler)-[:CodeRelation {type: 'HANDLES_TOOL'}]->(tool:Tool)
RETURN tool.name, tool.description, handler.name, handler.filePath
```

### 查某个节点的所有关系类型

```cypher
MATCH (n {name: "validateUser"})-[r:CodeRelation]->(m)
RETURN r.type, labels(m)[0] AS targetType, m.name, r.confidence, r.reason
```

## 讲解时怎么说

这套 schema 的关键不是“节点很多”，而是它给 Agent 提供了三类问题的结构化答案：

1. 代码在哪里：File、Folder、Section。
2. 代码是什么：Function、Class、Method、Interface、多语言节点。
3. 代码如何协作：CodeRelation.type 表达调用、导入、继承、实现、流程、社区、路由和工具处理关系。

## 容易讲错的点

- 不要说所有关系都是独立表。当前 LadybugDB 是统一 `CodeRelation` 表。
- 不要说所有 `NodeLabel` 都有持久化表。以 `NODE_TABLES` 为准。
- 不要把 Embedding 当成图边。Embedding 是 `CodeEmbedding` 节点表，用于语义搜索。
- 不要把 `Process` 当成函数。它是执行流聚合节点。
- 不要把 `Community` 当成人工模块。它来自图结构聚类，可被启发式或 LLM 增强。

## 相关笔记

- [[LadybugDB 图存储]]
- [[FTS 与 BM25]]
- [[Embedding 语义搜索]]
- [[MCP 工具清单]]
- [[Pipeline DAG 实现]]
