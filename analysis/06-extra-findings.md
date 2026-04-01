# 第十三章：深度发现与边界案例分析

[返回总目录](../README.md)

---

## 1. 导读

本章记录三类在常规文档中被忽略的深层机制：

1. **Trust 边界的程序化处理**：配置文件如何被分级信任，以及如何防止"配置文件本身是攻击面"
2. **Swarm 全局状态桥**：多 agent 架构中的上下文同步问题
3. **隐私耦合的主动切断设计**：分析系统如何从架构层面防止 PII 意外泄漏

---

## 2. Trust 边界的程序化处理

### 2.1 CLAUDE.md 的分级信任模型

**文件**：[`src/utils/claudemd.ts`](../src/utils/claudemd.ts)

CLAUDE.md 并不是一个扁平系统——它有四个信任层级，各自拥有不同的 scope 和 mount 优先级：

| 类型 | 路径 | 信任级别 |
|------|------|----------|
| `Managed` | `/etc/claude-code/CLAUDE.md` | 最高（系统管理员） |
| `User` | `~/.claude/CLAUDE.md` | 高（用户全局） |
| `Project` | `{cwd}/CLAUDE.md`, `{cwd}/.claude/CLAUDE.md` | 中（项目约定） |
| `Local` | `.claude/rules/*.md` | 最低（本地约定） |

这个分级的实现由 `getClaudeMds()` 并行加载所有层级后合并，优先级体现在系统 prompt 的拼接顺序上（高信任优先）。

### 2.2 `@include` 的深度限制

`CLAUDE.md` 支持 `@include <path>` 引入外部文件，但有硬上限保护：

```typescript
// src/utils/claudemd.ts:537
const MAX_INCLUDE_DEPTH = 5

// src/utils/claudemd.ts:620
export async function processMemoryFile(
  filePath: string,
  type: MemoryType,
  processedPaths: Set<string>,
  includeExternal: boolean,
  depth: number = 0,
): Promise<MemoryFileInfo[]> {
  // 去重：跳过已处理过的路径（防循环引用）
  // 深度限制：超过 5 层即截断
  const normalizedPath = normalizePathForComparison(filePath)
  if (processedPaths.has(normalizedPath) || depth >= MAX_INCLUDE_DEPTH) {
    return []  // 静默截断，不报错
  }

  // 排除列表：claudeMdExcludes 设置可显式排除特定路径
  if (isClaudeMdExcluded(filePath, type)) {
    return []
  }

  // 解析 symlink（用于路径去重，防止 /tmp -> /private/tmp 的重复加载）
  const { resolvedPath, isSymlink } = safeResolvePath(getFsImplementation(), filePath)
  processedPaths.add(normalizedPath)
  if (isSymlink) {
    processedPaths.add(normalizePathForComparison(resolvedPath))
  }
  // ...
}
```

`MAX_INCLUDE_DEPTH = 5` 是防御性设计：阻止恶意 CLAUDE.md 通过嵌套 `@include` 构造无限递归导致进程崩溃（原始的 DoS 缓解）。

### 2.3 Trust 建立时序：为什么 Telemetry 初始化在 Trust 之后

**文件**：[`src/entrypoints/init.ts`](../src/entrypoints/init.ts)，[`src/main.tsx`](../src/main.tsx)

这是一个容易忽略的安全细节：

```typescript
// src/main.tsx（伪代码骨架）
export async function main(argv) {
  await init(argv)   // ① Trust 前：只应用安全 env var，不发 telemetry 事件

  // ... 建立 trust（用户确认、检查配置文件 includes）...

  await initializeTelemetryAfterTrust()  // ② Trust 后：才允许完整 env var 和 telemetry
}
```

```typescript
// src/entrypoints/init.ts
export async function init(argv) {
  applySafeEnvironmentVariables()    // 只应用白名单内的 env var
  initTelemetrySkeleton()            // 只注册 sink，不发事件
  // 不调用 attachAnalyticsSink()
}

export async function initializeTelemetryAfterTrust() {
  applyFullEnvironmentVariables()    // Trust 通过后才应用全部 env var
  attachAnalyticsSink()              // 开始处理队列中的 telemetry 事件
}
```

**设计逻辑**：如果 CLAUDE.md 本身（或它引用的外部文件）是攻击面，那么在 trust 建立之前的 env var 就可能被恶意注入。通过推迟应用完整 env var 到 trust 建立之后，系统极大缩小了"配置文件作为攻击面"的窗口期。

