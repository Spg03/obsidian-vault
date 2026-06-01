```markdown
# Tree-sitter 解析层：六层架构

想象你有一个超高性能的代码X光机，需要同时看懂16种编程语言。Tree-sitter解析层的任务就是：对每种语言加载对应的“语法镜头”，把源代码照一遍，提取出所有函数、类、方法、导入、调用关系。

---

## 第1层：语法加载 (`parser-loader.ts`)

一句话：用一张注册表告诉系统“如果遇到Python文件，请加载这种镜头”。

**SOURCES 注册表**:
- `JavaScript`   → `tree-sitter-javascript`        (必需)
- `TypeScript`   → `tree-sitter-typescript`        (必需)
- `Python`       → `tree-sitter-python`            (必需)
- `Java`         → `tree-sitter-java`              (必需)
- `C`            → `tree-sitter-c`                 (可选 — ABI可能不兼容)
- `Swift`        → `tree-sitter-swift`             (可选 — 可能没prebuild)
- `Dart`         → `tree-sitter-dart`              (可选)
- `Kotlin`       → `tree-sitter-kotlin`            (可选)
- ...共16种+

**核心设计**：
- 每种语法有一个 `GrammarSource`：`{ load: () => require('tree-sitter-python'), unavailableNote: '...', optional?: true }`
- `optional: true` 表示这个语法加载失败不抛异常，只是跳过该语言文件
- 加载结果缓存在 `loadCache` 中，失败信息只打一次日志（`logged` Set防刷屏）
- `resolveLanguageKey()` 处理 TypeScript 的 `.ts` vs `.tsx` 区分——它们是同一个包里导出的两个不同语法对象

---

## 第2层：安全解析 (`safe-parse.ts`)

一句话：Windows上tree-sitter有bug——字符串超过32767字符会直接崩溃（SIGSEGV），这个文件用“切香肠”的方式绕过它。

```typescript
// 小文件（≤16KB）：直接解析，性能最好
if (sourceText.length <= DIRECT_PARSE_LIMIT_CHARS) {
  return parser.parse(sourceText, oldTree, options);
}

// 大文件：用callback模式，每次只喂16KB
// tree-sitter内部通过多次回调获取完整的源码，绕过了V8字符串转换的bug
const input: Parser.Input = (index) => {
  if (index >= sourceText.length) return null;
  return sourceText.slice(index, index + SAFE_PARSE_CHUNK_CHARS);
};
return parser.parse(input, oldTree, options);
```

这个bug是tree-sitter 0.21.x的C++ binding在处理V8大字符串时内存越界，callback模式让binding走的是 `char*` 而不是 `v8::String`，从而规避。这是一个典型的平台兼容性补丁——用很小的性能代价（多几次函数调用）换取Windows上的稳定性。

---

## 第3层：查询语言 (`tree-sitter-queries.ts`)

一句话：tree-sitter把源码变成AST之后，我们需要用声明式查询告诉它“我要找什么”——就像SQL之于数据库，tree-sitter query就是AST的查询语言。

以TypeScript为例，每个查询模式都是 `(AST节点模式) @捕获名`：

```scheme
;; 找类定义：匹配 class_declaration，把名字节点捕获为 @name
(class_declaration
  name: (type_identifier) @name) @definition.class

;; 找函数声明
(function_declaration
  name: (identifier) @name) @definition.function

;; 找箭头函数赋值：const foo = () => {}
(lexical_declaration
  (variable_declarator
    name: (identifier) @name
    value: (arrow_function))) @definition.function

;; 找import语句
(import_statement
  source: (string (string_fragment) @import.source)) @import

;; 找方法调用：user.save()
(call_expression
  function: (member_expression
    object: (identifier) @call.receiver
    property: (property_identifier) @call.name)) @call
