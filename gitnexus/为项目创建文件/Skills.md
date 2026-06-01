![[Pasted image 20260525091415.png]]
## 1.gitnexus-cli下的skill
```markdown
---
name: gitnexus-cli
description: "当用户需要运行 GitNexus CLI 命令时使用，例如分析/索引仓库、检查状态、清理索引、生成 wiki 或列出已索引的仓库。示例：“索引这个仓库”、“重新分析代码库”、“生成 wiki”"
---

# GitNexus CLI 命令

所有命令均通过 `npx` 运行 — 无需全局安装。

## 命令

### analyze — 构建或刷新索引

```bash
npx gitnexus analyze
```

在项目根目录下运行。该命令解析所有源文件，构建知识图谱，写入 `.gitnexus/` 目录，并生成 CLAUDE.md / AGENTS.md 上下文文件。

| 标志              | 作用                                                               |
| ----------------- | ------------------------------------------------------------------ |
| `--force`         | 即使已最新也强制完全重新索引                                       |
| `--embeddings`    | 启用嵌入生成以支持语义搜索（默认关闭）                             |
| `--drop-embeddings` | 重建时删除现有嵌入。默认情况下，不带 `--embeddings` 的 `analyze` 会保留它们。 |

**何时运行：** 第一次进入项目、重大代码变更后，或当 `gitnexus://repo/{name}/context` 报告索引已过时。在 Claude Code 中，`git commit` 和 `git merge` 之后的 PostToolUse 钩子会检测到过时状态，并通知代理运行 `analyze` — 钩子本身不会运行 analyze，以避免阻塞代理长达 120 秒并在超时时导致 KuzuDB 损坏。

### status — 检查索引新鲜度

```bash
npx gitnexus status
```

显示当前仓库是否具有 GitNexus 索引、上次更新时间以及符号/关系计数。用它来检查是否需要重新索引。

### clean — 删除索引

```bash
npx gitnexus clean
```

删除 `.gitnexus/` 目录并从全局注册表中注销该仓库。在索引损坏时或在项目中移除 GitNexus 之后，重新索引前使用。

| 标志      | 作用                                   |
| --------- | -------------------------------------- |
| `--force` | 跳过确认提示                           |
| `--all`   | 清理所有已索引的仓库，而不仅仅是当前仓库 |

### wiki — 从图谱生成文档

```bash
npx gitnexus wiki
```

使用 LLM 从知识图谱生成仓库文档。首次使用时需要 API 密钥（保存到 `~/.gitnexus/config.json`）。

| 标志                 | 作用                                       |
| -------------------- | ------------------------------------------ |
| `--force`            | 强制完全重新生成                           |
| `--model <model>`    | LLM 模型（默认：minimax/minimax-m2.5）     |
| `--base-url <url>`   | LLM API 基础 URL                           |
| `--api-key <key>`    | LLM API 密钥                               |
| `--concurrency <n>`  | 并行 LLM 调用数（默认：3）                 |
| `--gist`             | 将 wiki 作为公共 GitHub Gist 发布          |

### list — 列出所有已索引的仓库

```bash
npx gitnexus list
```

列出 `~/.gitnexus/registry.json` 中注册的所有仓库。MCP 工具 `list_repos` 提供相同信息。

## 索引之后

1. 读取 `gitnexus://repo/{name}/context` 以验证索引已加载
2. 使用其他 GitNexus 技能（`exploring`、`debugging`、`impact-analysis`、`refactoring`）完成你的任务

## 故障排除

- **“Not inside a git repository”**：在 git 仓库内的目录中运行命令
- **重新分析后索引仍然过时**：重启 Claude Code 以重新加载 MCP 服务器
- **嵌入生成缓慢**：省略 `--embeddings`（默认关闭）或设置 `OPENAI_API_KEY` 以使用更快的基于 API 的嵌入生成

