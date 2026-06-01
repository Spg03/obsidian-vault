/**

 * 分析命令

 *

 * 对仓库进行索引并将知识图谱存储在 .gitnexus/ 中

 *

 * 将核心分析委托给共享的 runFullAnalysis 编排器。

 * 此 CLI 包装器处理：堆内存管理、进度条、SIGINT 信号、

 * 技能生成 (--skills)、摘要输出和 process.exit()。

 */

  

import path from 'path';

import { execFileSync } from 'child_process';

import v8 from 'v8';

import cliProgress from 'cli-progress';

import { closeLbug } from '../core/lbug/lbug-adapter.js';

import { isWalCorruptionError, WAL_RECOVERY_SUGGESTION } from '../core/lbug/lbug-config.js';

import {

  getStoragePaths,

  getGlobalRegistryPath,

  RegistryNameCollisionError,

  AnalysisNotFinalizedError,

  assertAnalysisFinalized,

} from '../storage/repo-manager.js';

import { getGitRoot, hasGitDir } from '../storage/git.js';

import { runFullAnalysis } from '../core/run-analyze.js';

import { getMaxFileSizeBannerMessage } from '../core/ingestion/utils/max-file-size.js';

import { warnMissingOptionalGrammars } from './optional-grammars.js';

import { glob } from 'glob';

import fs from 'fs/promises';

import { cliError } from './cli-message.js';

import { isHfDownloadFailure } from '../core/embeddings/hf-env.js';

  

// 在模块加载时捕获 stderr.write，在任何组件（LadybugDB 原生初始化、

// 进度条、控制台重定向）能够对其进行 monkey-patch 之前。

// 即使 analyze 路径通过进度条的 bar.log() 重定向了 console.*，

// 下面的致命错误处理器也必须能够到达用户 —— 之前的行为会静默地

// 吞掉堆栈跟踪，使得在 Windows 上 #1169 与无操作成功无法区分。

const realStderrWrite = process.stderr.write.bind(process.stderr);

  

const writeFatalToStderr = (label: string, err: unknown): void => {

  const isErr = err instanceof Error;

  const message = isErr ? err.message : String(err);

  realStderrWrite(`\n  ${label}: ${message}\n`);

  if (isErr && err.stack) realStderrWrite(`${err.stack}\n`);

};

  

let fatalHandlersInstalled = false;

  

/**

 * 安装一次性的 `unhandledRejection` / `uncaughtException` 处理器，

 * 将失败信息输出到真实的 stderr（绕过进度条安装的任何控制台重定向）

 * 并强制非零退出码。没有这些处理器，从 {@link analyzeCommand} 的

 * try/catch 中逃逸的异步错误会被报告为退出码 0 且没有诊断信息 ——

 * 这是 #1169 中跟踪的静默失败模式。

 */

const installFatalHandlers = (): void => {

  if (fatalHandlersInstalled) return;

  fatalHandlersInstalled = true;

  process.on('unhandledRejection', (err) => {

    writeFatalToStderr('Analysis failed (unhandled rejection)', err);

    process.exit(1);

  });

  process.on('uncaughtException', (err) => {

    writeFatalToStderr('Analysis failed (uncaught exception)', err);

    process.exit(1);

  });

};

  

const HEAP_MB = 16384;

const TEST_RESPAWN_HEAP_MB = Number(process.env.GITNEXUS_TEST_RESPAWN_HEAP_MB);

const RESPAWN_HEAP_MB =

  Number.isFinite(TEST_RESPAWN_HEAP_MB) && TEST_RESPAWN_HEAP_MB > 0

    ? Math.floor(TEST_RESPAWN_HEAP_MB)

    : HEAP_MB;

const HEAP_FLAG = `--max-old-space-size=${RESPAWN_HEAP_MB}`;

/** 增加默认栈大小（KB）以防止深层类层次结构上的堆栈溢出。 */

const STACK_KB = 4096;

const STACK_FLAG = `--stack-size=${STACK_KB}`;

  

/**

 * 用于"子进程重新执行可能因 V8 OOM 而死亡"的启发式方法。

 *

 * 跨平台检测是尽力而为的：V8/Node 通常在 Linux/macOS/Windows 的

 * stderr/message 中发出稳定的堆耗尽短语（例如 "JavaScript heap out of memory"

 * 或 "Reached heap limit"），而某些环境只暴露状态/信号（例如 134/SIGABRT）。

 * 我们结合文本签名和进程退出签名。

 */

