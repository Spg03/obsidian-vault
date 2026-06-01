---
type: implementation
status: codex-generated
tags:
  - gitnexus
  - bridge-db
---

# Bridge DB

Bridge DB 是 Group 的跨仓库桥接库，常见文件名是 `bridge.lbug`。它保存 Contract 节点和 ContractLink 边，让跨仓库 impact 可以从一个仓库扇出到另一个仓库。

## 写入思路

```text
group sync
  -> read each repo LadybugDB
  -> extract contracts
  -> match contracts
  -> write bridge.lbug
```

## 为什么单独建桥接库

如果把所有仓库图谱直接合并，索引成本、命名冲突和权限边界都会变复杂。桥接库只保存跨仓库边界信息，既轻量，又能保持各仓库索引独立。

## 相关笔记

- [[Group 实现原理]]
- [[Cross Repo Impact]]