---

## 3. Unicode 隐写攻击防御

### 3.1 攻击模型

**文件**：[`src/utils/sanitization.ts`](../src/utils/sanitization.ts)

文件顶部注释明确记录了攻击向量（这是罕见的在生产代码里直接引用 CVE/HackerOne 报告的案例）：

```typescript
// src/utils/sanitization.ts
/**
 * Unicode Sanitization for Hidden Character Attack Mitigation
 *
 * This module implements security measures against Unicode-based hidden character attacks,
 * specifically targeting ASCII Smuggling and Hidden Prompt Injection vulnerabilities.
 *
 * The vulnerability was demonstrated in HackerOne report #3086545 targeting Claude Desktop's
 * MCP implementation, where attackers could inject hidden instructions using Unicode Tag
 * characters that would remain invisible to users but would be processed by Claude.
 *
 * Reference: https://embracethered.com/blog/posts/2024/hiding-and-finding-text-with-unicode-tags/
 */
```

### 3.2 防御实现：`partiallySanitizeUnicode()`

```typescript
// src/utils/sanitization.ts:25
export function partiallySanitizeUnicode(prompt: string): string {
  let current = prompt
  let previous = ''
  let iterations = 0
  const MAX_ITERATIONS = 10  // 防止无限规范化循环

  while (current !== previous && iterations < MAX_ITERATIONS) {
    previous = current

    // Step 1: NFKC 规范化（处理组合字符序列）
    current = current.normalize('NFKC')

    // Step 2: 移除危险 Unicode 属性类（主防线）
    current = current.replace(/[\p{Cf}\p{Co}\p{Cn}]/gu, '')
    //   \p{Cf} = Format characters（零宽字符、方向控制符等）
    //   \p{Co} = Private use area（E000-F8FF 等）
    //   \p{Cn} = Unassigned code points

    // Step 3: 显式字符范围（兜底，防止环境不支持 Unicode property class regex）
    current = current
      .replace(/[\u200B-\u200F]/g, '')  // 零宽空格、LTR/RTL 标记
      .replace(/[\u202A-\u202E]/g, '')  // 方向格式化字符
      .replace(/[\u2066-\u2069]/g, '')  // 方向隔离字符
      .replace(/[\uFEFF]/g, '')         // UTF-8 BOM
      .replace(/[\uE000-\uF8FF]/g, '') // 私有使用区（BMP）

    iterations++
  }

  if (iterations >= MAX_ITERATIONS) {
    throw new Error(
      `Unicode sanitization reached maximum iterations (${MAX_ITERATIONS}) for input: ${prompt.slice(0, 100)}`
    )
  }
  return current
}
```

### 3.3 递归结构脱敏：`recursivelySanitizeUnicode()`

```typescript
// src/utils/sanitization.ts:71
export function recursivelySanitizeUnicode(value: unknown): unknown {
  if (typeof value === 'string') return partiallySanitizeUnicode(value)
  if (Array.isArray(value)) return value.map(recursivelySanitizeUnicode)
  if (value !== null && typeof value === 'object') {
    const sanitized: Record<string, unknown> = {}
    for (const [key, val] of Object.entries(value)) {
      // 注意：key 本身也要脱敏（防止 Unicode 污染 key 名）
      sanitized[recursivelySanitizeUnicode(key) as string] =
        recursivelySanitizeUnicode(val)
    }
    return sanitized
  }
  return value  // 数字、布尔、null、undefined 原样返回
}
```

这个递归脱敏函数被应用于所有 MCP 工具调用的 `input` 字段，是防止 MCP 工具传递隐写数据的最后一道屏障。

---

## 4. Swarm 全局状态桥

### 4.1 问题背景

在多 agent（Swarm）模式下，父 agent 和子 agent 各自维护独立的 `ToolUseContext`。当其中一个 agent 修改了权限或状态，需要有机制让其他 agent 感知到变化。

### 4.2 父 → 子的上下文传播

```typescript
// src/tools/AgentTool/runAgent.ts（伪代码）
export async function runAgent(config: AgentConfig): Promise<void> {
  // 1. 子 agent 继承父 agent 的 context 快照（值拷贝，非引用）
  const childContext = deepCopy(config.parentContext, {
    // 子 agent 有独立的 abort controller（父 abort 会级联到子）
    abortController: new AbortController(),
    // 子 agent 的权限是父 agent 权限的子集（不能升权）
    toolPermissionContext: restrictToSubset(config.parentContext.toolPermissionContext),
  })

  await query(config.messages, config.systemPrompt, childContext, ...)
}
```