const childProcessLikelyOom = (err: unknown): boolean => {

  if (!err || typeof err !== 'object') return false;

  const e = err as {

    status?: unknown;

    signal?: unknown;

    stderr?: unknown;

    stdout?: unknown;

    message?: unknown;

  };

  

  const hasHeapOomSignature = (v: unknown): boolean => {

    const text = (

      Buffer.isBuffer(v) ? v.toString('utf8') : typeof v === 'string' ? v : ''

    ).toLowerCase();

    if (!text) return false;

    return (

      text.includes('javascript heap out of memory') ||

      text.includes('reached heap limit') ||

      text.includes('allocation failed - javascript heap out of memory') ||

      text.includes('fatalprocessoutofmemory')

    );

  };

  

  const fields = [e.message, e.stderr, e.stdout];

  if (fields.some((v) => hasHeapOomSignature(v))) return true;

  

  const hasAnyChildOutput = [e.stderr, e.stdout].some(

    (v) => (Buffer.isBuffer(v) && v.length > 0) || (typeof v === 'string' && v.length > 0),

  );

  if (hasAnyChildOutput) return false;

  

  return e.status === 134 || e.signal === 'SIGABRT';

};

  

const forceHeapOOMForTestIfEnabled = (): void => {

  if (process.env.GITNEXUS_TEST_FORCE_HEAP_OOM !== '1') return;

  // 分配 JS 字符串（而不是 Buffers），以便压力落在 V8 堆本身上。

  // Buffers 可以在堆外分配，这使得 OOM 触发不太可靠。

  const chunks: string[] = [];

  for (;;) chunks.push('x'.repeat(1024 * 1024));

};

  

/** 如果当前低于该值，则以 16GB 堆和更大的栈重新执行进程。 */

function ensureHeap(): boolean {

  const nodeOpts = process.env.NODE_OPTIONS || '';

  if (nodeOpts.includes('--max-old-space-size')) return false;

  

  const v8Heap = v8.getHeapStatistics().heap_size_limit;

  if (v8Heap >= HEAP_MB * 1024 * 1024 * 0.9) return false;

  

  // --stack-size 是 Node 24+ 中不允许在 NODE_OPTIONS 中使用的 V8 标志，

  // 因此仅将其作为直接 CLI 参数传递，而不是通过环境变量。

  const cliFlags = [HEAP_FLAG];

  if (!nodeOpts.includes('--stack-size')) cliFlags.push(STACK_FLAG);

  

  try {

    execFileSync(process.execPath, [...cliFlags, ...process.argv.slice(1)], {

      stdio: 'inherit',

      env: { ...process.env, NODE_OPTIONS: `${nodeOpts} ${HEAP_FLAG}`.trim() },

    });

  } catch (e: unknown) {

    if (childProcessLikelyOom(e)) {

      cliError(

        `  Analysis likely ran out of memory.\n` +

          `  Retry with a larger heap if your machine allows it:\n` +

          `    NODE_OPTIONS="--max-old-space-size=24576" gitnexus analyze [your-args]\n` +

          `    (Windows: set NODE_OPTIONS=--max-old-space-size=24576 && gitnexus analyze [your-args])\n` +

          `  If this persists, it may be a native crash unrelated to heap size.\n`,

        { recoveryHint: 'heap-oom-respawn' },

      );

    }

    const status =

      typeof e === 'object' && e !== null && 'status' in e && typeof e.status === 'number'

        ? e.status

        : 1;

    process.exitCode = status;

  }

  return true;

}

  

/**

 * `analyzeCommand` 写入用于向后兼容下游消费的 GITNEXUS_* 环境变量。

 * 在函数入口快照并在 finally 块中恢复，以便程序化调用者（测试、

 * 长时间运行的宿主）不会看到跨调用的泄漏状态。`GITNEXUS_WORKER_POOL_SIZE`

 * 不在这个列表中：该旋钮通过 `runFullAnalysis` 选项（参见 `workerPoolSize`

 * 管道）传递，因此 CLI 永远不需要为它而修改 `process.env`。

 */

const ANALYZE_CLI_ENV_KEYS = [

  'GITNEXUS_VERBOSE',

  'GITNEXUS_MAX_FILE_SIZE',

  'GITNEXUS_WORKER_SUB_BATCH_TIMEOUT_MS',

  'GITNEXUS_EMBEDDING_THREADS',

  'GITNEXUS_EMBEDDING_BATCH_SIZE',

  'GITNEXUS_EMBEDDING_SUB_BATCH_SIZE',

  'GITNEXUS_EMBEDDING_DEVICE',

] as const;

  

type AnalyzeEnvSnapshot = Record<(typeof ANALYZE_CLI_ENV_KEYS)[number], string | undefined>;

  

const snapshotAnalyzeEnv = (): AnalyzeEnvSnapshot => {

  const snap = {} as AnalyzeEnvSnapshot;

  for (const k of ANALYZE_CLI_ENV_KEYS) snap[k] = process.env[k];

  return snap;

};

  

const restoreAnalyzeEnv = (snap: AnalyzeEnvSnapshot): void => {

  for (const k of ANALYZE_CLI_ENV_KEYS) {

    const v = snap[k];

    if (v === undefined) delete process.env[k];

    else process.env[k] = v;

  }

};

  

export interface AnalyzeOptions {

  force?: boolean;

  repairFts?: boolean;

  /**

   * 嵌入生成开关。Commander 将 `--embeddings [limit]` 解析为：

   *   - 当标志被省略时为 `undefined`

   *   - 当不带参数传递时为 `true`（使用默认 50K 节点上限）

   *   - 当带参数传递时为字符串（`--embeddings 0` 完全禁用上限，

   *     `--embeddings <n>` 使用 `<n>` 作为上限）

   */

