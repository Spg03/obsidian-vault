---
type: implementation
status: codex-expanded
tags:
  - gitnexus
  - mcp
  - prompt-engineering
  - tools
---

# 工具层如何设计 Prompt

这篇基于 GitNexus 当前源码整理，重点看工具层 prompt 是如何设计出来的。参考入口：

```text
gitnexus/src/mcp/tools.ts
gitnexus/src/mcp/server.ts
gitnexus/src/mcp/resources.ts
gitnexus/src/cli/ai-context.ts
gitnexus/src/mcp/local/local-backend.ts
```

结论先说：GitNexus 的工具层 prompt 不是只写一句工具描述，而是由五层共同构成：

```text
Tool description
  + inputSchema
  + annotations
  + next-step hint
  + resources/prompts/AGENTS.md/Skills
  -> Agent workflow
```

它的目标不是“让模型知道有这个工具”，而是让模型知道：什么时候用、怎么传参、用完之后做什么、什么情况下必须停下来做风险分析。

## 1. 工具定义在哪里

MCP 工具定义集中在：

```text
gitnexus/src/mcp/tools.ts
```

核心类型是：

```ts
export interface ToolDefinition {
  name: string;
  description: string;
  annotations: ToolAnnotations;
  inputSchema: {
    type: 'object';
    properties: Record<string, ...>;
    required: string[];
  };
}
```

这说明 GitNexus 把每个工具的 prompt 分成三部分：

| 部分            | 对 Agent 的作用                    |
| ------------- | ------------------------------ |
| `description` | 语义层提示：这个工具做什么、什么时候用、下一步是什么     |
| `inputSchema` | 结构层提示：参数名、类型、默认值、枚举、范围、必填项     |
| `annotations` | 行为安全提示：只读、破坏性、幂等、是否 open world |

## 2. 三类 ToolAnnotations

`tools.ts` 定义了三种注解模板。

### READ_ONLY_TOOL_ANNOTATIONS

```ts
const READ_ONLY_TOOL_ANNOTATIONS = {
  readOnlyHint: true,
  destructiveHint: false,
  idempotentHint: true,
  openWorldHint: false,
};
```

语义：这是本地、只读、幂等的工具。适用于 `context`、`impact`、`detect_changes`、`route_map`、`shape_check` 等。

注意：`impact` 虽然是修改前工具，但它本身不修改代码，所以仍然是 read-only。

### QUERY_TOOL_ANNOTATIONS

```ts
const QUERY_TOOL_ANNOTATIONS = {
  readOnlyHint: true,
  destructiveHint: false,
  idempotentHint: true,
  openWorldHint: true,
};
```

语义：这是只读查询，但它可能做更开放的检索/召回。GitNexus 给 `query` 用这个注解，因为 `query` 是概念搜索，不是精确结构查询。

### DESTRUCTIVE_TOOL_ANNOTATIONS

```ts
const DESTRUCTIVE_TOOL_ANNOTATIONS = {
  readOnlyHint: false,
  destructiveHint: true,
  idempotentHint: false,
  openWorldHint: false,
};
```

语义：这个工具可能写文件或改变状态。GitNexus 给 `rename` 和 `group_sync` 用这个注解。

这里有一个很重要的设计点：`rename` 默认 `dry_run: true`，但工具本身有执行编辑能力，所以仍然按 destructive 暴露。`group_sync` 写 `contracts.json` 和 `bridge.lbug`，也按 destructive 暴露。

## 3. description 的结构模式

GitNexus 的工具 description 大多不是一句话，而是固定结构：

```text
一句话说明工具能力

Returns ...

WHEN TO USE: ...
AFTER THIS: ...

补充说明：
- 输出结构
- 风险等级
- group mode
- service 参数
- 示例查询
- 边界提醒
```

这其实是一种工具级 prompt 模板。

| 块 | 作用 |
|---|---|
| 能力说明 | 帮模型选择工具 |
| `Returns` | 帮模型预期输出结构 |
| `WHEN TO USE` | 触发条件，减少乱用工具 |
| `AFTER THIS` | 串联工作流，减少单次工具调用后停住 |
| 输出字段说明 | 帮模型解释结果 |
| Group/Service 说明 | 多仓库和 monorepo 场景下正确传参 |
| Risk/Depth 说明 | 引导模型做风险分层 |
| Examples/Tips | 给模型可模仿的调用和查询模式 |

## 4. inputSchema 也是 Prompt

`inputSchema` 看似只是 JSON Schema，其实也是 prompt。

例如 `query` 的参数包括：

