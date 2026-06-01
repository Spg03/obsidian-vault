---
type: reference
status: codex-generated
tags:
  - gitnexus
  - mcp
  - tools
---

# MCP 工具清单

工具数量会随版本变化，分享时不要死记“几个工具”，而要按能力分组讲。

## 仓库发现

- `list_repos`：列出已索引仓库。

## 理解代码

- `query`：按概念搜索执行流和相关符号。
- `context`：查看单个符号的调用方、被调用方、参与流程。
- `cypher`：直接查询图谱，适合高级分析。

## 修改前安全分析

- `impact`：看修改某个符号会影响谁。
- `detect_changes`：提交前根据 git diff 映射受影响符号和流程。
- `rename`：基于图谱和文本搜索做多文件重命名预览或执行。

## API 与工具边界

- `route_map`：看 API 路由和前端消费者。
- `shape_check`：检查响应字段和消费字段是否漂移。
- `api_impact`：修改 API handler 前看消费者和风险。
- `tool_map`：查看 MCP/RPC 工具定义和处理位置。

## 讲法

这些工具不是简单命令集合，而是 Agent 工作流的步骤：

```text
query -> context -> impact -> edit -> detect_changes
```

## 相关笔记

- [[Agent 工作流]]
- [[Prompt Skill AGENTS 注入机制]]