  embeddings?: boolean | string;

  /**

   * 在重建时显式删除现有嵌入，而不是保留它们。

   * 没有这个标志，即使 `--embeddings` 被省略，

   * 常规的 `analyze` 也会保留索引中已有的任何嵌入。

   */

  dropEmbeddings?: boolean;

  skills?: boolean;

  verbose?: boolean;

  /** 跳过 AGENTS.md 和 CLAUDE.md 的 gitnexus 块更新。 */

  skipAgentsMd?: boolean;

  /**

   * AGENTS.md 和 CLAUDE.md 中的统计信息包含。

   *

   * Commander.js 将 `--no-stats` 表示为 `stats: boolean`（默认为

   * `true`；当用户传递 `--no-stats` 时为 `false`），而不是

   * `noStats: boolean`。读取否定形式将始终是 `undefined`，

   * 因此该标志在修复之前对 markdown 重写路径是无操作的 (#1477)。

   * 想要知道"用户是否请求了 --no-stats?"的消费者应该与 `=== false`

   * 进行比较，以区分显式关闭情况和默认开启情况。

   */

  stats?: boolean;

  /** 跳过将标准 GitNexus 技能文件安装到 .claude/skills/gitnexus/。 */

  skipSkills?: boolean;

  /** 纯索引模式：跳过所有文件注入（AGENTS.md、CLAUDE.md、skills）。 */

  indexOnly?: boolean;

  /** 即使没有 .git 目录也索引该文件夹。 */

  skipGit?: boolean;

  /**

   * 用用户提供的别名覆盖默认的 basename 派生注册表 `name` (#829)。

   * 区分路径共享 basename 的仓库。持久化 —— 后续不带 `--name`

   * 的同一路径重新分析会保留别名。

   */

  name?: string;

  /**

   * 即使另一个路径已经使用相同的 `--name` 别名也允许注册 (#829)。

   * 有意与 `--force` 保持不同的标志，因为用户可能想要在相同名称下共存

   * 而不支付管道重新索引的成本。端到端映射到 registerRepo 的

   * `allowDuplicateName` 选项。

   */

  allowDuplicateName?: boolean;

  /**

   * 覆盖 walker 的大文件跳过阈值 (#991)。值以 KB 为单位；

   * 在下游被限制在 tree-sitter 32 MB 上限。为管道的其余部分

   * 设置 `GITNEXUS_MAX_FILE_SIZE`。

   */

  maxFileSize?: string;

  /** 以秒为单位覆盖 worker 子批次空闲超时。 */

  workerTimeout?: string;

  /** 解析 worker 池大小；0 禁用 worker（顺序回退）。 */

  workers?: string;

  embeddingThreads?: string;

  embeddingBatchSize?: string;

  embeddingSubBatchSize?: string;

  embeddingDevice?: string;

}

  

/**

 * 索引后技能步骤是否应该运行。

 *

 * 门控块按顺序做两件事：(1) 从 `--skills` 生成社区技能文件，

 * 以及 (2) 重新运行 `generateAIContextFiles` 以便 AGENTS.md/CLAUDE.md

 * 可以引用新写入的技能。两者一起被抑制 —— `--index-only` 丢弃整个步骤，

 * 而不仅仅是社区技能写入。名称保留用于测试契约；参见 `analyzeCommand`

 * 中的调用站点了解它还门控的 AGENTS.md/CLAUDE.md 重新生成。

 *

 * 保持为纯辅助函数，以便 `--index-only --skills` 契约可以在不启动

 * 完整分析管道的情况下进行单元测试 (#742 审查)。

 */

export const shouldGenerateCommunitySkillFiles = (

  options: Pick<AnalyzeOptions, 'skills' | 'indexOnly'> | undefined,

  pipelineResult: unknown,

): boolean => Boolean(options?.skills && pipelineResult && !options?.indexOnly);

  

export const analyzeCommand = async (inputPath?: string, options?: AnalyzeOptions) => {

  if (ensureHeap()) return;

  forceHeapOOMForTestIfEnabled();

  

  // 在重新执行解析后立即安装致命错误处理器，以便任何从下面 try/catch 中逃逸的

  // 异步错误 (#1169) 能够以堆栈跟踪和非零退出码的形式浮出水面，

  // 而不是静默的退出 0。

  installFatalHandlers();

  

  // 快照实现为下游消费写入的 GITNEXUS_* 环境变量，

  // 以便在程序化调用者（测试、长时间运行的宿主）中它们不会跨调用泄漏。

  // `process.exit(0)` 在成功路径上绕过 `finally` —— 这是有意为之：当进程正在退出时，

  // 恢复是无关紧要的。对于提前返回路径（验证错误）和 alreadyUpToDate 快速路径，

  // finally 会恢复调用前的值。

  const envSnap = snapshotAnalyzeEnv();

  try {

    await analyzeCommandImpl(inputPath, options);

  } finally {

    restoreAnalyzeEnv(envSnap);

  }

};

  

