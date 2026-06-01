---
type: source-map
status: codex-generated
tags:
  - gitnexus
  - cli
---

# CLI 命令全集源码导读

GitNexus CLI 是用户进入系统的第一层。源码入口是 `gitnexus/src/cli/index.ts`，基于 commander 注册命令，并通过 `createLazyAction` 懒加载具体实现，减少 `mcp` / `serve` 启动时的静态依赖开销。

## 源码入口

| 文件 | 职责 |
|---|---|
| `cli/index.ts` | commander 命令注册 |
| `cli/lazy-action.ts` | 动态 import 命令实现 |
| `cli/analyze.ts` | analyze 命令 |
| `cli/serve.ts` | HTTP server 命令 |
| `cli/mcp.ts` | MCP stdio 命令 |
| `cli/tool.ts` | query/context/impact/cypher/detect_changes 等 direct tool |
| `cli/group.ts` | group 子命令 |
| `cli/wiki.ts` | wiki 生成 |
| `cli/ai-context.ts`、`skill-gen.ts` | AI 上下文和 skill 生成 |

## 命令分类

| 分类 | 命令 | 说明 |
|---|---|---|
| 初始化 | `setup` | 配置 Cursor、Claude Code、OpenCode、Codex MCP |
| 索引 | `analyze [path]` | 完整索引仓库 |
| 注册 | `index [path...]` | 注册已有 `.gitnexus/` |
| 服务 | `serve` | 启动 HTTP API 和 Web UI |
| MCP | `mcp` | 启动 stdio MCP server |
| 状态 | `list`、`status`、`doctor` | repo 列表、当前状态、平台能力 |
| 清理 | `clean`、`remove` | 删除 index |
| 文档 | `wiki` | 生成知识库 wiki |
| hook | `augment` | 快速上下文增强 |
| 发布 | `publish` | 通知远端 registry |
| direct tools | `query`、`context`、`impact`、`cypher`、`detect-changes` | 不经过 MCP 直接调用 LocalBackend |
| eval | `eval-server` | SWE-bench 评测用轻量服务 |
| group | `group ...` | 跨仓库 group 管理 |

## analyze 的关键参数

| 参数 | 作用 |
|---|---|
| `--force` | 强制全量重建 |
| `--repair-fts` | 只修复 FTS，不重新分析 |
| `--embeddings [limit]` | 生成 embedding，可覆盖 50000 节点安全上限 |
| `--drop-embeddings` | 重建时丢弃旧 embedding |
| `--skills` | 基于 communities 生成 repo-specific skills |
| `--skip-agents-md` | 不更新 AGENTS.md/CLAUDE.md |
| `--skip-skills` | 不安装标准 skills |
| `--index-only` | 只建索引，不注入任何 AI 文件 |
| `--skip-git` | 当前 path 作为 repo root，不向上找 git root |
| `--name` | registry 中自定义仓库名 |
| `--workers` | worker pool 大小 |
| `--worker-timeout` | worker 子批次超时 |

## direct tools 为什么存在

`query/context/impact/cypher/detect-changes` 这些命令直接调用 `LocalBackend`，不需要启动 MCP server。适合 CI/eval 脚本、命令行调试、文档演示、MCP 客户端不可用时的 fallback。

## lazy action 的意义

`index.ts` 顶部注释提到全局 heap re-spawn 被移除，只有 analyze 需要大堆内存。命令实现通过懒加载避免 `gitnexus mcp` 启动时加载 analyze/tree-sitter/native 相关重依赖。

## 讲解抓手

> CLI 是 GitNexus 的产品门面，但不是业务核心。真正核心在 analyze 编排、Pipeline、LocalBackend。CLI 的设计重点是命令路由、懒加载、参数暴露和对不同使用场景的入口整合。
