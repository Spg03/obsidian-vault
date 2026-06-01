---
type: implementation
status: codex-generated
tags:
  - gitnexus
  - impact
  - cross-repo
---

# Cross Repo Impact

跨仓库 impact 的核心是“两段式分析”：先在当前仓库做本地影响分析，再通过 bridge.lbug 找到跨仓库消费者，最后对消费者所在仓库继续分析并合并结果。

## 流程

```text
impact(repo: "@group", target: "loginHandler")
  -> local impact in provider repo
  -> query bridge.lbug
  -> find consumer contracts
  -> run impact in consumer repos
  -> merge risk and affected processes
```

## 分享时的例子

api-gateway 修改 `POST /login` 的返回字段，user-service 或 web-app 里消费这个接口的代码可能受到影响。Group 通过 ContractLink 把这些仓库连起来。

## 边界

跨仓库匹配依赖契约抽取质量和匹配策略。显式 manifest link 最可靠，自动匹配需要结合 exact、wildcard、文本或语义召回，并保留置信度。

## 相关笔记

- [[Bridge DB]]
- [[边界与取舍]]