---
## 2.gitnexus-debugging下的skill
```markdown
---
name: gitnexus-debugging
description: "当用户正在调试一个 bug、追踪某个错误或询问为什么某处失败时使用。示例：“为什么 X 失败了？”、“这个错误从哪来？”、“追踪这个 bug”"
---

# 使用 GitNexus 调试

## 何时使用

- “为什么这个函数失败了？”
- “追踪这个错误从哪来”
- “谁调用了这个方法？”
- “这个端点返回 500”
- 调查 bug、错误或意外行为

## 工作流

```
1. gitnexus_query({query: "<错误或症状>"})            → 查找相关的执行流
2. gitnexus_context({name: "<可疑对象>"})              → 查看调用方/被调用方/流程
3. READ gitnexus://repo/{name}/process/{name}         → 追踪执行流
4. gitnexus_cypher({query: "MATCH path..."})          → 按需自定义追踪
```

> 如果提示“Index is stale” → 在终端中运行 `npx gitnexus analyze`。

## 检查清单

```
- [ ] 理解症状（错误消息、意外行为）
- [ ] 对错误文本或相关代码使用 gitnexus_query
- [ ] 从返回的流程中识别可疑函数
- [ ] 使用 gitnexus_context 查看调用方和被调用方
- [ ] 如果适用，通过流程资源追踪执行流
- [ ] 按需使用 gitnexus_cypher 进行自定义调用链追踪
- [ ] 阅读源文件以确认根本原因
```

## 调试模式

| 症状                | GitNexus 方法                                              |
| ------------------- | ---------------------------------------------------------- |
| 错误消息            | 对错误文本使用 `gitnexus_query` → 对抛出点使用 `context`   |
| 返回值错误          | 对函数使用 `context` → 追踪被调用方以分析数据流            |
| 间歇性失败          | 使用 `context` → 查找外部调用、异步依赖                    |
| 性能问题            | 使用 `context` → 查找具有大量调用方的符号（热点路径）      |
| 最近的回归          | 使用 `detect_changes` 查看你的变更影响了什么               |

## 工具

**gitnexus_query** — 查找与错误相关的代码：

```
gitnexus_query({query: "payment validation error"})
→ 流程: CheckoutFlow, ErrorHandling
→ 符号: validatePayment, handlePaymentError, PaymentException
```

**gitnexus_context** — 获取可疑对象的完整上下文：

```
gitnexus_context({name: "validatePayment"})
→ 传入调用: processCheckout, webhookHandler
→ 传出调用: verifyCard, fetchRates (外部 API！)
→ 流程: CheckoutFlow (第 3/7 步)
```

**gitnexus_cypher** — 自定义调用链追踪：

```cypher
MATCH path = (a)-[:CodeRelation {type: 'CALLS'}*1..2]->(b:Function {name: "validatePayment"})
RETURN [n IN nodes(path) | n.name] AS chain
```

## 示例：“支付端点间歇性返回 500”

```
1. gitnexus_query({query: "payment error handling"})
   → 流程: CheckoutFlow, ErrorHandling
   → 符号: validatePayment, handlePaymentError

2. gitnexus_context({name: "validatePayment"})
   → 传出调用: verifyCard, fetchRates (外部 API！)

3. READ gitnexus://repo/my-app/process/CheckoutFlow
   → 第 3 步: validatePayment → 调用 fetchRates (外部)

4. 根本原因: fetchRates 调用外部 API 时没有合适的超时
```


---
## 3.gitnexus-exploring下的skill

```markdown
---
name: gitnexus-exploring
description: "当用户询问代码如何工作、想了解架构、追踪执行流或探索代码库中不熟悉的部分时使用。示例：“X 是如何工作的？”、“谁调用了这个函数？”、“给我展示认证流程”"
---

# 使用 GitNexus 探索代码库

## 何时使用

- “认证是如何工作的？”
- “项目结构是什么样的？”
- “展示主要组件”
- “数据库逻辑在哪里？”
- 理解之前未看过的代码

## 工作流

```
1. READ gitnexus://repos                          → 发现已索引的仓库
2. READ gitnexus://repo/{name}/context            → 代码库概览，检查是否过时
3. gitnexus_query({query: "<你想理解的内容>"})    → 查找相关的执行流
4. gitnexus_context({name: "<符号>"})             → 深入特定符号
5. READ gitnexus://repo/{name}/process/{name}     → 追踪完整的执行流
```

> 如果第 2 步提示“Index is stale” → 在终端中运行 `npx gitnexus analyze`。

## 检查清单

```
- [ ] READ gitnexus://repo/{name}/context
- [ ] 对你想要理解的概念使用 gitnexus_query
- [ ] 查看返回的流程（执行流）
- [ ] 对关键符号使用 gitnexus_context 查看调用方/被调用方
- [ ] READ 流程资源以获取完整的执行轨迹
- [ ] 阅读源文件以获取实现细节
```

## 资源

