以下是将 GitNexus Web UI 架构说明整理为标准 Markdown 格式的版本，保留了所有标题、流程图、组件树、表格和代码块，便于在 Obsidian 中直接使用。

```markdown
# GitNexus Web UI 架构

## “Web UI”是什么？

GitNexus 有两种使用方式：
- **CLI + MCP**：命令行 + AI Agent 集成（之前讲解的部分）
- **Web UI**：浏览器里打开一个可视化界面，能看图、搜代码、跟 AI 聊天

Web UI 不是替代 CLI，而是互补。CLI 适合 AI Agent 自动化使用，Web UI 适合人类浏览和理解代码结构。

---

## 整体架构：三层分离

```
![[gitnexus-web-ui-architecture.png]]
```

**技术栈**：React 19 + TypeScript + Vite + Tailwind CSS + Sigma.js，后端为 Express + LadybugDB。

---

## 前端架构：组件树

用一张图说明组件关系：

```
App.tsx (路由控制器)
│
├── viewMode === 'onboarding'
│   └── DropZone.tsx          ← 拖拽连接/输入URL
│       └── RepoLanding.tsx   ← 选择已有仓库或新建
│       └── RepoAnalyzer.tsx  ← 远程分析进度
│       └── OnboardingGuide   ← 新手指引
│
├── viewMode === 'loading'
│   └── LoadingOverlay.tsx    ← 进度条 (下载图谱中...)
│       └── AnalyzeProgress   ← 分析进度详情
│
└── viewMode === 'exploring'  ← 主界面
    ├── Header.tsx            ← 顶栏 (搜索框/仓库切换/设置)
    ├── FileTreePanel.tsx     ← 左侧文件树
    ├── GraphCanvas.tsx       ← 中间：知识图谱可视化 (Sigma.js)
    │   └── QueryFAB.tsx      ← 右下角悬浮搜索按钮
    ├── CodeReferencesPanel   ← 代码引用叠加层
    ├── RightPanel.tsx        ← 右侧面板
    │   ├── [Chat Tab]        ← AI 聊天
    │   │   ├── ToolCallCard  ← 工具调用卡片
    │   │   └── MarkdownRenderer ← Markdown 渲染
    │   └── [Processes Tab]   ← 执行流面板
    │       └── ProcessFlowModal ← 流程详情弹窗
    ├── SettingsPanel.tsx      ← 设置弹窗 (LLM 配置)
    └── StatusBar.tsx          ← 底部状态栏
```

---

## 核心系统一：状态管理 (`useAppState`)

整个应用的状态集中在一个巨大的 Context 里（`hooks/useAppState.tsx`）。这相当于应用的“大脑”，管理：

| 状态类别 | 内容 |
|----------|------|
| 视图 | `viewMode`: `onboarding` → `loading` → `exploring` |
| 图谱 | `graph`: KnowledgeGraph 实例（节点 + 关系） |
| 选中 | `selectedNode`: 当前点击的图谱节点 |
| AI 聊天 | `chatMessages`, `isChatLoading`, `agentError` |
| 代码引用 | `codeReferences`: AI 引用的代码片段 |
| 高亮 | AI 引用高亮、工具调用高亮、爆炸半径高亮 |
| 设置 | LLM 提供商、API key |
| 多仓库 | `availableRepos`, `currentRepo`, `switchRepo` |

---

## 核心系统二：知识图谱可视化 (`GraphCanvas` + `useSigma`)

这是整个 Web UI 最引人注目的部分——看到的那张花花绿绿的节点-连线图。

### 数据流

```
后端 /api/graph (NDJSON流)
  → KnowledgeGraph (纯数据, graph.ts)
  → graphology Graph (图论数据结构)
  → Sigma.js 渲染 (WebGL画布)