const analyzeCommandImpl = async (inputPath?: string, options?: AnalyzeOptions): Promise<void> => {

  if (options?.verbose) {

    process.env.GITNEXUS_VERBOSE = '1';

  }

  

  if (options?.maxFileSize) {

    process.env.GITNEXUS_MAX_FILE_SIZE = options.maxFileSize;

  }

  

  if (options?.workerTimeout) {

    const workerTimeoutSeconds = Number(options.workerTimeout);

    if (!Number.isFinite(workerTimeoutSeconds) || workerTimeoutSeconds < 1) {

      cliError('  --worker-timeout 必须至少为 1 秒。\n');

      process.exitCode = 1;

      return;

    }

    process.env.GITNEXUS_WORKER_SUB_BATCH_TIMEOUT_MS = String(

      Math.round(workerTimeoutSeconds * 1000),

    );

  }

  

  // `--workers` 通过 `runFullAnalysis` 选项 → PipelineOptions → createWorkerPool

  // 传递，有意绕过 GITNEXUS_WORKER_POOL_SIZE 环境通道，因此此 CLI 表面

  // 永远不会为池大小而修改 `process.env`。因此测试可以背靠背地使用不同的

  // --workers 值重新调用 analyzeCommand 并观察他们传递的值，而不是之前调用泄漏的值。

  let workerPoolSize: number | undefined;

  if (options?.workers !== undefined) {

    const parsedWorkers = Number(options.workers);

    if (!Number.isInteger(parsedWorkers) || parsedWorkers < 0) {

      cliError(

        '  --workers 必须是非负整数。' +

          '传递 0 以禁用 worker 池（顺序回退）。\n',

      );

      process.exitCode = 1;

      return;

    }

    workerPoolSize = parsedWorkers;

  }

  

  // 解析 `--embeddings [limit]`：`true` → 默认上限，字符串 → 数字上限

  // （0 完全禁用上限）。在此处验证，以便失败符合兄弟验证模式

  // （在 bar.start() 之前退出 —— 否则 process.exit() 会留下进度条的隐藏光标未清除）。

  let embeddingsNodeLimit: number | undefined;

  if (typeof options?.embeddings === 'string') {

    const parsed = Number(options.embeddings);

    if (!Number.isInteger(parsed) || parsed < 0) {

      cliError(

        `  --embeddings 期望非负整数（得到 "${options.embeddings}"）。` +

          `传递 0 以禁用安全上限，或省略值以保持默认。\n`,

      );

      process.exitCode = 1;

      return;

    }

    embeddingsNodeLimit = parsed;

  }

  const embeddingsEnabled = !!options?.embeddings;

  

  const setPositiveEnv = (

    optionName: string,

    envName: string,

    value: string | undefined,

  ): boolean => {

    if (value === undefined) return true;

    const parsed = Number(value);

    if (!Number.isInteger(parsed) || parsed <= 0) {

      cliError(`  ${optionName} 必须是正整数。\n`);

      process.exitCode = 1;

      return false;

    }

    process.env[envName] = String(parsed);

    return true;

  };

  

  if (

    !setPositiveEnv(

      '--embedding-threads',

      'GITNEXUS_EMBEDDING_THREADS',

      options?.embeddingThreads,

    ) ||

    !setPositiveEnv(

      '--embedding-batch-size',

      'GITNEXUS_EMBEDDING_BATCH_SIZE',

      options?.embeddingBatchSize,

    ) ||

    !setPositiveEnv(

      '--embedding-sub-batch-size',

      'GITNEXUS_EMBEDDING_SUB_BATCH_SIZE',

      options?.embeddingSubBatchSize,

    )

  ) {

    return;

  }

  

  if (options?.embeddingDevice) {

    const allowed = new Set(['auto', 'cpu', 'dml', 'cuda', 'wasm']);

    if (!allowed.has(options.embeddingDevice)) {

      cliError('  --embedding-device 必须是以下之一：auto, cpu, dml, cuda, wasm。\n');

      process.exitCode = 1;

      return;

    }

    process.env.GITNEXUS_EMBEDDING_DEVICE = options.embeddingDevice;

  }

  

  if (options?.repairFts && options?.force) {

    cliError(

      '  不能将 `--repair-fts` 与 `--force` 组合使用。' +

        '使用 `--repair-fts` 进行快速的仅 FTS 修复，或使用 `--force` 进行完全重建。\n',

    );

    process.exitCode = 1;

    return;

  }

  

  console.log('\n  GitNexus 分析器\n');

  

  // `--index-only` 是更强的契约 —— 它抑制所有形式的文件注入，

  // 包括 `--skills` 通常会产生的社区技能写入。

  // 显式显示覆盖，以便用户不会奇怪为什么管道重新索引运行了但没有出现技能文件。

  // 管道仍然重新运行（参见下面的 `force: options?.force || options?.skills`）；

  // 警告纯粹是关于丢弃的索引后写入步骤。

  if (options?.indexOnly && options?.skills) {

    console.log(

      '  注意：--index-only 覆盖 --skills；社区技能文件将不会被写入。\n',

    );

  }

  

  let repoPath: string;

  if (inputPath) {

    repoPath = path.resolve(inputPath);

  } else if (options?.skipGit) {

    // --skip-git：将 cwd 视为索引根目录，不要向上走到父 git 仓库。

    repoPath = path.resolve(process.cwd());

  } else {

    const gitRoot = getGitRoot(process.cwd());

    if (!gitRoot) {

      console.log(

        '  不在 git 仓库内。\n  提示：传递 --skip-git 以在没有 .git 目录的情况下索引任何文件夹。\n',

      );

      process.exitCode = 1;

      return;

    }

    repoPath = gitRoot;

  }

  

  const repoHasGit = hasGitDir(repoPath);

  if (!repoHasGit && !options?.skipGit) {

    console.log(

      '  不是 git 仓库。\n  提示：传递 --skip-git 以在没有 .git 目录的情况下索引任何文件夹。\n',

    );

    process.exitCode = 1;

    return;

  }

  if (!repoHasGit) {

    console.log(

      '  警告：未找到 .git 目录 — 提交跟踪和增量更新已禁用。\n',

    );

  }

  

  // 如果目标仓库包含可选语法会解析但原生绑定缺失的文件，

  // 在分析前警告，以便用户了解为什么这些文件最终未被解析，

  // 而不是默默地获得降级的索引。

  try {

    const matches = await glob(['**/*.dart', '**/*.proto'], {

      cwd: repoPath,

      ignore: ['**/node_modules/**', '**/.git/**', '**/dist/**', '**/build/**'],

      dot: false,

      nodir: true,

      absolute: false,

    });

    if (matches.length > 0) {

      const present = new Set<string>();

      for (const m of matches) {

        const ext = path.extname(m).toLowerCase();

        if (ext) present.add(ext);

      }

      warnMissingOptionalGrammars({ context: 'analyze', relevantExtensions: present });

    }

  } catch {

    // 尽力而为警告 —— 永远不要阻塞分析的预检。

  }

  

  // KuzuDB 迁移清理由 runFullAnalysis 内部处理。

  // 注意：--skills 在 runFullAnalysis 之后使用返回的 pipelineResult 处理。

  

  if (process.env.GITNEXUS_NO_GITIGNORE) {

    console.log(

      '  GITNEXUS_NO_GITIGNORE 已设置 — 跳过 .gitignore（仍然读取 .gitnexusignore）\n',

    );

  }

  

  const maxFileSizeBanner = getMaxFileSizeBannerMessage();

  if (maxFileSizeBanner) {

    console.log(`${maxFileSizeBanner}\n`);

  }

  

  // ── CLI 进度条设置 ─────────────────────────────────────────

  const bar = new cliProgress.SingleBar(

    {

      format: '  {bar} {percentage}% | {phase}',

      barCompleteChar: '\u2588',

      barIncompleteChar: '\u2591',

      hideCursor: true,

      barGlue: '',

      autopadding: true,

      clearOnComplete: false,

      stopOnComplete: false,

    },

    cliProgress.Presets.shades_grey,

  );

  

  bar.start(100, 0, { phase: 'Initializing...' });

  

  // 优雅的 SIGINT 处理。Pino 的默认目标是 `sync: false`（缓冲的）—

  // 在退出前刷新，以便正在处理的记录到达 stderr。

  // 参见 `gitnexus/src/core/logger.ts:flushLoggerSync`。

  let aborted = false;

  const sigintHandler = () => {

    if (aborted) process.exit(1);

    aborted = true;

    bar.stop();

    console.log('\n  已中断 — 正在清理...');

    closeLbug()

      .catch(() => {})

      .finally(async () => {

        const { flushLoggerSync } = await import('../core/logger.js');

        flushLoggerSync();

        process.exit(130);

      });

  };

  process.on('SIGINT', sigintHandler);

  

  // 通过 bar.log() 路由控制台输出，以防止进度条损坏。

  // 这是一种刻意的 UI 模式（不是日志记录问题）：analyze 在 stdout 上运行

  // 长时间存在的进度条；任何并发的 console.* 写入都会在中途渲染时覆盖该条。

  // 我们捕获原始值，在运行期间切换到 barLog，并在完成/错误/SIGINT 时恢复。

  const origLog = console.log.bind(console);

  // eslint-disable-next-line no-console -- 进度条 UX 的有意控制台路由

  const origWarn = console.warn.bind(console);

  // eslint-disable-next-line no-console -- 进度条 UX 的有意控制台路由

  const origError = console.error.bind(console);

  let barCurrentValue = 0;

  const barLog = (...args: any[]) => {

    process.stdout.write('\x1b[2K\r');

    origLog(args.map((a) => (typeof a === 'string' ? a : String(a))).join(' '));

    bar.update(barCurrentValue);

  };

  console.log = barLog;

  // eslint-disable-next-line no-console -- 进度条 UX 的有意控制台路由

  console.warn = barLog;

  // eslint-disable-next-line no-console -- 进度条 UX 的有意控制台路由

  console.error = barLog;

  

  // 跟踪每阶段的经过时间

  let lastPhaseLabel = 'Initializing...';

  let phaseStart = Date.now();

  

  const updateBar = (value: number, phaseLabel: string) => {

    barCurrentValue = value;

    if (phaseLabel !== lastPhaseLabel) {

      lastPhaseLabel = phaseLabel;

      phaseStart = Date.now();

    }

    const elapsed = Math.round((Date.now() - phaseStart) / 1000);

    const display = elapsed >= 3 ? `${phaseLabel} (${elapsed}s)` : phaseLabel;

    bar.update(value, { phase: display });

  };

  

  const elapsedTimer = setInterval(() => {

    const elapsed = Math.round((Date.now() - phaseStart) / 1000);

    if (elapsed >= 3) {

      bar.update({ phase: `${lastPhaseLabel} (${elapsed}s)` });

    }

  }, 1000);

  

  const t0 = Date.now();

  

  // ── 运行共享的分析编排器 ───────────────────────────────

  try {

    const skipAll = options?.indexOnly;

    const skipAgentsMd = skipAll || options?.skipAgentsMd;

    const skipSkills = skipAll || options?.skipSkills;

    const result = await runFullAnalysis(

      repoPath,

      {

        // 管道重新索引 — 与 --skills 进行或运算，因为技能生成

        // 需要新鲜的 pipelineResult。对注册表冲突保护没有影响

        // （参见下面的 allowDuplicateName）。

        force: options?.force || options?.skills,

        repairFts: options?.repairFts,

        embeddings: embeddingsEnabled,

        embeddingsNodeLimit,

        dropEmbeddings: options?.dropEmbeddings,

        verbose: options?.verbose,

        skipGit: options?.skipGit,

        skipAgentsMd,

        skipSkills,

        // commander.js `.option('--no-stats', …)` 将标志注册为

        // `options.stats`（布尔值，默认 `true`；当用户传递 --no-stats 时为 `false`），

        // 而不是 `noStats: boolean`。在这里读取否定形式将始终返回

        // `undefined`，因此该标志在修复之前对 markdown 重写路径是无操作的 (#1477)。

        // 想要知道"用户是否请求了 --no-stats?"的消费者应该与 `=== false`

        // 进行比较，以区分显式关闭情况和默认开启情况。

        noStats: options?.stats === false,

        registryName: options?.name,

        // 注册表冲突绕过 — 它自己的 CLI 标志，有意不超载 --force。

        // 遇到冲突保护的用户应该能够接受重复名称而不支付

        // 管道重新索引的成本。参见 #829 审查第二轮。

        allowDuplicateName: options?.allowDuplicateName,

        // Worker 池大小从 --workers 传递，替换之前的 GITNEXUS_WORKER_POOL_SIZE

        // 环境突变。`undefined` 推迟到管道内部的环境/自动公式回退。

        workerPoolSize,

      },

      {

        onProgress: (_phase, percent, message) => {

          updateBar(percent, message);

        },

        onLog: barLog,

      },

    );

  

    if (result.alreadyUpToDate) {

      // 即使快速路径也必须证明仓库是可发现的。先前的

      // 运行可以写入 meta.json 然后在 registerRepo() 之前失败；

      // 在该半完成状态下，runFullAnalysis 在下一次调用时返回 alreadyUpToDate，

      // 除非我们也在这里检查注册表。

      await assertAnalysisFinalized(repoPath);

      clearInterval(elapsedTimer);

      process.removeListener('SIGINT', sigintHandler);

      console.log = origLog;

      // eslint-disable-next-line no-console -- 有意进度条路由后的恢复

      console.warn = origWarn;

      // eslint-disable-next-line no-console -- 有意进度条路由后的恢复

      console.error = origError;

      bar.stop();

      console.log('  已经是最新状态\n');

      // 可以安全地返回而不调用 process.exit(0) —— runFullAnalysis 中的提前返回路径

      // 永远不会打开 LadybugDB，因此没有本机句柄阻止退出。

      return;

    }

  

    if (result.ftsRepairedOnly) {

      clearInterval(elapsedTimer);

      process.removeListener('SIGINT', sigintHandler);

      console.log = origLog;

      // eslint-disable-next-line no-console -- 有意进度条路由后的恢复

      console.warn = origWarn;

      // eslint-disable-next-line no-console -- 有意进度条路由后的恢复

      console.error = origError;

      bar.stop();

      console.log('  FTS 索引修复成功\n');

      return;

    }

  

    // 后完成不变量 (#1169)：runFullAnalysis 名义上写入 meta.json 并注册仓库，

    // 但在 Windows 上观察到它成功返回但两个工件都不存在（仅横幅输出，退出 0）。

    // 在声明成功之前验证两者，以便静默完成状态以非零退出码和可操作的错误

    // 浮出水面，而不是被误认为是健康的索引。

    await assertAnalysisFinalized(repoPath);

  

    // 技能生成（仅 CLI，使用来自分析的 pipelineResult）。

    // 门控以便 `--index-only --skills` 也跳过社区技能写入

    // （`shouldGenerateCommunitySkillFiles` — 参见单元测试）。

    if (shouldGenerateCommunitySkillFiles(options, result.pipelineResult)) {

      updateBar(99, '生成技能文件...');

      try {

        const { generateSkillFiles } = await import('./skill-gen.js');

        const { generateAIContextFiles } = await import('./ai-context.js');

        const skillResult = await generateSkillFiles(

          repoPath,

          result.repoName,

          result.pipelineResult,

        );

        if (skillResult.skills.length > 0) {

          barLog(`  生成了 ${skillResult.skills.length} 个技能文件`);

          // 现在我们有技能信息后重新生成 AI 上下文文件

          const s = result.stats;

          const communityResult = result.pipelineResult?.communityResult;

          let aggregatedClusterCount = 0;

          if (communityResult?.communities) {

            const groups = new Map<string, number>();

            for (const c of communityResult.communities) {

              const label = c.heuristicLabel || c.label || 'Unknown';

              groups.set(label, (groups.get(label) || 0) + c.symbolCount);

            }

            aggregatedClusterCount = Array.from(groups.values()).filter(

              (count: number) => count >= 5,

            ).length;

          }

          const { storagePath: sp } = getStoragePaths(repoPath);

          await generateAIContextFiles(

            repoPath,

            sp,

            result.repoName,

            {

              files: s.files ?? 0,

              nodes: s.nodes ?? 0,

              edges: s.edges ?? 0,

              communities: s.communities,

              clusters: aggregatedClusterCount,

              processes: s.processes,

            },

            skillResult.skills,

            {

              skipAgentsMd,

              skipSkills,

              // 镜像 runFullAnalysis `noStats` 桥接 (#1477) — 相同表达式；

              // 由 analyze-no-stats-bridge.test.ts 在 `--skills` 路径上测试。

              noStats: options?.stats === false,

            },

          );

        }

      } catch {

        /* 尽力而为 */

      }

    }

  

    const totalTime = ((Date.now() - t0) / 1000).toFixed(1);

  

    clearInterval(elapsedTimer);

    process.removeListener('SIGINT', sigintHandler);

  

    console.log = origLog;

    // eslint-disable-next-line no-console -- 有意进度条路由后的恢复

    console.warn = origWarn;

    // eslint-disable-next-line no-console -- 有意进度条路由后的恢复

    console.error = origError;

  

    bar.update(100, { phase: '完成' });

    bar.stop();

  

    // ── 摘要 ────────────────────────────────────────────────────

    const s = result.stats;

    console.log(`\n  仓库索引成功 (${totalTime}秒)\n`);

    console.log(

      `  ${(s.nodes ?? 0).toLocaleString()} 节点 | ${(s.edges ?? 0).toLocaleString()} 边 | ${s.communities ?? 0} 集群 | ${s.processes ?? 0} 流程`,

    );

    console.log(`  ${repoPath}`);

  

    try {

      await fs.access(getGlobalRegistryPath());

    } catch {

      console.log('\n  提示：运行 `gitnexus setup` 为您的编辑器配置 MCP。');

    }

  

    console.log('');

  } catch (err: unknown) {

    clearInterval(elapsedTimer);

    process.removeListener('SIGINT', sigintHandler);

    console.log = origLog;

    // eslint-disable-next-line no-console -- 有意进度条路由后的恢复

    console.warn = origWarn;

    // eslint-disable-next-line no-console -- 有意进度条路由后的恢复

    console.error = origError;

    bar.stop();

  

    const msg = err instanceof Error ? err.message : String(err);

  

    // 来自 --name 的注册表名称冲突 (#829) — 作为可操作的错误而不是通用堆栈跟踪显示。

    if (err instanceof RegistryNameCollisionError) {

      cliError(

        `\n  注册表名称冲突：\n` +

          `    "${err.registryName}" 已被 "${err.existingPath}" 使用。\n\n` +

          `  选项：\n` +

          `    • 选择不同的别名：  gitnexus analyze --name <alias>\n` +

          `    • 允许重复：       gitnexus analyze --allow-duplicate-name  (使 "-r ${err.registryName}" 有歧义)\n`,

        { registryName: err.registryName, existingPath: err.existingPath },

      );

      process.exitCode = 1;

      return;

    }

  

    // 完成不变量失败 (#1169) — 保持丰富的可操作消息完整，

    // 并通过 realStderrWrite 写入，以便它不会被慢终端上的残留栏刷新擦除。

    if (err instanceof AnalysisNotFinalizedError) {

      writeFatalToStderr('分析未完成', err);

      realStderrWrite(

        `\n  诊断清单：\n` +

          `    1. 重新运行 "gitnexus analyze" - 瞬态原生错误通常在重试时清除。\n` +

          `    2. 检查 ${err.storagePath} - 残留的 lbug.wal 表示写入中止。\n` +

          `    3. 如果失败持续，使用 NODE_OPTIONS="--max-old-space-size=8192 --trace-exit" 运行\n` +

          `       并将跟踪附加到 GitNexus 问题跟踪器。\n\n`,

      );

      process.exitCode = 1;

      return;

    }

  

    // WAL 损坏 — 索引文件无法读取。给出清晰的恢复路径，

    // 而不是令人困惑的堆栈跟踪（单独的原始错误消息已足够）。

    if (isWalCorruptionError(err) || msg.includes('LadybugDB WAL corruption')) {

      cliError(

        `  GitNexus 索引有损坏的 WAL 文件。\n` +

          `  这通常发生在之前的分析在写入中途被中断时。\n` +

          `  ${WAL_RECOVERY_SUGGESTION}\n`,

        { recoveryHint: 'wal-corruption' },

      );

      process.exitCode = 1;

      return;

    }

  

    // HF 下载失败 — 显示清晰的指导，而不显示原始堆栈跟踪。

    // 在 writeFatalToStderr 之前检查，以便用户看到一条聚焦的消息，

    // 而不是堆栈转储转储后跟第二个修复块。

    if (isHfDownloadFailure(msg) || msg.includes('Failed to download embedding model')) {

      cliError(

        `  无法下载嵌入模型。\n` +

          `  huggingface.co 可能无法从您的网络访问\n` +

          `  （例如在公司代理或区域防火墙后面）。\n` +

          `  建议：\n` +

          `    1. 设置 HF_ENDPOINT 为镜像并重试：\n` +

          `         HF_ENDPOINT=https://hf-mirror.com npx gitnexus analyze --embeddings\n` +

          `         (Windows: set HF_ENDPOINT=https://hf-mirror.com && npx gitnexus analyze --embeddings)\n` +

          `    2. 检查您的代理 / VPN 设置。\n` +

          `    3. 一旦下载完成，模型会被缓存 — 未来运行可以离线工作。\n`,

        { recoveryHint: 'hf-endpoint-unreachable' },

      );

      process.exitCode = 1;

      return;

    }

  

    // 绕过重定向的 console.error，将完整堆栈写入在模块加载时捕获的真实 stderr。

    // 重定向的 console.error 用 `\x1b[2K\r`（ANSI 清行）包装每一行，

    // 并随后强制 bar.update()，这在某些 Windows 终端上会视觉上擦除失败消息 ——

    // 这是 #1169 中静默退出症状的典型形状。

    writeFatalToStderr('分析失败', err);

  

    // 为已知失败模式提供有用的指导

    if (

      msg.includes('Maximum call stack size exceeded') ||

      msg.includes('call stack') ||

      msg.includes('Map maximum size') ||

      msg.includes('Invalid array length') ||

      msg.includes('Invalid string length') ||

      msg.includes('allocation failed') ||

      msg.includes('heap out of memory') ||

      msg.includes('JavaScript heap')

    ) {

      cliError(

        `  此错误通常发生在非常大的仓库上。\n` +

          `  建议：\n` +

          `    1. 将大型供应商/生成目录添加到 .gitnexusignore\n` +

          `    2. 增加 Node.js 堆：NODE_OPTIONS="--max-old-space-size=16384"\n` +

          `    3. 增加栈大小：NODE_OPTIONS="--stack-size=4096"\n`,

        { recoveryHint: 'large-repo' },

      );

    } else if (msg.includes('ERESOLVE') || msg.includes('Could not resolve dependency')) {

      // 注意：原始的 arborist "Cannot destructure property 'package' of

      // 'node.target'" 崩溃发生在 npm 内部，在 gitnexus 代码运行*之前*，

      // 因此无法在这里捕获。此分支处理在运行时表面化的依赖解析错误

      // （例如动态 require 失败）。

      cliError(

        `  这看起来像是 npm 依赖解析问题。\n` +

          `  建议：\n` +

          `    1. 清除 npm 缓存：    npm cache clean --force\n` +

          `    2. 更新 npm：        npm install -g npm@latest\n` +

          `    3. 重新安装 gitnexus： npm install -g gitnexus@latest\n` +

          `    4. 或直接尝试 npx：  npx gitnexus@latest analyze\n`,

        { recoveryHint: 'npm-resolution' },

      );

    } else if (

      msg.includes('MODULE_NOT_FOUND') ||

      msg.includes('Cannot find module') ||

      msg.includes('ERR_MODULE_NOT_FOUND')

    ) {

      cliError(

        `  无法加载必需的模块。安装可能已损坏。\n` +

          `  建议：\n` +

          `    1. 重新安装：  npm install -g gitnexus@latest\n` +

          `    2. 清除缓存： npm cache clean --force && npx gitnexus@latest analyze\n`,

        { recoveryHint: 'module-not-found' },

      );

    }

  

    process.exitCode = 1;

    return;

  }

  

  // LadybugDB 的原生模块持有阻止 Node 退出的打开句柄。

  // ONNX Runtime 还注册了在某些平台上会段错误的原生 atexit 钩子 (#38, #40)。

  // 强制退出以确保干净终止。

  process.exit(0);

};