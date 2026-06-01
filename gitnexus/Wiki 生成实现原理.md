
```markdown
# GitNexus Wiki 生成系统

## “Wiki”是什么意思？

先说概念。在 GitNexus 里，**Wiki** 不是维基百科那个意思，而是用 AI 从知识图谱自动生成代码文档。你可以把它理解成：给代码库做了一次 CT 扫描（`analyze`），然后让 AI 医生（LLM）根据扫描结果写一份诊断报告。

最终产物是一个单文件 HTML（`index.html`），打开就是一个带侧边栏导航、Mermaid 架构图、可全文浏览的代码文档网站。离线可用，不需要服务器。

```bash
gitnexus analyze  →  知识图谱（CT 扫描结果）
gitnexus wiki     →  AI 文档（医生诊断报告）
```

---

## 整体流程图

![[gitnexus-wiki-generation-flowchart.png]]
---

## 第一层：CLI 入口 (`cli/wiki.ts`)

就是 `gitnexus wiki` 命令的实现。核心流程：

1. 检查 git repo + 索引是否存在
2. 解析 LLM 配置 (API key / model / provider)
   - 支持 OpenAI、Azure、OpenRouter、Cursor CLI、自定义端点
   - 首次运行有交互式引导（选提供商、输 key、选模型）
3. 创建 `WikiGenerator` → 调用 `generator.run()`
4. 生成完成后可选发布到 GitHub Gist

有个贴心的设计——**交互式引导**（cli/wiki.ts:231-396）。第一次运行时没有配置，会一步步问你：用哪个提供商？模型用哪个？API key 是什么？输完保存到 `~/.gitnexus/config.json`，下次就不问了。

---

## 第二层：WikiGenerator 核心 (`core/wiki/generator.ts`)

这是整个 Wiki 系统的大脑，`WikiGenerator` 类只有一个主入口 `run()`，但内部有两条路径：

### 全量生成 vs 增量更新

```
run()
  ├── 检查 meta.json 的 fromCommit
  │   ├── 跟当前 HEAD 一样 → "已是最新"（只重建 HTML viewer）
  │   └── 不一样 →
  │       ├── 旧 commit 可达（同一个分支）→ 增量更新
  │       └── 旧 commit 不可达（切了分支）→ 全量生成
  └── 全量生成
```

**增量更新**（`incrementalUpdate`，第 748 行）的机制：

```bash
git diff fromCommit..HEAD --name-only
  → 改动了哪些文件？
  → 查 moduleFiles 映射：这些文件属于哪些模块？
  → 只重新生成受影响的模块
  → 新增文件 >5 个 → 回退到全量生成（结构可能变了）
```

---

## 第三层：Phase 1——LLM 分组（模块划分）

这是最关键的一步，决定了文档的结构。

### 输入

给 LLM 喂两个东西：

```
文件列表:                   目录树:
- src/auth/login.ts:         src/
  login() (Function),        src/auth/
  validateToken (Function)   src/db/
- src/db/connection.ts:      src/api/
  connect (Function),        ...
  query (Function)
```

### LLM 调用

- **System**: `"你是一个文档架构师。把文件按功能分组成模块。每个模块代表一个内聚的特性/层次/领域。输出纯 JSON，不要解释。"`
- **User**: `[文件列表 + 目录树]`

LLM → `{"Authentication": ["src/auth/..."], "Database": ["src/db/..."]}`

### 为什么不让 LLM 翻译模块名？

`generator.ts:462-468` 有个重要的注释：**分组阶段不应用 `--lang` 参数**。因为 LLM 输出的 JSON key 是模块名，如果让 LLM 翻译，"Authentication" 可能变成 "认证"，这会破坏 slug 稳定性（URL 会变），而且 JSON 解析也可能出问题。标题的翻译交给后续的页面生成阶段。

### 安全网

LLM 可能出错，所以有层层防御：

1. JSON 解析失败 → 降级为按顶层目录分组（`fallbackGrouping`，第 558 行）
2. 有文件漏分配 → 自动归入 "Other" 模块
3. 模块太大 → Token 预算检查（默认 30K tokens），超大模块按子目录拆分（`splitBySubdirectory`，第 578 行）
4. **不可变快照** → 分组结果保存到 `first_module_tree.json`，下次生成时可以复用（不用再烧 LLM token）

---

## 第四层：Phase 2——模块页面生成

### 叶子模块（Leaf Module）

叶子模块是没有子模块的模块，直接读源码 + 查图谱，让 LLM 写文档。

**输入信息有五种：**

1. `SOURCE_CODE`：模块内所有文件的源码（拼接在一起）
2. `INTRA_CALLS`：模块内部调用关系（A函数 → B函数）
3. `OUTGOING_CALLS`：模块对外调用（本模块 → 其他模块）
4. `INCOMING_CALLS`：外部对模块的调用（其他模块 → 本模块）
5. `PROCESSES`：经过本模块的执行流（完整 trace）

这些数据都来自 LadybugDB 的 Cypher 查询（`graph-queries.ts`）：

```cypher
-- 模块内调用
MATCH (a)-[:CodeRelation {type: 'CALLS'}]->(b)
WHERE a.filePath IN [${fileList}] AND b.filePath IN [${fileList}]

