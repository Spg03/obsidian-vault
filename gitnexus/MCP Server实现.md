以下是将你提供的 GitNexus MCP Server 架构说明整理为标准 Markdown 格式的版本。已保留全部内容并调整了标题层级、列表、代码块和表格，便于在 Obsidian 等 Markdown 编辑器中阅读。

```markdown
# GitNexus MCP Server 架构

## 整体架构图

```
![[gitnexus-mcp-architecture-v2.png]]
```

---

## 第一层：两种传输方式

GitNexus MCP 支持两种接入方式，就像一个服务台同时支持电话和柜台两种渠道。

### STDIO 模式（compatible-stdio-transport.ts）

AI 工具通过 stdin/stdout 与 GitNexus 通信。这是最常用的方式——Cursor/Claude Code 启动一个 `gitnexus mcp` 子进程，然后通过管道发 JSON-RPC 消息。

有意思的是，这个 Transport 支持两种帧格式，**自动检测**：

- 首字节是 `{` 或 `[` → newline 分隔（每行一个 JSON）
- 首字节是 `C` → Content-Length 头（HTTP 风格）

具体实现（compatible-stdio-transport.ts:86-101）：`detectFraming()` 看一眼 buffer 的第一个字节就能判断客户端用的是哪种格式。这种兼容性让 GitNexus 能同时对接不同版本的 MCP 客户端。

### HTTP 模式（mcp-http.ts）

通过 Express 的 `/api/mcp` 端点提供服务。每个客户端有独立的 session（30 分钟 TTL），后台每 5 分钟清理过期 session。

- 客户端 → POST `/api/mcp`（无 session-id） → 创建新 session → 返回 session-id
- 客户端 → POST `/api/mcp`（带 session-id） → 复用已有 session

---

## 第二层：Stdio 安全机制——“哨兵模式”

这是整个 MCP 里最精妙的设计之一，用人话说就是一个防止“乱说话”的网关。

### 问题是什么？

MCP 协议规定：stdout 上只能输出合法的 JSON-RPC 消息。但 GitNexus 加载时，依赖库（如 ONNX Runtime、transformers.js）可能会往 stdout 打印日志。哪怕只是一行 `"Loading model..."`，都会破坏协议，导致客户端报 `MCP error -32000`。

### 哨兵怎么工作？

`stdio-context.ts` 实现了一个精巧的三层拦截：

```
 ![[gitnexus-mcp-stdout-sentinel-flow 1.png]]```

核心是利用 Node.js 的 `AsyncLocalStorage`（stdio-context.ts:37）：

1. Transport 发送消息时，调用 `withMcpWrite(() => stdout.write(payload))`，在 `AsyncLocalStorage` 里打个标记 `{mcp: true}`
2. 哨兵的 `write` 函数（第 104 行）检查这个标记：
   - 有标记 → 这是合法的 MCP 消息 → 放行到真正的 stdout
   - 无标记 → 是依赖库的 stray write → 重定向到 stderr，加上 `[mcp:stdout-redirect]` 前缀
3. **限流保护**：最多重定向 10 次，超出后静默丢弃，防止某个库疯狂输出刷爆日志

还有一个关键细节——`stdio-capture.ts`。它在模块加载的第一时间就捕获了真正的 `stdout.write` 和 `stderr.write`。这是因为 `pool-adapter.ts` 里的 `silenceStdout`/`restoreStdout` 会在加载模型时临时替换 `stdout.write`。如果不提前捕获，restore 的时候就会“恢复”到错误的函数。这就是为什么 `stdio-capture.ts` 被设计成零依赖的叶子模块——连 `@ladybugdb/core` 都不能 import，避免触发 native module 的加载横幅。

---

## 第三层：MCP Server——Protocol Handler

`server.ts` 就是标准的 MCP SDK 用法，注册了三种 Handler。

### Tools Handler（第 156-193 行）

- `listTools` 请求 → 返回 `GITNEXUS_TOOLS` 的定义（11 个工具的 JSON Schema）
- `callTool`  请求 → 转发给 `backend.callTool(name, args)` → 自动追加 next-step hint

### Resources Handler（第 102-152 行）

- `listResources`    → 2 个静态资源 (repos, setup)
- `listTemplates`    → 8 个动态模板 (repo/{name}/context, ...)
- `readResource`     → 解析 URI → 调用对应的 resource 函数

### Prompts Handler（第 197-278 行）

两个预定义的 Agent 工作流模板：
- `detect_impact`：引导 Agent 依次执行 `detect_changes` → `context` → `impact` → 输出风险报告
- `generate_map`：引导 Agent 读取 clusters/processes → 生成架构文档

### Next-Step Hints——隐式工作流引导

这是一个聪明的设计（server.ts:40-78）。每个工具返回时自动追加一个 `---\n**Next:** ...` 提示，告诉 Agent“下一步该做什么”。例如：

- `query`    → "Next: 用 context() 深入了解具体符号"
- `context`  → "Next: 如果要改代码，用 impact() 检查爆炸半径"
- `impact`   → "Next: 先看 d=1 的项目（一定会坏）"

这相当于在工具层面内置了一个隐式的工作流导航，不需要 Agent 自己推理下一步。

---

## 第四层：LocalBackend——真正的业务逻辑

这是最“肥”的一层，`local-backend.ts` 实现了所有工具的查询逻辑。

### 多仓库管理（resolveRepo，第 444 行）

```
resolveRepo("GitNexus")
  → 先在内存缓存里找（id匹配 → 名称匹配 → 路径匹配 → 模糊匹配）
  → 找不到就重新读注册表（refreshRepos）
  → 再找不到就抛错，列出所有可用仓库
```

支持通过 `repo` 参数指定目标仓库。只有一个仓库时可以省略。还做了 **sibling drift** 检测——如果用户 cwd 在另一个 clone 里（同一 remoteUrl 但不同路径），会警告但不会拒绝服务。

### 懒加载 LadybugDB（ensureInitialized，第 528 行）

- 首次查询某仓库 → `initLbug(repoId, lbugPath)` → 记住已初始化
- 后续查询 → 检查 `meta.json` 的 `indexedAt` 是否变化
  - 如果索引被重建了 → 关闭旧连接 → 重新打开
  - 节流：5 秒内最多检查一次

### query 工具——混合搜索（第 802 行）

这是最核心的搜索流程，分 5 步：

**Step 1: 并行搜索**
- BM25（FTS 全文索引）
- Semantic（向量相似度）  
  `Promise.all` 并发

**Step 2: RRF 合并**  
Reciprocal Rank Fusion: `score = 1/(60+rank)`  
同一个符号可能同时被 BM25 和语义搜索命中，分数累加

**Step 3: 符号→流程映射**  
对每个匹配的符号，查 Cypher：
```cypher
MATCH (n)-[:STEP_IN_PROCESS]->(p:Process)
```
同时查社区归属：
```cypher
MATCH (n)-[:MEMBER_OF]->(c:Community)
```

**Step 4: 按流程分组 + 排名**  
每个流程的得分 = 包含的匹配符号得分之和 + cohesion * 0.1  
cohesion 是一种“内聚度”信号，优先展示更内聚的模块

**Step 5: 格式化输出**  
返回 `{ processes: [...], process_symbols: [...], definitions: [...] }`

### Embedder——语义搜索（core/embedder.ts）

用的是 Snowflake arctic-embed-xs（384 维），通过 HuggingFace 的 transformers.js 加载：

用户查询 `"MCP server"`  
→ `embedQuery("MCP server")` → `[0.12, -0.34, ...]`（384 维）  
→ 在 `EMBEDDING_TABLE` 里做向量相似度搜索  
→ 返回最相似的符号

支持 DML（DirectML，Windows GPU）、CUDA、CPU 三种设备，自动 fallback。模型只在第一次搜索时加载（懒加载），之后常驻内存。

---

## 第五层：11 个 MCP 工具一览

| 工具           | 类型   | 功能                                 |
|----------------|--------|--------------------------------------|
| list_repos     | 只读   | 列出所有已索引仓库                   |
| query          | 只读   | 混合搜索（BM25+语义），按执行流分组   |
| cypher         | 只读   | 直接执行 Cypher 图查询               |
| context        | 只读   | 360° 符号视图（调用者/被调用者/流程参与） |
| impact         | 只读   | 爆炸半径分析（d=1/2/3 深度）         |
| detect_changes | 只读   | Git diff → 受影响符号和流程          |
| rename         | 破坏性 | 基于知识图谱的安全重命名              |
| route_map      | 只读   | API 路由映射                         |
| tool_map       | 只读   | MCP/RPC 工具定义映射                 |
| shape_check    | 只读   | API 响应形状 vs 消费者属性访问对比    |
| api_impact     | 只读   | API 路由变更影响分析                 |
| group_list     | 只读   | 列出仓库组                           |
| group_sync     | 破坏性 | 同步跨仓库契约注册表                 |

---

## 设计亮点总结

1. **双传输层**：STDIO（本地 Agent）+ HTTP（Web UI / 远程），共享同一套 `createMCPServer`
2. **哨兵模式**：`AsyncLocalStorage` 标记 + Proxy 拦截，保证 MCP 协议不被“污染”
3. **零依赖叶子模块**：`stdio-capture.ts` 确保 stdout 引用在最早期就被安全捕获
4. **混合搜索**：BM25 + 语义搜索并行，RRF 融合，兼顾精确匹配和语义理解
5. **懒加载 + 热更新**：LadybugDB 按需初始化，检测索引重建自动重连，5 秒节流
6. **隐式工作流**：Next-step hints 让 Agent 不需要推理“下一步该做什么”
7. **多仓库 + 组模式**：支持 `@groupName` 前缀实现跨仓库查询
