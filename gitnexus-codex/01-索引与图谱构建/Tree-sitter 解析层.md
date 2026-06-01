---
type: implementation
status: codex-generated
tags:
  - gitnexus
  - tree-sitter
---

# Tree-sitter 解析层

Tree-sitter 是 GitNexus 从“代码文本”进入“结构化 AST”的入口。parse 阶段通过各语言 provider 把 AST 转成统一的符号、导入、调用和引用模型。

## 它解决什么问题

不同语言的语法差异很大，但 GitNexus 的上层图谱希望看到统一结构：

- Function
- Class
- Method
- Interface
- Import
- Call
- Reference

Tree-sitter 负责把语法树提供出来，GitNexus 的语言适配层负责把语言细节翻译成统一语义。

## 旧调用解析 DAG

旧路径可以概括为六步：

1. 提取调用。
2. 分类调用形式。
3. 推断接收者。
4. 选择分发策略。
5. 解析目标。
6. 发出 CALLS 边。

语言特有行为通过 `LanguageProvider` 钩子进入，例如隐式 receiver 推断和 dispatch 策略选择。

## 新 scope-resolution 管线

新路径先建立全局作用域索引，再解析引用：

```text
ParsedFile[]
  -> ScopeResolutionIndexes
  -> ReferenceIndex
  -> KnowledgeGraph edges
```

这个思路更像编译器前端：先建 symbol table，再做引用绑定。

## 分享时的重点

Tree-sitter 只给 AST，不直接给工程语义。GitNexus 的价值在于把 AST 提升成跨文件、跨语言、可查询的工程关系。

## 相关笔记

- [[Pipeline DAG 实现]]
- [[图谱 Schema 速览]]
