---
type: source-map
status: codex-generated
tags:
  - gitnexus
  - plugin
  - ide
---

# Plugin 与 IDE 集成系统

GitNexus 不只通过 CLI/MCP 暴露能力，也提供 Claude Plugin 和 Cursor Integration，把 MCP 配置、hooks、skills 打包到编辑器生态里。

## 源码入口

| 路径 | 作用 |
|---|---|
| `gitnexus-claude-plugin/.claude-plugin/plugin.json` | Claude plugin manifest |
| `gitnexus-claude-plugin/.mcp.json` | MCP 配置 |
| `gitnexus-claude-plugin/hooks/` | hooks 和锁处理 |
| `gitnexus-claude-plugin/skills/` | 标准 GitNexus skills |
| `gitnexus-cursor-integration/README.md` | Cursor 集成说明 |
| `gitnexus-cursor-integration/hooks/` | Cursor hooks |
| `gitnexus-cursor-integration/skills/` | Cursor skills |

## 集成内容

| 能力 | 作用 |
|---|---|
| MCP config | 让编辑器启动 `gitnexus mcp` |
| hooks | 在编辑器事件中调用 augment / lock 等辅助逻辑 |
| skills | 指导 Agent 如何探索、impact、debug、refactor |
| hook lock | 避免 hook 并发或 DB 锁冲突 |

## Skill 类型

标准 skills 包括 gitnexus-cli、gitnexus-debugging、gitnexus-exploring、gitnexus-guide、gitnexus-impact-analysis、gitnexus-pr-review、gitnexus-refactoring。它们把工具能力变成任务工作流，例如探索架构用 query/context/process resource，改符号前用 impact，提交前用 detect_changes，重命名用 gitnexus_rename。

## 与 ai-context 的关系

`gitnexus analyze` 可以安装/更新 AGENTS.md / CLAUDE.md 的 gitnexus block、标准 skills、基于 communities 的 generated skills。Plugin/IDE 集成则是这些能力的分发方式之一。

## 讲解抓手

> Plugin/IDE 集成层解决“如何让 Agent 真的按 GitNexus 工作流行动”。MCP 提供工具，skills 和 hooks 提供行为约束，二者缺一不可。
