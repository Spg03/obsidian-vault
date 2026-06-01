---
type: example
status: codex-generated
tags:
  - gitnexus
  - agents
---

# AGENTS.md 示例

AGENTS.md 是项目级 Agent 规则文件。GitNexus analyze 可以把索引状态、工具使用要求、资源地址、技能入口写入其中。

## 它解决什么问题

Agent 经常忘记先看影响面，或者在长上下文里丢失项目规则。AGENTS.md 把这些规则落成稳定文件，让 Agent 每次进入项目时能重新加载。

## 典型规则

- 修改函数、类、方法前必须做 impact。
- 提交前必须 detect_changes。
- 高风险修改需要先告知用户。
- 探索陌生代码时优先 query/context，而不是盲目 grep。
- 重命名不要 find-and-replace，使用图谱辅助 rename。

## 相关笔记

- [[Prompt Skill AGENTS 注入机制]]
- [[Agent 工作流]]