```ts
query: Natural language or keyword search query
task_context: What you are working on
goal: What you want to find
limit: Max processes to return
max_symbols: Max symbols per process
include_content: Include full symbol source code
repo: Indexed repository name or group mode
service: Optional monorepo service root
```

这会告诉 Agent：

- 不要只传关键词，也可以传 task_context 和 goal 来改善排序。
- 默认不要拉全文，除非需要 `include_content`。
- 多仓库时必须传 `repo`。
- group/monorepo 时要理解 `service`。

再比如 `impact` 的参数：

```ts
target
target_uid
direction
file_path
kind
maxDepth
crossDepth
relationTypes
includeTests
minConfidence
repo
service
subgroup
timeoutMs
```

这些参数本身就在告诉 Agent：影响分析不是一个粗粒度 yes/no，它可以被限定方向、深度、关系类型、置信度、测试文件和跨仓库扇出范围。

## 5. 工具清单完整口径

当前 `GITNEXUS_TOOLS` 一共有 13 个工具。

| 工具 | 注解 | Prompt 设计重点 |
|---|---|---|
| `list_repos` | read-only | 多 repo 发现；提示下一步读 repo context；多仓库时必须指定 `repo` |
| `query` | query/read-only/open-world | 概念搜索执行流；强调不是 grep；返回 process grouped 结果；下一步 context |
| `cypher` | read-only | 高级结构查询；先读 schema；内置节点、边、示例查询和 Tips |
| `context` | read-only | 单符号 360 度上下文；处理重名消歧；下一步 impact 或 process trace |
| `detect_changes` | read-only | 提交前分析 git diff；映射 changed symbols 和 affected processes |
| `rename` | destructive | 图谱 + 文本搜索协同重命名；默认 dry_run；下一步 detect_changes |
| `impact` | read-only | 修改前影响面；按 d=1/d=2/d=3 和风险等级解释 |
| `route_map` | read-only | API 路由、handler、consumer 映射；修改前更推荐 api_impact |
| `tool_map` | read-only | MCP/RPC 工具定义和 handler 映射 |
| `shape_check` | read-only | API 返回字段和消费者访问字段漂移检查 |
| `api_impact` | read-only | API handler 修改前综合影响报告 |
| `group_list` | read-only | 查看 group 配置，作为 group_sync 前置 |
| `group_sync` | destructive | 重建 contracts.json 和 bridge；group.yaml 或成员 repo 重建后使用 |

下面逐个展开。

## 6. list_repos 的 Prompt 设计

源码描述重点：

```text
WHEN TO USE: First step when multiple repos are indexed, or to discover available repos.
AFTER THIS: READ gitnexus://repo/{name}/context ...
When multiple repos are indexed, you MUST specify the "repo" parameter ...
```

设计意图：

1. 让 Agent 在多仓库环境下先发现 repo，而不是随便默认一个。
2. 明确下一步不是直接 query，而是读 repo context。
3. 用 MUST 强调后续工具要带 `repo`。

这是典型“入口工具 prompt”：它不是为了给答案，而是为了把 Agent 带到正确上下文。

## 7. query 的 Prompt 设计

`query` 的 description 里有几个关键点：

- 查询 code knowledge graph。
- 返回和概念相关的 execution flows。
- 不是文件匹配，而是流程和关系。
- 结果按 process 分组。
- 使用 BM25 + 向量语义搜索 + RRF 混合排序。
- 支持 group mode 和 service。
- 下一步用 context 深入具体符号。

设计意图：

```text
用户问“支付流程怎么工作”
  -> query 找执行流
  -> context 看核心符号
  -> process resource 看完整 trace
```

它明确把 Agent 从 grep 思路拉出来：

> Use this when you need execution flows and relationships, not just file matches.

这是 GitNexus 工具层 prompt 的核心价值之一。

## 8. cypher 的 Prompt 设计

`cypher` 的 description 最长，因为它需要给模型足够 schema 上下文。

它包含：

- WHEN TO USE：复杂结构查询。
- 前置动作：先读 `gitnexus://repo/{name}/schema`。
- 节点说明：File、Folder、Function、Class、Interface、Method、CodeElement、Community、Process、Route、Tool。
- 多语言节点：`Struct`、`Enum`、`Trait`、`Impl` 等要用反引号。
- 边说明：统一 `CodeRelation` 表，通过 `type` 过滤。
- 边属性：type、confidence、reason、step。
- 示例查询：caller、community、process、methods、properties、field writers、method overrides、diamond inheritance。

设计意图：

`cypher` 是高级工具，模型很容易写错 schema，所以工具 description 直接内置 mini schema 和示例，让模型可以模仿。