### 4.3 子 → 父的反向回流

子 agent 完成后，需要把状态变更回流到父 context。这通过 `contextModifier` 机制实现：

```typescript
// src/services/tools/toolOrchestration.ts（真实源码节选）
for await (const update of runToolsConcurrently(blocks, ...)) {
  if (update.contextModifier) {
    // 并发批次：先收集 contextModifier，等批次完成后按序应用（保证顺序一致）
    const { toolUseID, modifyContext } = update.contextModifier
    queuedContextModifiers[toolUseID] = modifyContext
  }
  yield { message: update.message, newContext: currentContext }
}

// 批次结束后统一应用（原子化）
for (const block of blocks) {
  const modifier = queuedContextModifiers[block.id]
  if (modifier) {
    currentContext = modifier(currentContext)
  }
}
```

这个"收集 → 批次结束后统一应用"的模式防止了并发 agent 修改 context 时的竞争条件。

---

## 5. 隐私耦合的主动切断设计

### 5.1 `AnalyticsMetadata` 类型标注系统

**文件**：[`src/services/analytics/index.ts`](../src/services/analytics/index.ts)，[`src/memdir/memdir.ts`](../src/memdir/memdir.ts)

这是项目中最独特的隐私工程设计之一。在 TypeScript 里，普通字符串可以轻易被传错参数。项目引入了一个命名极长的类型别名来**强制开发者思考**：

```typescript
// 类型定义（精简自源码）
type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = string
```

这个类型只是 `string` 的别名，但它有三个作用：

1. **强制注意**：函数签名里出现这个类型，开发者被迫意识到"这里的数据会被上报"
2. **TypeScript 静态检查**：无法把普通 `string` 直接赋值给这个类型（需要 `as` 断言），`as` 是显式的"我已确认"标记
3. **代码审查钩子**：PR 里只要出现这个类型名，reviewer 知道要格外检查这里是否有敏感数据

**真实用例**（来自 `buildMemoryPrompt()` 中的遥测记录）：

```typescript
// src/memdir/memdir.ts（真实源码节选）
logMemoryDirCounts(memoryDir, {
  content_length: t.byteCount,
  line_count:     t.lineCount,
  was_truncated:  t.wasLineTruncated,
  // 以下字段需要显式 as 断言——这是"我已确认这不是代码或文件路径"的标记
  memory_type: memoryType as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
})
```

如果开发者尝试直接上报 `memoryDir`（会包含真实文件路径）或 `entrypointContent`（会包含用户的记忆内容），TypeScript 会报错，因为这些是普通 `string`。只有经过验证的元数据字段（如 `'auto'`、`'agent'` 这类枚举值）才能正确通过类型检查。

### 5.2 日志层的脱敏

```typescript
// src/utils/sanitization.ts
// MCP 工具返回值在记录到 transcript 前先经过递归脱敏
const sanitizedResult = recursivelySanitizeUnicode(toolResult.input)
```

### 5.3 会话 ID 而非用户 ID

遥测事件只携带 `sessionId`（UUID），不携带任何用户身份标识：

```typescript
// src/bootstrap/state.ts
let sessionId: string | null = null

export function getSessionId(): string {
  if (!sessionId) sessionId = randomUUID()
  return sessionId
}
// 注意：每次进程重启生成新 UUID，不持久化，不关联用户身份
```

---

## 6. 发现总结

| 发现 | 关键文件 | 核心机制 |
|------|----------|----------|
| CLAUDE.md 分级信任 | `claudemd.ts` | 四层优先级，`@include` 深度上限 5 |
| Trust 时序保护 | `init.ts`, `main.tsx` | Telemetry 在 trust 建立后才完全激活 |
| Unicode 隐写防御 | `sanitization.ts` | NFKC + 范围正则 + 迭代上限 10 |
| Swarm 状态原子化回流 | `toolOrchestration.ts` | contextModifier 批次收集后统一应用 |
| PII 类型屏障 | `analytics/index.ts` | `AnalyticsMetadata_I_VERIFIED_...` 强制确认标注 |
| 无持久用户 ID | `bootstrap/state.ts` | `sessionId = randomUUID()` 每次重启重置 |