```

### 四步转换过程

**Step 1: 从后端拉数据**

`backend-client.ts` 的 `fetchGraph()` 支持两种模式：
- **NDJSON 流式**（`application/x-ndjson`）：一行一个 JSON，边下载边解析。大仓库不卡。
- **普通 JSON**：小仓库一次性返回。

**Step 2: KnowledgeGraph 数据结构**

`graph.ts` 里是一个非常简单的内存图：

```typescript
{ nodes: Map<id, GraphNode>, relationships: Map<id, GraphRelationship> }
```

这只是一个存储，没有图算法。

**Step 3: graphology 转换** (`lib/graph-adapter.ts`)

把 GitNexus 的图数据转成 graphology 库的 `Graph` 对象。graphology 是一个专业的图论库，提供：
- 图遍历、邻居查询
- 度数计算
- 社区检测（Leiden 算法，内置在 `vendor/leiden/`）

**Step 4: Sigma.js 渲染** (`hooks/useSigma.ts`)

Sigma.js 是一个用 WebGL 渲染大规模图的可视化库。`useSigma` hook 管理了整个渲染生命周期：

- **`sigmaRef` (Sigma 实例)**
  - **布局算法**: ForceAtlas2 (物理模拟)
    - 节点互相排斥 (repulsion)
    - 边像弹簧 (attraction)
    - 持续迭代直到“稳定”
    - 可以用 Play/Pause 按钮控制
  - **节点渲染**: 圆形 + 标签
    - 颜色按节点类型: Function=蓝, Class=绿, File=灰...
    - 大小按度数和类型: 连接越多的节点越大
  - **边渲染**: 曲线 (`EdgeCurveProgram`)
    - 灰色半透明, 避免遮挡节点
  - **交互**:
    - 缩放/平移 (鼠标滚轮+拖拽)
    - 点击节点 → 选中 → 右侧面板显示详情
    - 悬停 → 高亮邻居
  - **AI 高亮**:
    - AI 引用的节点 → 脉冲动画 (pulse/ripple/glow)
    - 爆炸半径节点 → 红色高亮

---

## 核心系统三：浏览器里的 AI Agent (`core/llm/agent.ts`)

这是最特别的设计：**AI Agent 不是跑在服务器上，而是跑在浏览器里。**

### 为什么？

因为 LLM API key 是用户自己的，不应该经过 GitNexus 服务器。浏览器直接调 OpenAI/Anthropic/Gemini 的 API。

### 架构

```
RightPanel (聊天UI)
    │
    ▼
LangChain Agent (createReactAgent)
    │
    ├── Chat Model (OpenAI / Anthropic / Gemini / Ollama / OpenRouter...)
    │     │
    │     └── LLM API (直接浏览器 fetch)
    │
    └── 7 Tools (tools.ts)
          ├── search  → backend-client.search()  → /api/search
          ├── cypher  → backend-client.runQuery() → /api/query
          ├── grep    → backend-client.grep()     → /api/grep
          ├── read    → backend-client.readFile() → /api/file
          ├── overview → 从 graph 缓存构造
          ├── explore → 组合查询
          └── impact  → 组合查询
