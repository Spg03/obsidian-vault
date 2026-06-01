---
type: implementation
status: codex-expanded
tags:
  - gitnexus
  - prompt-engineering
  - skill
---

# Prompt Skill AGENTS 注入机制

GitNexus 的一个关键思想是：有了图谱还不够，还要让 Agent 知道“什么时候查图谱、怎么查、查完做什么、什么时候不能直接改”。这就是 Prompt/Skill/AGENTS 注入机制。

源码入口：

```text
gitnexus/src/mcp/tools.ts
gitnexus/src/mcp/server.ts
gitnexus/src/cli/ai-context.ts
gitnexus/src/cli/skill-gen.ts
```

## 五层注入

| 层 | 文件 | 作用 |
|---|---|---|
| Tool description | `mcp/tools.ts` | 工具级提示，告诉 Agent WHEN TO USE / AFTER THIS |
| Next-step hint | `mcp/server.ts` | 工具结果后追加下一步动作 |
| AGENTS.md | `cli/ai-context.ts` | 项目级 MUST / NEVER / ALWAYS |
| CLAUDE.md | `cli/ai-context.ts` | Claude Code 专用上下文入口 |
| Skill | `cli/skill-gen.ts` 和 bundled skills | 任务级工作流 |

这五层不是重复，而是不同粒度的约束。

## Tool description：工具级 prompt

以 `query` 为例，description 会说明：

- 用于理解代码如何协同工作。
- 返回 process-grouped 结果。
- 查询后应该用 context 深入符号。
- group mode 和 service 参数如何使用。

这比只暴露一个函数名 `query` 有用得多。模型选工具时会读 description，因此 description 的写法直接影响工具调用质量。

## Next-step hint：结果级 prompt

`server.ts` 里的 `getNextStepHint` 会根据当前工具返回下一步建议。例如：

```text
query -> context
context -> impact
impact -> review d=1
rename -> detect_changes
```

这解决的是 Agent 常见问题：拿到一点结果后就停止，或者跳过关键检查。

## AGENTS.md：项目级规则

`ai-context.ts` 生成的 GitNexus block 会使用强约束词：

- MUST run impact before editing symbol。
- MUST run detect_changes before committing。
- MUST warn user on HIGH / CRITICAL risk。
- NEVER rename with find-and-replace。

这里的重点是：规则必须具体到工具和参数，不能写成“请谨慎修改代码”。模糊提示很容易被模型忽略。

## CLAUDE.md：生态适配

CLAUDE.md 和 AGENTS.md 的思想类似，但服务 Claude Code。GitNexus 同时生成两者，是为了适配不同 Agent 客户端的上下文加载机制。

## Skill：任务级 workflow

Skill 不是通用规则，而是“遇到某类任务时按这个流程做”。例如：

- exploring：先读 repo context，再 query，再 context，再 process。
- impact-analysis：修改前先 impact，按风险等级报告。
- refactoring：不要 find-and-replace，使用 rename 或图谱辅助。
- debugging：围绕错误入口追调用链。

社区 Skill 更进一步：它来自 KnowledgeGraph 的 community 聚类，把某个功能模块的相关文件、符号、术语写进专用 Skill。

## 这套设计的本质

GitNexus 不只是提供上下文，还在提供“上下文使用协议”。

```text
图谱能力
  -> MCP 工具
  -> 工具描述
  -> next-step hint
  -> AGENTS.md / CLAUDE.md
  -> Skill workflow
  -> Agent 行为
```

## 分享时的关键句

Prompt Engineering 在 GitNexus 里不是写一段万能提示词，而是分层注入：

- 工具级提示负责“什么时候调用”。
- 结果级提示负责“下一步做什么”。
- 项目级规则负责“哪些行为必须/禁止”。
- Skill 负责“某类任务的固定流程”。

## 边界

这些约束不能保证 Agent 100% 遵守。它们是行为引导，不是沙箱。真正安全仍需要：

- impact 分析。
- detect_changes。
- 测试和类型检查。
- 人工 review。

## 相关笔记

- [[工具层如何设计 Prompt]]

- [[AGENTS.md 示例]]
- [[Skills 示例]]
- [[MCP 工具清单]]
- [[Agent 工作流]]

