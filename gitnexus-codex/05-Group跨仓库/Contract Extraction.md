---
type: implementation
status: codex-generated
tags:
  - gitnexus
  - contract
---

# Contract Extraction

Contract Extraction 的目标是从每个仓库里抽取“可被其他仓库消费的边界”。这些边界可以是 HTTP 路由、gRPC 服务、Thrift、Topic/Queue、include 依赖或 workspace dependency。

## 抽取来源

| 类型 | 示例 |
|---|---|
| HTTP Route | `POST /login` |
| gRPC Service | proto service / method |
| Thrift | service / method |
| Topic / Queue | publish / subscribe |
| Include / C | 头文件依赖 |
| Workspace Dep | package workspace 依赖 |
| Manifest Link | group.yaml 中人工声明的 link |

## 为什么需要契约

跨仓库调用很难靠单仓库 AST 直接看出来。Contract 是服务边界的共同语言：provider 暴露契约，consumer 消费契约，匹配成功后才能建立跨仓库边。

## 相关笔记

- [[Group 实现原理]]
- [[Bridge DB]]