这是一种“把 DSL 教给模型”的 prompt 设计。

## 9. context 的 Prompt 设计

`context` 强调：

- 单符号 360 度视图。
- incoming/outgoing references。
- calls/imports/extends/implements/methods/properties/overrides。
- process participation。
- 重名消歧：uid、file_path、kind。
- ACCESSES 边和 field read/write。
- CALLS 可以经过 field access chain 和 method-call chain。
- group mode 和 service。

设计意图：

`query` 找到候选后，Agent 不能立刻改，必须先把目标符号放到完整关系网里看。

所以 `context` 的 AFTER THIS 是：

```text
Use impact() if planning changes,
or READ process resource for full execution trace.
```

这里把“理解”和“修改前安全检查”衔接起来。

## 10. detect_changes 的 Prompt 设计

`detect_changes` 是提交前工具。description 强调：

- 分析 uncommitted git changes。
- 把 git diff hunks 映射到 indexed symbols。
- 再追踪 affected processes。
- 用于 before committing、pre-commit review、PR preparation。
- 支持 unstaged、staged、all、compare。
- 支持 linked git worktree。

设计意图：

它不是修改前工具，而是修改后验证工具。它回答：

> 我实际改动影响了哪些符号和执行流？

这和 `impact` 的关系是：

```text
impact: 修改前预测风险
detect_changes: 修改后验证实际影响
```

## 11. rename 的 Prompt 设计

`rename` 被标记为 destructive，但默认 dry_run。

description 重点：

- 多文件协同重命名。
- 图谱 + 文本搜索。
- 默认 preview。
- 比 find-and-replace 更安全。
- 每个 edit 带 confidence。
- graph 命中高置信，text_search 命中低置信。
- 之后运行 detect_changes。

设计意图：

重命名是 Agent 最容易乱来的任务之一。GitNexus 用 prompt 明确禁止“直接全文替换”的思路，让模型先预览、再验证。

`inputSchema` 也配合这个目标：

- `symbol_uid` 支持零歧义。
- `file_path` 支持消歧。
- `dry_run` 默认 true。

## 12. impact 的 Prompt 设计

`impact` 是 GitNexus 工具层 prompt 的核心工具。

description 包含：

- blast radius。
- 修改前使用。
- 输出 risk、summary、affected_processes、affected_modules、byDepth。
- d=1、d=2、d=3 语义。
- 默认关系类型。
- class member 需要 HAS_METHOD/HAS_PROPERTY。
- field access 需要 ACCESSES。
- 重名消歧。
- confidence。
- group mode 和 service。

最重要的 prompt 是 depth 解释：

```text
d=1: WILL BREAK
d=2: LIKELY AFFECTED
d=3: MAY NEED TESTING
```

这不是普通字段说明，而是在教 Agent 如何解释风险。

`impact` 的工具设计把“图遍历结果”变成“工程决策语言”。

## 13. route_map 的 Prompt 设计

`route_map` 用来展示 API 路由映射：

- 哪些组件或 hooks fetch 哪些 API。
- 哪些 handler 服务这些 endpoint。
- middleware wrapper chain。

Prompt 里还有一个关键引导：

```text
For pre-change analysis, prefer api_impact
```

设计意图：

`route_map` 适合理解结构，但如果要改 API handler，应该升级到 `api_impact`，因为它会组合 route_map、shape_check 和 impact。

## 14. tool_map 的 Prompt 设计

`tool_map` 展示 MCP/RPC 工具定义：

- 工具在哪里定义。
- 哪里处理。
- 描述是什么。

设计意图：

GitNexus 不只分析业务代码，也把工具 API 当成入口边界。修改工具协议时，Agent 应该先看 tool_map，而不是只 grep tool name。

## 15. shape_check 的 Prompt 设计

`shape_check` 解决 API shape drift：

- 路由返回了哪些 top-level keys。
- consumer 访问了哪些 keys。
- consumer 访问了 route 没返回的 key 就 MISMATCH。
- 依赖 Route 节点里的 `responseKeys`。

Prompt 里明确：

```text
For pre-change analysis, prefer api_impact
```

设计意图：

shape_check 是 API 契约层工具。它不是看调用链，而是看“返回形状”和“消费形状”是否一致。

## 16. api_impact 的 Prompt 设计

`api_impact` 是 API handler 修改前的综合工具。

description 里明确：

- BEFORE modifying any API route handler。
- 至少需要 route 或 file。
- 展示 consumers、response fields、middleware、execution flows。
- 风险等级规则：
  - LOW：0-3 consumers。
  - MEDIUM：4-9 consumers 或存在 mismatch。
  - HIGH：10+ consumers 或 4+ consumers mismatch。