-- 跨模块调用（出去的）
MATCH (a)-[:CodeRelation {type: 'CALLS'}]->(b)
WHERE a.filePath IN [${fileList}] AND NOT b.filePath IN [${fileList}]

-- 涉及的执行流
MATCH (s)-[:STEP_IN_PROCESS]->(p:Process)
WHERE s.filePath IN [${fileList}]
```

### Token 预算控制

源码可能很长，不能无脑全塞给 LLM。有两种节制：

1. `truncateSource`（第 942 行）——超过 `maxTokens` 就截断，加 `"... (truncated)"` 提示
2. `splitBySubdirectory`（第 578 行）——在分组阶段就把大模块拆成子模块

### 父模块（Parent Module）

父模块**不读源码**，而是读子模块已生成的文档摘要（前 800 字符或到 `### Architecture` 为止），再结合跨子模块的调用关系，让 LLM 写一个综合概述。

这样设计的目的是避免重复——你已经可以在子模块页面看细节了，父页面只需要告诉你“这几个子模块是怎么协作的”。

### 并发控制

叶子模块可以并行生成（互不依赖），父模块必须等子模块完成（需要读子模块的文档）。所以：

1. `flattenModuleTree()` 把树拆成 `leaves[]` 和 `parents[]`
2. `leaves` → `runParallel(leaves, concurrency=3)`
   - 自适应限流：遇到 429 → 并发数-1 → 等5秒重试
3. `parents` → 顺序执行（每个依赖子模块结果）

---

## 第五层：Phase 3——总览 + HTML 打包

### Overview 页面

汇总所有模块摘要 + 模块间调用边 + Top5 执行流，让 LLM 生成一个带 Mermaid 架构图的总览页面。

### HTML Viewer (`html-viewer.ts`)

这是一个自包含的单文件 HTML，非常巧妙：

`index.html` = 内嵌 CSS + 内嵌 JS + 内嵌 JSON 数据

所有 markdown 页面被打包成一个 JS 对象 `PAGES = { "overview": "...", "auth": "..." }`，侧边栏导航树是 `TREE = [...]`。点击导航时，前端用 `marked.js` 实时渲染 markdown，用 `mermaid.js` 渲染架构图。`.md` 链接被自动改写为 `#slug` hash 导航。

**好处**：零依赖、离线可用，一个 HTML 文件丢给别人就能看。

---

## 第六层：安全细节

### Mermaid 消毒器 (`mermaid-sanitizer.ts`)

LLM 输出的 Mermaid 图可能有问题——比如节点 ID 里有特殊字符（`/`, `.`, `:`），或者边标签里有不安全的括号。`sanitizeMermaidDiagram` 做了三件事：

1. 替换不合规节点 ID——如 `src/auth/login.ts` → `src_auth_login_ts`（并自动加引号标签保留原名）
2. 引号包裹不安全边标签——`|foo(bar)|` → `|"foo(bar)"|`
3. `\n` 转 `<br/>`——节点标签里的换行符

### Prompt 模板 (`prompts.ts`)

所有 LLM prompt 都有明确的规则约束：

- `MODULE_SYSTEM_PROMPT`：不许说“我写了...”这种废话，不许编造不存在的 API，Mermaid 图不能超过 10 个节点
- `OVERVIEW_SYSTEM_PROMPT`：不许 dump 原始数据，不许做模块索引表
- `PARENT_SYSTEM_PROMPT`：只写子模块间的关系，别重复子模块内容

### 输出语言控制（`--lang` 参数）

`buildSystemPrompt`（第 199 行）在 system prompt 末尾追加：

```
IMPORTANT: Write ALL documentation content in chinese.
This includes prose, code comments in examples, and diagram labels.
```

但页面的 H1 标题（`# Module Name`）**始终是英文**——因为标题决定了文件名（slug），改了 slug 就会断链接。

---

## 设计亮点总结

| 亮点 | 说明 |
|------|------|
| LLM 分组 + 人工审核 | `--review` 模式先生成模块树让用户确认/编辑，再生成页面 |
| 不可变快照 | `first_module_tree.json` 保存首次分组结果，后续生成可复用 |
| 增量更新 | 只重新生成受 git diff 影响的模块，而不是整个 wiki |
| 自包含 HTML | 一个文件带所有 CSS/JS/数据，离线可看 |
| Token 预算 | 模块拆分 + 截断 + 父模块不读源码，控制 LLM 成本 |
| 自适应限流 | 遇到 429 rate limit 自动降并发、等重试 |
| DB 心跳 | 长时间 LLM 调用时每 60 秒 touch 一下 LadybugDB，防止连接超时 |
| 降级安全网 | LLM 返回非法 JSON → 回退到按目录分组 |