| 资源                                     | 你获得的内容                                                |
| ---------------------------------------- | ----------------------------------------------------------- |
| `gitnexus://repo/{name}/context`         | 统计信息、过时警告（约 150 tokens）                         |
| `gitnexus://repo/{name}/clusters`        | 所有功能区域及其内聚性得分（约 300 tokens）                 |
| `gitnexus://repo/{name}/cluster/{name}`  | 区域成员及其文件路径（约 500 tokens）                       |
| `gitnexus://repo/{name}/process/{name}`  | 逐步执行轨迹（约 200 tokens）                               |

## 工具

**gitnexus_query** — 查找与某个概念相关的执行流：

```
gitnexus_query({query: "payment processing"})
→ 流程: CheckoutFlow, RefundFlow, WebhookHandler
→ 按流程分组的符号及文件位置
```

**gitnexus_context** — 符号的 360 度视图：

```
gitnexus_context({name: "validateUser"})
→ 传入调用: loginHandler, apiMiddleware
→ 传出调用: checkToken, getUserById
→ 流程: LoginFlow (第 2/5 步), TokenRefresh (第 1/3 步)
```

## 示例：“支付处理是如何工作的？”

```
1. READ gitnexus://repo/my-app/context       → 918 个符号，45 个流程
2. gitnexus_query({query: "payment processing"})
   → CheckoutFlow: processPayment → validateCard → chargeStripe
   → RefundFlow: initiateRefund → calculateRefund → processRefund
3. gitnexus_context({name: "processPayment"})
   → 传入: checkoutHandler, webhookHandler
   → 传出: validateCard, chargeStripe, saveTransaction
4. 阅读 src/payments/processor.ts 以获取实现细节
```
```


---
## 4.gitnexus-guide下的skill
```markdown
---
name: gitnexus-guide
description: "当用户询问 GitNexus 本身时使用 — 可用的工具、如何查询知识图谱、MCP 资源、图谱模式或工作流参考。示例：“有哪些 GitNexus 工具可用？”、“如何使用 GitNexus？”"
---

# GitNexus 指南

所有 GitNexus MCP 工具、资源和知识图谱模式的快速参考。

## 始终从这里开始

对于任何涉及代码理解、调试、影响分析或重构的任务：

1. **读取 `gitnexus://repo/{name}/context`** — 代码库概览 + 检查索引新鲜度
2. **将你的任务与下方的技能匹配** 并 **读取该技能文件**
3. **遵循该技能的工作流和检查清单**

> 如果第 1 步警告索引已过时，请先在终端中运行 `npx gitnexus analyze`。

## 技能

| 任务                                 | 要读取的技能                |
| ------------------------------------ | --------------------------- |
| 理解架构 / “X 是如何工作的？”        | `gitnexus-exploring`        |
| 爆炸半径 / “如果我修改 X，什么会坏？” | `gitnexus-impact-analysis`  |
| 追踪 bug / “为什么 X 失败了？”       | `gitnexus-debugging`        |
| 重命名 / 提取 / 拆分 / 重构          | `gitnexus-refactoring`      |
| 工具、资源、模式参考                  | `gitnexus-guide`（本文件）  |
| 索引、状态、清理、wiki CLI 命令       | `gitnexus-cli`              |

## 工具参考

| 工具             | 它提供的能力                                                              |
| ---------------- | ------------------------------------------------------------------------- |
| `query`          | 按流程分组的代码智能 — 与某个概念相关的执行流                             |
| `context`        | 符号的 360 度视图 — 分类的引用、它参与的流程                              |
| `impact`         | 符号的爆炸半径 — 深度 1/2/3 级别会出问题的地方及置信度                    |
| `detect_changes` | Git-diff 影响 — 你当前的变更会影响什么                                    |
| `rename`         | 多文件协调重命名，附带带置信度标记的编辑项                                |
| `cypher`         | 原始图谱查询（先读取 `gitnexus://repo/{name}/schema`）                   |
| `list_repos`     | 发现已索引的仓库                                                          |

## 资源参考

轻量级读取（约 100-500 tokens）用于导航：

| 资源                                           | 内容                                   |
| ---------------------------------------------- | -------------------------------------- |
| `gitnexus://repo/{name}/context`               | 统计信息、过时检查                      |
| `gitnexus://repo/{name}/clusters`              | 所有功能区域及其内聚性得分              |
| `gitnexus://repo/{name}/cluster/{clusterName}` | 区域成员                               |
| `gitnexus://repo/{name}/processes`             | 所有执行流                             |
| `gitnexus://repo/{name}/process/{processName}` | 逐步执行轨迹                           |
| `gitnexus://repo/{name}/schema`                | 供 Cypher 使用的图谱模式                |