- mismatch confidence low 时要说明属性归因近似。
- 组合 route_map、shape_check、impact。

设计意图：

这是一个“复合工具 prompt”：它把多个低阶工具整合成一个修改前工作流，避免 Agent 自己漏步骤。

## 17. group_list 的 Prompt 设计

`group_list` 是 Group 工具入口。

description 很短：

```text
WHEN TO USE: Discover groups before group_sync.
```

设计意图：

Group 操作涉及跨仓库配置和写入，不能直接 sync。先 list，确认有哪些 group、repos、manifest links。

## 18. group_sync 的 Prompt 设计

`group_sync` 会重建 Contract Registry，所以是 destructive。

description：

- Rebuild contracts.json。
- 提取 HTTP contracts。
- 应用 manifest links。
- exact-match cross-links。
- after changing group.yaml or re-indexing member repos。

参数：

- `name` 必填。
- `skipEmbeddings`：描述里目前写 Exact + BM25 only，但源码 sync 当前核心路径是 exact/wildcard/manifest；这点分享时要小心，不要讲成完整语义匹配已经在线。
- `exactOnly`：Exact match only。

设计意图：

告诉 Agent：这是状态重建工具，不是普通查询。用之前应该知道 group 名，通常由 `group_list` 或用户指定。

## 19. getNextStepHint：返回结果后的 Prompt

工具描述解决“调用前怎么选”，`getNextStepHint` 解决“调用后做什么”。源码在：

```text
gitnexus/src/mcp/server.ts
```

它在 `CallToolRequestSchema` handler 里执行：

```ts
const result = await backend.callTool(name, args);
const resultText = typeof result === 'string' ? result : JSON.stringify(result, null, 2);
const hint = getNextStepHint(name, args);
return resultText + hint;
```

当前 hint 映射：

| 工具 | Next-step hint |
|---|---|
| `list_repos` | 读取 `gitnexus://repo/{name}/context`，检查 overview 和 staleness |
| `query` | 用 `context({name})` 深入具体符号 |
| `context` | 如果要修改，运行 `impact({target, direction: "upstream"})`；或读 processes |
| `impact` | 先 review d=1；再读 processes |
| `detect_changes` | review affected processes；对高风险符号用 context；读 process detail |
| `rename` | 运行 detect_changes 验证副作用 |
| `cypher` | 对结果符号用 context；需要 schema 则读 schema resource |
| legacy `search` | 对结果符号用 context |
| legacy `explore` | 如果要修改，运行 impact |
| legacy `overview` | 读 cluster/process resources |

设计意图：

很多 Agent 有“单步工具调用倾向”，拿到结果就停。GitNexus 用 Next-step hint 把工具调用串成路径：

```text
list_repos -> context resource
query -> context
context -> impact
impact -> review d=1 / processes
edit -> detect_changes
```

这是一种非常实用的工具层 prompt 技术：把下一步行动直接贴在工具结果后面。

## 20. MCP Prompts：任务模板层

`server.ts` 还注册了 MCP prompts：

```text
detect_impact
generate_map
```

### detect_impact

Prompt 内容要求 Agent：

1. 运行 `detect_changes`。
2. 对关键流程里的 changed symbol 运行 `context`。
3. 对高风险项运行 `impact`。
4. 输出 risk report。

它把“提交前影响分析”封装成任务模板。

### generate_map

Prompt 内容要求 Agent：

1. 读 repo context。
2. 读 clusters。
3. 读 processes。
4. 读前 5 个关键 process trace。
5. 生成 mermaid 架构图。
6. 写 ARCHITECTURE.md。

它把“从图谱生成架构文档”封装成任务模板。

设计意图：

Tools 是原子能力，Prompts 是任务编排模板。

## 21. Resources：稳定上下文层

`resources.ts` 暴露了静态 resource 和动态 resource template。

静态：

| Resource | 作用 |
|---|---|
| `gitnexus://repos` | 所有 indexed repos |
| `gitnexus://setup` | 返回 AGENTS.md setup 内容 |

动态：

| Resource Template | 作用 |
|---|---|
| `gitnexus://repo/{name}/context` | stats、staleness、可用工具、资源入口 |
| `gitnexus://repo/{name}/clusters` | 功能模块 |
| `gitnexus://repo/{name}/processes` | 执行流列表 |
| `gitnexus://repo/{name}/schema` | Cypher schema |
| `gitnexus://repo/{name}/cluster/{clusterName}` | 模块详情 |
| `gitnexus://repo/{name}/process/{processName}` | 执行流 trace |
| `gitnexus://group/{name}/contracts` | 跨仓库 contract registry |
| `gitnexus://group/{name}/status` | group staleness 状态 |

