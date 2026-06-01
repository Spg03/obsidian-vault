---
type: source-map
status: codex-generated
tags:
  - gitnexus
  - eval
---

# Eval 框架与质量评估

`eval/` 目录是 GitNexus 的质量评估和实验框架，主要用于比较不同 Agent 模式、模型和工具接入方式在任务上的效果。

## 目录结构

| 路径 | 作用 |
|---|---|
| `eval/run_eval.py` | 评测主入口 |
| `eval/tool_registry.py` | 工具注册 |
| `eval/agents/gitnexus_agent.py` | GitNexus-aware agent |
| `eval/bridge/mcp_bridge.py` | MCP bridge |
| `eval/bridge/gitnexus_tools.sh` | GitNexus CLI 工具桥 |
| `eval/configs/models/*.yaml` | 模型配置 |
| `eval/configs/modes/*.yaml` | baseline/native/native_augment 模式 |
| `eval/prompts/*.jinja` | 评测 prompt 模板 |
| `eval/environments/gitnexus_docker.py` | Docker 环境 |
| `eval/analysis/analyze_results.py` | 结果分析 |
| `eval/tests/` | eval 框架测试 |

## 模式分类

| 模式 | 含义 |
|---|---|
| baseline | 普通 Agent，不注入 GitNexus 图谱能力 |
| native | 使用 GitNexus native tools |
| native_augment | 在 native 基础上加入 augment 上下文增强 |

这正好可以服务技术分享中的核心对比：普通 Agent vs GitNexus Agent。

## 为什么 eval 重要

GitNexus 的目标不是“图谱看起来很完整”，而是提升 Agent 编程效果。因此质量评估应该看工具是否被正确调用、修改前是否做 impact、是否减少盲改、是否更快定位相关代码、是否减少回归。

## 讲解定位

Eval 是工程保障层，不是主链路。如果时间有限，只需要说明：GitNexus 有独立 eval 框架，用 baseline/native/native_augment 比较图谱工具对 Agent 任务的影响。
