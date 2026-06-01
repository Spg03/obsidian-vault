---
type: source-deep-dive
status: codex-generated
tags:
  - gitnexus
  - storage
  - staleness
---

# Storage Repo Manager 与 Git Staleness

Repo Manager 和 Git Staleness 负责“这个索引属于哪个仓库、是否过期、MCP 从哪里发现它”。这是 GitNexus 能被不同编辑器、不同 cwd、不同 worktree 使用的基础。

## 源码入口

| 文件 | 职责 |
|---|---|
| `gitnexus/src/storage/repo-manager.ts` | `.gitnexus` 存储路径、meta、全局 registry |
| `gitnexus/src/storage/git.ts` | git root、commit、remote、worktree identity |
| `gitnexus/src/core/git-staleness.ts` | index stale / sibling clone drift 检查 |
| `gitnexus/src/mcp/local/local-backend.ts` | list repos / repo resolution 消费这些能力 |

## 本地存储布局

```text
<repo>/.gitnexus/
  lbug
  lbug.wal
  lbug.lock
  meta.json
  parse-cache/
  wiki/
```

`getStoragePaths(repoPath)` 返回 storagePath、lbugPath、metaPath。全局 registry 位于 `~/.gitnexus/registry.json`，让 MCP server 从任意 cwd 启动时也能知道本机有哪些已索引仓库。

## RegistryEntry

| 字段 | 用途 |
|---|---|
| `name` | repo alias |
| `path` | repo path |
| `storagePath` | `.gitnexus` 路径 |
| `indexedAt` | 索引时间 |
| `lastCommit` | 索引时 commit |
| `remoteUrl` | sibling clone 检测 |
| `stats` | 展示统计 |

## canonicalizePath

`canonicalizePath` 不只是 `path.resolve`。它会尝试 `realpathSync.native`，处理 macOS `/var` 与 `/private/var` symlink 差异，以及 Windows 8.3 short-name 和 long-name 差异。路径不存在时 fallback 到 `path.resolve`。

## worktree identity

`git.ts` 中 `resolveRepoIdentityRoot` 用来避免 git worktree 被注册成不同项目：canonical checkout root 返回 canonical root；linked worktree root 返回 canonical repo root；普通子目录保留原 path，尊重 `--skip-git`。

## Staleness 检查

`checkStaleness(repoPath,lastCommit)` 调用 `git rev-list --count <lastCommit>..HEAD`。如果结果大于 0，说明索引落后 HEAD，会返回 `isStale=true` 和提示。

## sibling clone drift

`checkCwdMatch` 先 longest-prefix 找当前 cwd 是否在某个 registered repo path 下；如果不是，向上找 `.git`，读取 cwd 的 `origin` remote。如果 registry 里有相同 remoteUrl 但不同 path，认为是 sibling clone，再比较 cwd HEAD 和 indexed lastCommit，给出 drift 提示。

## 讲解抓手

> Repo Manager 解决“索引放在哪里、如何被发现”，Git Staleness 解决“当前工作区和索引是不是同一个代码状态”。这是 Agent 使用本地图谱时非常关键的新鲜度保护。
