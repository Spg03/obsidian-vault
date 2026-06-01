---
type: source-map
status: codex-generated
tags:
  - gitnexus
  - cobol
---

# COBOL 子系统实现

COBOL 是 GitNexus 中比较特殊的语言路径。它不走普通 tree-sitter provider 模式，而是有独立预处理、COPY 展开和 JCL 解析。

## 源码入口

| 文件 | 作用 |
|---|---|
| `core/ingestion/cobol/cobol-preprocessor.ts` | COBOL 源码预处理 |
| `core/ingestion/cobol/cobol-copy-expander.ts` | COPY book 展开 |
| `core/ingestion/cobol/jcl-parser.ts` | JCL 解析 |
| `core/ingestion/cobol/jcl-processor.ts` | JCL 处理并写入图谱 |
| `pipeline-phases/cobol.ts` | Pipeline 阶段入口 |

## 为什么单独处理

COBOL/JCL 的工程结构和现代语言不同：COPY book 类似文本展开，不等同于普通 import；JCL 描述作业流和程序执行，不是普通函数调用；固定格式/列敏感/老式语法对通用 parser 不友好。因此 GitNexus 给它单独建子系统，而不是强行塞进 tree-sitter 主路径。

## Pipeline 位置

```text
scan -> structure -> [markdown, cobol] -> parse -> ...
```

COBOL 在 parse 之前运行，和 markdown 一样属于 standalone processor。

## 讲解定位

COBOL 不是主线重点，但在“工程边界能力”中很有价值：GitNexus 的管线不是只能接 tree-sitter 语言。通过 standalone phase，也可以接入 COBOL/JCL 这类需要特殊预处理和执行模型的语言。
