---
type: workflow
status: codex-expanded
tags:
  - gitnexus
  - agent-workflow
---

# Agent 工作流

GitNexus 对 Agent 最大的改变，不是“让它多一个搜索工具”，而是把编程任务拆成固定检查链路。

## 普通 Agent 的问题

普通 Agent 面对“修改 validateUser”时，经常这样走：

```text
grep validateUser
  -> 打开定义文件
  -> 读附近代码
  -> 改函数
  -> 跑测试或不跑
```

风险在于：

- 它可能漏掉间接调用方。
- 它不知道这个函数参与哪些执行流。
- 它不知道 API、工具、路由入口是否受影响。
- 它容易把“找到文本”误当成“理解影响面”。

## GitNexus 推荐工作流

```text
用户提出任务
  -> list_repos / repo context
  -> query 找相关流程
  -> context 看目标符号 360 度上下文
  -> impact 看上游影响面
  -> 高风险先报告
  -> 编辑代码
  -> detect_changes 验证实际影响
  -> 跑测试 / 类型检查
```

## 每一步的意义

| 步骤 | 工具 | 目的 |
|---|---|---|
| 找项目 | list_repos / resource | 多仓库时确认目标 repo |
| 找流程 | query | 不只是找文件，而是找执行流 |
| 看符号 | context | 看 incoming、outgoing、process participation |
| 看影响 | impact | 修改前知道 d=1 / d=2 / d=3 风险 |
| 修改后验证 | detect_changes | 把 git diff 映射回符号和流程 |

## 示例：validateUser

第一步，找认证流程：

```text
query({
  query: "authentication validation login user",
  goal: "find validateUser related execution flows"
})
```

第二步，看目标符号：

```text
context({
  name: "validateUser",
  kind: "Function"
})
```

你想拿到的信息不是源码本身，而是：

- 谁调用它。
- 它调用谁。
- 它参与哪些流程。
- 它是否是某个入口链路上的关键步骤。

第三步，修改前 impact：

```text
impact({
  target: "validateUser",
  direction: "upstream",
  maxDepth: 3
})
```

重点看 d=1，因为 d=1 是直接会破的调用方；d=2、d=3 是需要测试的传递影响。

第四步，改完验证：

```text
detect_changes({
  scope: "all"
})
```

如果 detect_changes 的影响范围和修改前 impact 预期不一致，说明修改可能波及了意料之外的流程。

## 工作流背后的思想

GitNexus 让 Agent 的行为从“文本优先”变成“关系优先”：

```text
先问图谱：哪些流程和符号相关？
再读源码：具体实现是什么？
再做影响分析：改了会影响谁？
最后修改并验证：实际 diff 是否符合预期？
```

## 分享时的对比图

左边：

```text
User -> Agent -> grep -> read nearby files -> edit
```

右边：

```text
User -> Agent -> query -> context -> impact -> edit -> detect_changes
```

这张图非常适合开场，因为它直接回答“为什么需要 GitNexus”。

## 相关笔记

- [[MCP 工具清单]]
- [[Prompt Skill AGENTS 注入机制]]
- [[边界与取舍]]
