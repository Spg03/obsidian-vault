```markdown
# GitNexus Group 机制——跨仓库代码图谱

## 一句话说清楚

**Group** 解决的核心问题是：你有 5 个微服务仓库，A 调了 B 的 API，B 发了消息给 C。怎么让 GitNexus 知道这些跨仓库的依赖关系？

答案就是 Group——把多个仓库“编织”成一张更大的图。

---

## 整体数据流

```
![[gitnexus-group-cross-repo-data-flow.png]]
```

---

## 第一层：group.yaml——群的“身份证”

`config-parser.ts` 负责解析和验证。一个典型的 `group.yaml` 长这样：

```yaml
version: 1
name: my-microservices
description: "我们的微服务体系"

repos:
  gateway: my-org/api-gateway        # key = 在group里的名字, value = registry里的仓库名
  users: my-org/user-service
  orders: my-org/order-service

links:                                # 手工声明的跨仓库关系
  - from: gateway
    to: users
    type: http
    contract: "POST /api/login"       # 具体契约标识
    role: provider                    # users 提供这个接口
  - from: gateway
    to: users
    type: http
    contract: "GET /api/users/:id"
    role: provider

detect:                               # 自动检测开关
  http: true
  grpc: true
  thrift: true
  topics: true
  shared_libs: true
  workspace_deps: false               # 默认关闭，需要显式开启
  includes: false

matching:
  bm25_threshold: 0.7
  embedding_threshold: 0.65
  max_candidates_per_step: 3
  exclude_links_paths:                # 排除“噪音”路径
    - /health
    - /ping
```

**设计细节**——`parseGroupConfig`（config-parser.ts:46）对新字段采用回退默认值策略。比如 `workspace_deps` 默认 `false`，这样老用户升级后不会突然多出自动发现的 link。新功能都是 opt-in。

---

## 第二层：Contract Extraction——自动发现“接口边界”

`contract-extractor.ts` 定义了统一的 Extractor 接口：

```typescript
interface ContractExtractor {
  type: ContractType;           // http | grpc | thrift | topic | lib | include
  canExtract(repo): boolean;    // 这个仓库能不能提取
  extract(dbExecutor, repoPath): ExtractedContract[];
}
```

### 6 种 Extractor 一览

| Extractor | 文件 | 提取什么 |
|-----------|------|----------|
| HttpRouteExtractor | http-route-extractor.ts | Express/FastAPI/Go-Chi 等框架的路由定义 |
| GrpcExtractor | grpc-extractor.ts | gRPC service 定义（proto 文件） |
| ThriftExtractor | thrift-extractor.ts | Thrift service 定义 |
| TopicExtractor | topic-extractor.ts | Kafka/RabbitMQ topic 的发布和订阅 |
| IncludeExtractor | include-extractor.ts | C/C++ 头文件暴露的公开 API |
| ManifestExtractor | manifest-extractor.ts | group.yaml 里的 links 手工声明 |

每个 Extractor 有自己的语言模式识别文件，比如 `http-patterns/go.ts` 知道怎么从 Go 代码里提取路由：

```go
// Go: r.GET("/users/:id", handler) → GET /users/:id
// Python: @app.route("/users/<id>") → GET /users/:id
// Node: app.get("/users/:id", ...)  → GET /users/:id
```

### 提取结果

每个 Contract 的结构：

```json
{
  "contractId": "http::POST::/api/login",  // 全局唯一的“契约标识”
  "type": "http",
  "role": "provider",                       // provider 或 consumer
  "symbolUid": "gitnexus:...",             // 图谱中的符号 ID
  "symbolRef": { "filePath": "src/routes.ts", "name": "loginHandler" },
  "confidence": 0.9,
  "service": "gateway",                     // monorepo 里的服务边界
  "meta": { "method": "POST", "path": "/api/login", "source": "auto" }
}
```

`role` 很有意思——一个 Express 路由 `app.post("/login", handler)` 既是 provider（提供这个接口），也可能同时是 consumer（如果 `handler` 内部 `fetch` 了其他服务）。GitNexus 会分别提取。

### Monorepo 的 Service Boundary 检测

`service-boundary-detector.ts` 检测 monorepo 里的子服务边界（如 `services/auth/`、`services/billing/`），给每个 contract 打上 `service` 标签。这样同一个 monorepo 里的不同子服务之间也能建立 cross-link。

---

## 第三层：同步流程（sync.ts）——把契约织成网络

`syncGroup()` 是核心编排函数，分三步：

### Step 1: 遍历每个仓库，提取契约（sync.ts:110-203）

```
for each (groupPath, registryName) in config.repos:
  1. 解析仓库路径 → 打开 LadybugDB
  2. 检测 service boundaries (monorepo)
  3. 依次跑 5 种 auto-extractor:
     HTTP → gRPC → Thrift → Topic → Include
  4. 每个 contract 打上 repo + service 标签
  5. 记录 repo snapshot (indexedAt, lastCommit)
