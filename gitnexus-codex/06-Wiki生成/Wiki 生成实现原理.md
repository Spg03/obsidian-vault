---
type: implementation
status: codex-generated
tags:
  - gitnexus
  - wiki
---

# Wiki 生成实现原理

Wiki 生成是 GitNexus 把图谱和源码进一步转成面向人的文档。它不是简单遍历文件夹，而是先分模块，再逐模块生成页面，最后生成总览和 HTML viewer。

## 流程图

![[gitnexus-wiki-generation-flowchart.png]]

## 四个阶段

| 阶段 | 做什么 |
|---|---|
| Phase 0 收集材料 | 从 LadybugDB 读取源文件、导出符号、文件列表 |
| Phase 1 LLM 分组 | 按功能把文件分成模块树 |
| Phase 2 生成页面 | 叶子模块并行生成，父模块汇总子模块 |
| Phase 3 总览打包 | 生成 overview.md、module_tree.json、index.html |

## 为什么适合技术分享

Wiki 生成展示了 GitNexus 的另一个价值：图谱不仅服务 Agent 修改代码，也能服务人类理解项目结构。

## 相关笔记

- [[GitNexus 索引构建实现原理]]
- [[Prompt Skill AGENTS 注入机制]]