## 图谱模式

**节点：** File, Function, Class, Interface, Method, Community, Process
**边（通过 CodeRelation.type）：** CALLS, IMPORTS, EXTENDS, IMPLEMENTS, DEFINES, MEMBER_OF, STEP_IN_PROCESS

```cypher
MATCH (caller)-[:CodeRelation {type: 'CALLS'}]->(f:Function {name: "myFunc"})
RETURN caller.name, caller.filePath
```
```
```


---

## 5.gitnexus-impact-analysis下的skill
```markdown
---
name: gitnexus-impact-analysis
description: "当用户想知道修改某处会破坏什么，或在编辑代码前需要安全性分析时使用。示例：“修改 X 安全吗？”、“什么依赖于此？”、“会破坏什么？”"
---

# 使用 GitNexus 进行影响分析

## 何时使用

- “修改这个函数安全吗？”
- “如果我修改 X，什么会坏？”
- “展示爆炸半径”
- “谁使用了这段代码？”
- 在进行非平凡代码更改之前
- 提交之前 — 了解你的变更会影响什么

## 工作流

```
1. gitnexus_impact({target: "X", direction: "upstream"})  → 什么依赖于此
2. READ gitnexus://repo/{name}/processes                   → 检查受影响的执行流
3. gitnexus_detect_changes()                               → 将当前 git 变更映射到受影响的流程
4. 评估风险并向用户报告
```

> 如果提示“Index is stale” → 在终端中运行 `npx gitnexus analyze`。

## 检查清单

```
- [ ] 使用 gitnexus_impact({target, direction: "upstream"}) 查找依赖方
- [ ] 首先审查 d=1 的项（这些一定会破坏）
- [ ] 检查高置信度（>0.8）的依赖
- [ ] READ 流程以检查受影响的执行流
- [ ] 提交前检查使用 gitnexus_detect_changes()
- [ ] 评估风险级别并向用户报告
```

## 理解输出

| 深度   | 风险等级         | 含义                     |
| ------ | ---------------- | ------------------------ |
| d=1    | **一定会破坏**   | 直接的调用方/导入方      |
| d=2    | 很可能受影响     | 间接依赖                 |
| d=3    | 可能需要测试     | 传递效应                 |

## 风险评估

| 受影响范围                       | 风险     |
| -------------------------------- | -------- |
| <5 个符号，少量流程              | 低       |
| 5-15 个符号，2-5 个流程          | 中       |
| >15 个符号或多个流程             | 高       |
| 关键路径（认证、支付）           | 严重     |

## 工具

**gitnexus_impact** — 用于符号爆炸半径的主要工具：

```
gitnexus_impact({
  target: "validateUser",
  direction: "upstream",
  minConfidence: 0.8,
  maxDepth: 3
})

→ d=1（一定会破坏）：
  - loginHandler (src/auth/login.ts:42) [CALLS, 100%]
  - apiMiddleware (src/api/middleware.ts:15) [CALLS, 100%]

→ d=2（很可能受影响）：
  - authRouter (src/routes/auth.ts:22) [CALLS, 95%]
```

**gitnexus_detect_changes** — 基于 git-diff 的影响分析：

```
gitnexus_detect_changes({scope: "staged"})

→ 变更：3 个文件中的 5 个符号
→ 受影响流程：LoginFlow, TokenRefresh, APIMiddlewarePipeline
→ 风险：中
```

## 示例：“如果我修改 validateUser，什么会坏？”

```
1. gitnexus_impact({target: "validateUser", direction: "upstream"})
   → d=1: loginHandler, apiMiddleware（一定会破坏）
   → d=2: authRouter, sessionManager（很可能受影响）

2. READ gitnexus://repo/my-app/processes
   → LoginFlow 和 TokenRefresh 涉及 validateUser

3. 风险：2 个直接调用方，2 个流程 = 中
```
```