```

**关键设计**：
- 每种语言有独立的查询字符串（`TYPESCRIPT_QUERIES`, `PYTHON_QUERIES`, `JAVA_QUERIES` 等）
- 查询编译发生在 `processFileGroup()` 中：`new Parser.Query(lang, queryString)` — 只编译一次，对所有文件复用
- 捕获名（`@definition.class`, `@call.name`）是后续提取器识别符号类型的契约

---

## 第4层：语言提供者 (`language-provider.ts` + `languages/*.ts`)

一句话：这是整个解析体系的大脑——“策略模式”的集大成者。每种语言通过一个600+行的接口告诉系统它有什么特殊行为。

```typescript
// language-provider.ts 的核心接口（简化）
interface LanguageProvider {
  id: SupportedLanguages;
  extensions: string[];                    // ['.py']
  treeSitterQueries: string;               // 该语言的查询字符串
  typeConfig: TypeExtractionConfig;        // 类型提取规则
  exportChecker: ExportChecker;            // 如何判断一个符号是否导出
  importResolver: ImportResolver;          // 如何解析import路径
  namedBindingExtractor?: ...;             // from X import Y 这种命名绑定
  importSemantics: 'namespace' | 'wildcard' | ...;
  mroStrategy: 'c3' | 'linear' | ...;     // 多继承的方法解析顺序

  // 30+ 可选钩子——每个钩子都是语言特性的“差异点”
  callExtractor?: CallExtractor;           // 如何提取函数调用
  fieldExtractor?: FieldExtractor;         // 如何提取类的字段
  methodExtractor?: MethodExtractor;       // 如何提取类的方法
  heritageExtractor?: HeritageExtractor;   // 如何提取继承关系
  classExtractor?: ClassExtractor;         // 类声明/结构体/接口
  variableExtractor?: VariableExtractor;   // 变量声明
  preprocessSource?: (src, path) => string;// 源码预处理（如C++去掉UE宏）
  labelOverride?: (node, label) => NodeLabel | null; // 重写符号标签
  builtInNames: ReadonlySet<string>;       // 内置名称黑名单
  entryPointPatterns?: RegExp[];           // 入口点识别模式

  // RFC #909 作用域解析钩子（新一代，Python/TS/C#/JS已迁移）
  emitScopeCaptures?: ...;
  interpretImport?: ...;
  bindingScopeFor?: ...;
  mergeBindings?: ...;
  // ...更多
}
```

**Python Provider 示例**（`languages/python.ts`）：

```typescript
export const pythonProvider = defineLanguage({
  id: SupportedLanguages.Python,
  extensions: ['.py'],
  entryPointPatterns: [/^app$/, /^(get|post|put|delete|patch)_/i],
  astFrameworkPatterns: [
    { framework: 'fastapi', patterns: ['@app.get', '@app.post', ...] },
    { framework: 'flask', patterns: ['@app.route', ...] },
  ],
  treeSitterQueries: PYTHON_QUERIES,
  typeConfig: pythonConfig,
  exportChecker: pythonExportChecker,
  importResolver: createImportResolver(pythonImportConfig),
  importSemantics: 'namespace',          // Python是命名空间导入，不是通配符
  mroStrategy: 'c3',                     // Python使用C3线性化
  builtInNames: new Set(['print','len','range','str',...]),

  // RFC #909 新一代作用域解析
  emitScopeCaptures: emitPythonScopeCaptures,
  interpretImport: interpretPythonImport,
  bindingScopeFor: pythonBindingScopeFor,
  // ...
});
```

**TypeScript Provider 的亮点**（`languages/typescript.ts`）：
- `tsExtractFunctionName()` — 解决了箭头函数和HOC的命名问题：`const Button = forwardRef((p, r) => { ... })` 这种模式下的箭头函数被正确命名为 `Button` 而不是匿名
- 支持 Zustand stores、TanStack Query、React Context等现代React模式的函数命名

**注册表**（`languages/index.ts`）：
- `providers` 是一个 `Record<SupportedLanguages, LanguageProvider>`，用 `satisfies` 确保编译时穷尽检查
- `getProviderForFile()` 通过扩展名在 `extensionMap` 中O(1)查找——这在上一个会话中提到的“工厂流水线”里是关键的调度机制

---

## 第5层：Worker解析 (`parse-worker.ts`)

一句话：在主线程之外，用多个Worker线程并行解析文件，然后把结果序列化传回主线程。

这是整个系统最长的一个文件（2500+行），核心流程：

```
主线程 dispatch(files) → Worker收到 {type:'sub-batch', files:[...]}
  → processBatch(files):
      1. 按语言分组（减少setLanguage切换）
      2. 对每种语言：
         a. setLanguage(grammar)
         b. 编译查询 → new Parser.Query(lang, queryString)
         c. 对每个文件：
            - parseSourceSafe(parser, content) → AST
            - query.matches(rootNode) → 匹配列表
            - 遍历matches，提取：
              · import → ExtractedImport (preprocessImportPath)
              · heritage → ExtractedHeritage (继承 extends/implements)
              · call → ExtractedCall (调用关系 + 接收者类型推断)
              · assignment → ExtractedAssignment (字段写入)
              · definition → ParsedNode (定义：函数/类/方法/属性/变量)
              · decorator → ExtractedDecoratorRoute (装饰器路由)
              · route.fetch → ExtractedFetchCall (HTTP客户端调用)
              · express_route → 路由注册 (Express/Hono)
              · ORM queries → Prisma/Supabase检测
            - buildTypeEnv(tree) → 类型环境（用于接收者类型推断）
            - extractParsedFile() → RFC #909 新一代解析结果
      3. 返回 ParseWorkerResult
  → Worker合并到accumulated
  → 主线程收到flush → 获取最终result
```

**Worker内部的性能优化**：
- 4个O(1)缓存（per-file清空）：`classIdCache`, `functionIdCache`, `exportCache`, `fieldInfoCache` — 避免重复的父节点链遍历
- 文件按语言分组：`byLanguage` Map → 减少 `parser.setLanguage()` 调用
- 零拷贝传输：文件内容用 `Uint8Array` 通过ArrayBuffer传输，Worker端用 `TextDecoder` 解码
- 子批次流式处理：大仓库分多次 `sub-batch` 消息发送，Worker边收边处理，避免一次性传输所有文件内容撑爆内存

**类型推断（`buildTypeEnv`）**：
- 在解析阶段就进行per-file的类型推断，用于确定 `user.save()` 中 `user` 的类型是 `User`
- 这是调用解析的关键输入——后续的 `processCalls` 需要知道接收者类型才能把 `user.save()` 解析到 `User.save()`

---

## 第6层：解析编排 (`parsing-processor.ts` → `parse-impl.ts` → `parse.ts`)

一句话：这是我们在上一个会话中讲过的12阶段DAG的parse阶段的实现——负责“我应该用Worker池还是主线程串行解析？怎么分块？怎么缓存？”

```
parse.ts (Pipeline Phase)
  └─ runChunkedParseAndResolve() (parse-impl.ts)
       │
       ├─ 1. 过滤可解析文件 (isLanguageAvailable)
       ├─ 2. 按字母排序 (保证跨平台chunk稳定性)
       ├─ 3. 按字节预算分块 (默认2MB/chunk, GITNEXUS_CHUNK_BYTE_BUDGET可配)
       ├─ 4. 决定Worker vs 串行 (文件≥15 或 总字节≥512KB → Worker)
       ├─ 5. 逐块循环:
       │    ├─ 预取文件内容 (parseChunkConcurrency个chunk预取)
       │    ├─ 检查parse cache (chunk哈希 → 命中则跳过Worker)
       │    ├─ 命中: mergeChunkResults() 重放缓存
       │    └─ 未命中: processParsing() → dispatch到Worker/串行
       │         └─ 缓存结果到 parseCache (隔离期间跳过)
       │
       ├─ 6. 延迟提取 (所有chunk完成后统一处理):
       │    ├─ processImportsFromExtracted (70→75%)
       │    ├─ synthesizeWildcardImportBindings (Go/Ruby/C++)
       │    ├─ seedCrossFileReceiverTypes (跨文件接收者类型)
       │    ├─ processHeritageFromExtracted (75→80%)
       │    ├─ processRoutesFromExtracted (80→85%)
       │    └─ processCallsFromExtracted (85→95%)
       │
       └─ 7. 清理 & 返回 ParseOutput
```

**核心设计决策**（来自PR #1693的优化）：

1. **延迟提取**：在PR #1693之前，每个chunk做完Worker解析后立即在主线程做import/call/heritage提取——这导致Worker在等主线程，CPU利用率只有4-5%。改为全部延迟到最后统一处理后，Worker可以连续处理chunk，利用率大幅提升。
2. **Parse Cache**：基于chunk内容哈希的缓存。如果这次analyze的某chunk文件内容和上次完全一样，跳过Worker直接重放结果。缓存粒度由chunk大小决定——2MB预算下，改一个文件最多失效一个chunk。
3. **隔离机制**：如果某个文件导致Worker崩溃，WorkerPool会把它加入隔离名单。该chunk的缓存也会被跳过（U20.U2），确保下次analyze重新dispatch。
4. **进度条分段**：Parse阶段占20→70%，延迟提取占70→95%，确保进度条单调前进而不是卡住。

---

## 整体数据流图

```
源代码 (.py/.ts/.java/...)
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 第6层: parse-impl.ts (编排)                          │
│   分块 → 预取 → Worker调度 → 延迟提取                 │
└──────────────────────┬──────────────────────────────┘
                       │ dispatch(files)
                       ▼
┌─────────────────────────────────────────────────────┐
│ 第5层: parse-worker.ts (Worker线程)                  │
│   按语言分组 → setLanguage → parse → query.matches    │
│   → 遍历匹配 → 提取符号/导入/调用/继承/路由            │
│   → buildTypeEnv → 序列化结果传回主线程               │
└──────────────────────┬──────────────────────────────┘
                       │ 每个match
                       ▼
┌─────────────────────────────────────────────────────┐
│ 第3+4层: tree-sitter-queries + LanguageProvider      │
│   Query告诉tree-sitter "找什么"                       │
│   Provider告诉提取器 "怎么理解这个东西"                │
│   → 16种语言 × 各自的查询 + 各自的策略钩子             │
└──────────────────────┬──────────────────────────────┘
                       │ parser.parse()
                       ▼
┌─────────────────────────────────────────────────────┐
│ 第1+2层: parser-loader + safe-parse                  │
│   加载正确的语法 → 安全解析（Windows补丁）             │
│   → 产出具体AST                                       │
└─────────────────────────────────────────────────────┘
```

---

## 关键设计模式总结

| 模式 | 在哪 | 解决的问题 |
|------|------|------------|
| 注册表模式 | `SOURCES` 表, `providers` 表 | 加一种语言只需加一行，不写if/else |
| 策略模式 | `LanguageProvider` 30+钩子 | 每种语言的差异被隔离在各自的Provider里 |
| Worker池 | `worker-pool.ts` + `parse-worker.ts` | 多核并行解析，零拷贝传输 |
| 内容寻址缓存 | chunk哈希 → `parseCache` | 不改的代码不重解析 |
| 回调安全解析 | `Parser.Input` callback | 绕过Windows tree-sitter SIGSEGV bug |
| 延迟批处理 | deferred Worker arrays | Worker不等待主线程，CPU利用率从5%→充分利用 |

---

**之前没讲到的核心要点**：Tree-sitter解析并不只是“语法→AST”。GitNexus在解析阶段同步完成了符号定义、导入提取、调用关系、类型推断、路由检测、ORM检测——这些不是事后分析，而是在第一次遍历AST时就一把抓取的。Worker返回的不是裸AST，而是已经结构化好的 `ParseWorkerResult`（nodes + relationships + symbols + imports + calls + heritage + routes + ...），主线程只需要做跨文件的关联解析。