```

### Step 2: 处理 Manifest Links + Workspace Deps（sync.ts:211-249）

1. 自动发现 workspace 依赖（Cargo workspace, Go workspace...）
2. 合并 `group.yaml` 的 `links` + 自动发现的 workspace links
3. `ManifestExtractor` 把 link 转换成 CrossLink

### Step 3: 三级匹配级联（matching.ts）

这是最精彩的部分——怎么把 consumer 和 provider 配对？

> Consumer: gateway 里 `fetch("POST /api/login")`  
> Provider: users 里的 `app.post("/login", handler)`

**匹配级联**:
- **Step A: Exact Match** → `"POST::/api/login" === "POST::/api/login"` ✓
- **Step B: Wildcard** → gRPC: `"UserService/*"` 匹配 `"UserService/Login"`
- **Step C: BM25** → 文本相似度 ≥ 0.7
- **Step D: Embedding** → 语义向量相似度 ≥ 0.65 (可选)

`normalizeContractId`（normalization.ts:54）是关键——同一个接口在不同语言里写法不同：
- `POST /api/login`、`post /api/Login`、`POST /api/login/` → 全部规范化为 `http::POST::/api/login`
- gRPC 服务名大小写不敏感：`AuthService`、`authService`、`authservice` → 全部小写

**噪音过滤**（matching.ts:32-52）：health check 端点（`/ping`、`/health`）每个服务都有，如果不排除会产生 N×M 的假链接。

---

## 第四层：Bridge DB——跨仓库的“关系总机”

`bridge-db.ts` 是 Group 系统的物理存储层。它使用一个独立的 LadybugDB 实例（`bridge.lbug`）来存储跨仓库数据。

### 数据模型

**节点类型**:
- `Contract { id, contractId, type, role, repo, service, symbolUid, filePath, symbolName, confidence, meta }`
- `RepoSnapshot { id, indexedAt, lastCommit }`

**关系类型**:
- `ContractLink { matchType, confidence, contractId, fromRepo, toRepo }`
- `(a:Contract)-[:ContractLink]->(b:Contract)`

### 原子写入 + 崩溃恢复

`writeBridge`（bridge-db.ts:359）的设计非常严谨：

1. 创建临时目录 (mkdtemp, 随机后缀, 防预测路径攻击)
2. 在临时目录里写入完整 `bridge.lbug`
3. 旧的 `bridge.lbug` → `bridge.lbug.bak` (备份)
4. 新的 `bridge.lbug (tmp)` → `bridge.lbug` (原子 rename)
5. 删除 `.bak`
6. 写入 `meta.json`

如果第4步之前崩溃 → 下次启动从 `.bak` 恢复

### ContractLookupIndex（bridge-db.ts:72-162）

这是一个内存索引，消除了 N+1 查询——之前每个 cross-link 要做最多 6 次 DB 查询，现在是 O(1) 纯函数查找。

### 三级查找（findContractNode）

解析 cross-link 的端点时，按优先级查找：
1. `symbolUid` 精确匹配 ——最强锚定，不怕重命名
2. `filePath + symbolName` 匹配 ——第二选择
3. `filePath` 里恰好只有一个 contract ——无奈的 fallback

---

## 第五层：Cross-Repo Impact——跨仓库爆炸半径

`cross-impact.ts` 实现了 `impact({target: "loginHandler", repo: "@my-group"})` 的跨仓库部分。

### 两阶段执行

**Phase 1: 本地 impact (当前仓库)**
```
impact(target, direction=upstream, maxDepth)
→ 找出仓库内谁调了 loginHandler
→ 收集受影响的 symbolUids
```

**Phase 2: 桥接扇出**
查 `bridge.lbug`：

```cypher
-- 上游：谁消费了我的接口？
MATCH (consumer:Contract)-[:ContractLink]->(provider:Contract)
WHERE provider.repo = $localRepo
  AND provider.symbolUid IN $uids