---
## 6.gitnexus-refactoring下的skill
```markdown
---
name: gitnexus-refactoring
description: "当用户想要安全地重命名、提取、拆分、移动或重构代码时使用。示例：“重命名这个函数”、“提取到模块”、“重构这个类”、“移动到单独的文件”"
---

# 使用 GitNexus 进行重构

## 何时使用

- “安全地重命名这个函数”
- “提取到模块”
- “拆分这个服务”
- “移动到新文件”
- 任何涉及重命名、提取、拆分或重组代码的任务

## 工作流

```
1. gitnexus_impact({target: "X", direction: "upstream"})  → 映射所有依赖方
2. gitnexus_query({query: "X"})                            → 查找涉及 X 的执行流
3. gitnexus_context({name: "X"})                           → 查看所有传入/传出引用
4. 计划更新顺序：接口 → 实现 → 调用方 → 测试
```

> 如果提示“Index is stale” → 在终端中运行 `npx gitnexus analyze`。

## 检查清单

### 重命名符号

```
- [ ] gitnexus_rename({symbol_name: "oldName", new_name: "newName", dry_run: true}) — 预览所有编辑
- [ ] 审查图谱编辑（高置信度）和 ast_search 编辑（仔细审查）
- [ ] 如果满意：gitnexus_rename({..., dry_run: false}) — 应用编辑
- [ ] gitnexus_detect_changes() — 验证只有预期的文件发生了变化
- [ ] 为受影响的流程运行测试
```

### 提取模块

```
- [ ] gitnexus_context({name: target}) — 查看所有传入/传出引用
- [ ] gitnexus_impact({target, direction: "upstream"}) — 查找所有外部调用方
- [ ] 定义新模块接口
- [ ] 提取代码，更新导入
- [ ] gitnexus_detect_changes() — 验证受影响的范围
- [ ] 为受影响的流程运行测试
```

### 拆分函数/服务

```
- [ ] gitnexus_context({name: target}) — 理解所有被调用方
- [ ] 按职责分组被调用方
- [ ] gitnexus_impact({target, direction: "upstream"}) — 映射需要更新的调用方
- [ ] 创建新函数/服务
- [ ] 更新调用方
- [ ] gitnexus_detect_changes() — 验证受影响的范围
- [ ] 为受影响的流程运行测试
```

## 工具

**gitnexus_rename** — 自动化的多文件重命名：

```
gitnexus_rename({symbol_name: "validateUser", new_name: "authenticateUser", dry_run: true})
→ 跨 8 个文件的 12 处编辑
→ 10 处图谱编辑（高置信度），2 处 ast_search 编辑（需审查）
→ 变更：[{file_path, edits: [{line, old_text, new_text, confidence}]}]
```

**gitnexus_impact** — 首先映射所有依赖方：

```
gitnexus_impact({target: "validateUser", direction: "upstream"})
→ d=1: loginHandler, apiMiddleware, testUtils
→ 受影响的流程: LoginFlow, TokenRefresh
```

**gitnexus_detect_changes** — 重构后验证你的变更：

```
gitnexus_detect_changes({scope: "all"})
→ 变更: 8 个文件，12 个符号
→ 受影响的流程: LoginFlow, TokenRefresh
→ 风险: 中
```

**gitnexus_cypher** — 自定义引用查询：

```cypher
MATCH (caller)-[:CodeRelation {type: 'CALLS'}]->(f:Function {name: "validateUser"})
RETURN caller.name, caller.filePath ORDER BY caller.filePath
```

## 风险规则

| 风险因素              | 缓解措施                                 |
| --------------------- | ---------------------------------------- |
| 调用方多（>5）        | 使用 gitnexus_rename 进行自动化更新      |
| 跨区域引用            | 事后使用 detect_changes 验证范围         |
| 字符串/动态引用       | 使用 gitnexus_query 查找它们             |
| 外部/公共 API         | 进行版本控制和适当的废弃处理             |

## 示例：将 `validateUser` 重命名为 `authenticateUser`

```
1. gitnexus_rename({symbol_name: "validateUser", new_name: "authenticateUser", dry_run: true})
   → 12 处编辑：10 处图谱（安全），2 处 ast_search（需审查）
   → 文件: validator.ts, login.ts, middleware.ts, config.json...

2. 审查 ast_search 编辑（config.json：动态引用！）

3. gitnexus_rename({symbol_name: "validateUser", new_name: "authenticateUser", dry_run: false})
   → 跨 8 个文件应用了 12 处编辑

4. gitnexus_detect_changes({scope: "all"})
   → 受影响: LoginFlow, TokenRefresh
   → 风险: 中 — 为这些流程运行测试
```
```