Resources 的 prompt 作用是提供“稳定、可读、低成本上下文”。比如 `context` resource 会明确列出：

- project。
- staleness。
- stats。
- tools_available。
- re_index 提示。
- resources_available。

这让 Agent 在动手前有一个项目级导航页。

## 22. AGENTS.md：工具层规则外溢到项目上下文

`ai-context.ts` 会生成 AGENTS.md / CLAUDE.md，其中有一段 GitNexus block。

设计原则写在源码注释里：

- Inline critical workflows。
- 使用 RFC 2119 语言：MUST、NEVER、ALWAYS。
- 三层边界：Always / When / Never。
- 保持在一定长度内，避免上下文过长导致遵循度下降。
- 使用具体工具命令和参数。
- 加 self-review checklist。

实际生成内容包括：

```text
MUST run impact analysis before editing any symbol
MUST run detect_changes before committing
MUST warn user if impact returns HIGH or CRITICAL
Use query instead of grepping for unfamiliar code
Use context for full symbol context
NEVER rename with find-and-replace
```

这说明 GitNexus 的工具层 prompt 不只存在 MCP 工具定义里，还会被写进项目上下文，成为 Agent 的长期规则。

## 23. GitNexus 工具层 Prompt 的设计原则

从源码可以总结出 9 条原则。

### 原则 1：工具名要短，description 要完整

工具名如 `query`、`context`、`impact` 很短，方便模型调用。真正的语义放在 description 里。

### 原则 2：每个工具都说明 WHEN TO USE

避免模型“看到工具就乱用”。例如 `route_map` 和 `api_impact` 都涉及 API，但 description 明确 pre-change 时优先 `api_impact`。

### 原则 3：每个关键工具都说明 AFTER THIS

这把工具串成工作流，而不是孤立命令。

### 原则 4：把输出解释写进 prompt

`impact` 解释 d=1/d=2/d=3，`api_impact` 解释 LOW/MEDIUM/HIGH，`rename` 解释 graph/text_search confidence。

### 原则 5：用 schema 约束参数

枚举、默认值、minimum、maximum、required 都是约束模型行为的 prompt。

### 原则 6：用 annotations 给安全信号

只读工具和破坏性工具在 MCP 层有不同信号。尤其 `rename`、`group_sync` 不能伪装成普通 read-only。

### 原则 7：把多仓库口径写进每个相关工具

`repo`、group mode、service、subgroup 分散在 query/context/impact 等工具里，避免 Agent 在多仓库环境下误查默认仓库。

### 原则 8：把任务模板提升到 MCP Prompts

`detect_impact` 和 `generate_map` 不是新工具，而是用已有工具编排任务。

### 原则 9：用 AGENTS.md 和 Skill 做长期约束

工具描述是调用时约束，AGENTS.md 是项目级约束，Skill 是任务级约束。三者叠加才形成稳定工作流。

## 24. 分享时怎么讲

可以用这张逻辑图讲：

```text
工具选择前：Tool description + annotations + inputSchema
工具调用中：LocalBackend 执行真实图谱查询
工具返回后：getNextStepHint 追加下一步
任务级编排：MCP Prompts
项目级约束：AGENTS.md / CLAUDE.md / Skills
```

一句话：

> GitNexus 的工具层 prompt 不是写给人看的 API 文档，而是写给 Agent 的操作协议。它把“查什么、何时查、怎么查、查完做什么、什么时候必须做风险分析”都放进工具层。

## 25. 容易讲错的点

- 不要只讲 AGENTS.md。工具层 prompt 的第一现场其实是 `tools.ts`。
- 不要把 `description` 当普通说明，它是模型选择工具的核心上下文。
- 不要忽略 `inputSchema`，参数说明也是 prompt。
- 不要把 `annotations` 当装饰字段，它给 Agent/客户端安全语义。
- 不要说 `rename` 是安全只读工具，它是 destructive，只是默认 dry_run。
- 不要说 `group_sync` 只是查询，它会写 contracts registry 和 bridge。
- 不要说 `query` 等于 grep，它返回 process-grouped code intelligence。
- 不要说 `impact` 会修改代码，它是 read-only 风险分析。

## 相关笔记

- [[MCP Server 实现]]
- [[MCP 工具清单]]
- [[Prompt Skill AGENTS 注入机制]]
- [[Agent 工作流]]
- [[图谱 Schema 速览]]