```

→ 找到跨仓库的消费者  
→ 对每个消费者运行 `impactByUid` (带超时控制)  
→ 合并所有结果

### 安全阀

- **timeout 钳制**：100ms ~ 5min（`clampTimeout`），防止一个请求拖垮服务器
- **crossDepth 限制**：当前只支持 1 跳（`MAX_SUPPORTED_CROSS_DEPTH = 1`），多跳预留未来
- **graceful degradation**：某个跨仓库邻居的 impact 挂了不影响其他的

---

## 第六层：@groupName 语法——MCP 如何使用 Group

这是我们之前 MCP 那讲里反复出现的 `@groupName` 前缀。`resolve-at-member.ts` 就是它的实现：

**用户写**: `query({query: "auth", repo: "@my-group"})`

```
↓
resolveAtGroupMemberRepoPath("my-group", undefined)
  1. 找 ~/.gitnexus/groups/my-group/group.yaml
  2. 读取 config.repos
  3. explicitMemberPath 没传 → 返回第一个 repo (字典序)
  4. → repoPath = "gateway"
```

**用户写**: `query({query: "auth", repo: "@my-group/users"})`

```
  1. explicitMemberPath = "users"
  2. 验证 "users" 在 config.repos 里
  3. → repoPath = "users"
```

### 在 LocalBackend 里的路由（local-backend.ts:746-751）

```typescript
// callTool 方法里
if (repoParam.startsWith('@')) {
  return this.callToolAtGroupRepo(method, params);
}
// 否则走正常单仓库路径
```

`callToolAtGroupRepo` 对不同的工具做不同的处理：
- `query/context`：带 `service` 前缀过滤
- `impact`：走 `cross-impact.ts` 的两阶段流程
- `group_list/group_sync`：直接调 GroupService

---

## 数据流全景总结

![[gitnexus-group-data-flow-overview.png]]

---

## 设计亮点

| 亮点 | 说明 |
|------|------|
| 自动提取 + 手工标注互补 | auto-extractor 覆盖常见模式，group.yaml links 覆盖特殊情况 |
| 四级匹配级联 | Exact → Wildcard → BM25 → Embedding，逐级降级 |
| Contract ID 规范化 | `POST /api/login` 和 `post /api/Login/` 归一化为同一标识 |
| 噪音过滤 | `/health`、`/ping` 自动排除，避免 N×M 假链接 |
| 原子写入 + 崩溃恢复 | tmp → rename + .bak 备份 |
| 内存索引优化 | ContractLookupIndex 消除 N+1 查询 |
| Monorepo 感知 | service boundary 检测 + service 前缀过滤 |
| 超时钳制 | 跨仓库 impact 有严格的时间预算，防止雪崩 |
| 新功能 opt-in | workspace_deps、includes 等默认关闭，不破坏老用户 |