```

### Agent 的系统提示

Agent 的名字叫 **Nexus**，系统提示词（`agent.ts:51`）要求它：
1. **必须引用来源**——每个事实都要 `[[file:line]]` 标注
2. **必须验证**——用 Cypher 检查结果完整性
3. **五步工作流**：搜索 → 阅读 → 追溯 → 引用 → 验证

### 动态上下文注入 (`context-builder.ts`)

每次 Agent 启动时，从图谱自动提取代码库的“快照”注入到 system prompt：
- **热门节点 (hotspots)**: 连接数最多的函数/类
- **项目统计**: 文件数、函数数、类数...
- **文件夹树**: 目录结构概览

这样 Agent 不需要从零探索，一开始就有全局视野。

### 前端如何展示 Agent 的工具调用

当 Agent 调用工具时（如 `search("auth")`），前端会显示一个 **ToolCallCard** 组件——折叠卡片，展示工具名、参数和返回结果。这让用户能看到 Agent“在想什么”。

---

## 核心系统四：后端 HTTP API (`server/api.ts`)

Express 服务器提供了 20+ 个 REST 端点：

| 端点 | 方法 | 用途 |
|------|------|------|
| `/api/health` | GET | 健康检查 |
| `/api/heartbeat` | GET | SSE 心跳（前端断线检测） |
| `/api/info` | GET | 服务器版本信息 |
| `/api/repos` | GET | 列出已索引仓库 |
| `/api/repo` | GET/DELETE | 获取/删除仓库元数据 |
| `/api/graph` | GET | **核心**：返回完整知识图谱（支持 NDJSON 流） |
| `/api/query` | POST | 执行 Cypher 查询 |
| `/api/search` | POST | 混合搜索（BM25+语义） |
| `/api/grep` | GET | 正则搜索文件内容 |
| `/api/file` | GET | 读取文件内容 |
| `/api/processes` | GET | 列出所有执行流 |
| `/api/process` | GET | 单个执行流详情 |
| `/api/clusters` | GET | 列出所有功能区域 |
| `/api/cluster` | GET | 单个功能区域详情 |
| `/api/analyze` | POST | 启动远程分析任务 |
| `/api/analyze/:jobId` | GET/DELETE | 查询/取消分析任务 |
| `/api/analyze/:jobId/progress` | SSE | 流式分析进度 |
| `/api/embed` | POST | 启动向量嵌入生成 |
| `/api/embed/:jobId` | GET/DELETE | 查询/取消嵌入任务 |
| `/api/mcp` | ALL | MCP over HTTP（上一讲的内容） |

### 连接流程

从用户打开 Web UI 到看到图谱的完整流程：

1. 用户输入服务器地址（或本地 `localhost:4747`）
2. `GET /api/repo` → 验证仓库存在，获取元数据
3. `GET /api/graph?stream=true` → NDJSON 流式下载图谱
   - 每行 `{"type":"node","data":{...}}`
   - 每行 `{"type":"relationship","data":{...}}`
4. 前端构建 `KnowledgeGraph` → `graphology Graph` → Sigma.js 渲染
5. ForceAtlas2 布局开始，节点自动排布
6. 用户看到图谱 → 进入 `exploring` 模式

### 安全设计

- **CORS 白名单**：只允许 localhost、127.0.0.1、局域网 (RFC 1918)、`gitnexus.vercel.app`
- **限流**：`/api/file`、`/api/grep`、`/api/analyze` 有频率限制
- **URL 验证**：`validateBackendUrl` 拒绝 `javascript:`、`file:`、`data:` 等危险协议

---

## 核心系统五：SSE 心跳与断线重连

![[gitnexus-heartbeat-reconnect-sequence.png]]

与 MCP 的硬错误不同，Web UI 的断线处理是**柔性降级**：只显示一条横幅，不重置整个界面。服务器回来后自动恢复。

---

## 设计亮点总结

| 亮点 | 说明 |
|------|------|
| 浏览器端 AI Agent | API key 不经过服务器，直连 LLM 提供商 |
| NDJSON 流式图谱 | 大仓库不阻塞，边下载边解析 |
| ForceAtlas2 物理布局 | 节点自动排布出清晰的聚类结构 |
| Agent 可视化 | `ToolCallCard` 展示 AI 的思考过程 |
| AI 高亮系统 | 引用节点脉冲动画 + 爆炸半径红色标记 |
| 柔性断线 | SSE 心跳 + 指数退避重连 + 不丢状态 |
| 多仓库热切换 | Header 下拉切换仓库，图谱+AI 上下文同步更新 |
| URL 可收藏 | `?server=` + `?project=` 参数，刷新不丢状态 |
| 远程分析 | Web UI 可以触发远程服务器的分析任务，SSE 流式进度 |
