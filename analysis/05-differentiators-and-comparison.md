# 第十二章：程序架构及亮点

[返回总目录](../README.md)

---

## 1. 导读

本章只讨论 Claude Code 自身的程序架构亮点，不做同类产品对比（对比内容见 [`analysis/08-competitive-comparison.md`](./08-competitive-comparison.md)）。

结论先行：这个项目区别于"模型 API 加壳工具"的核心，在于三点同时成立：
1. **统一的执行内核**，多种运行形态复用同一套 query/tool/permission 闭环
2. **文件化、分层的 memory 体系**，可审计、可分发、可治理
3. **local-first 但无缝扩展**，从本地独立运行平滑扩展到 remote/bridge/swarm

---

## 2. 亮点一：统一执行内核

**文件**：[`src/query.ts`](../src/query.ts)，[`src/QueryEngine.ts`](../src/QueryEngine.ts)

最突出的工程决策：用同一套 `query()` 主循环同时支撑所有运行形态：

```text
REPL（有 UI）       ──┐
headless/SDK       ──┤
subagent（子任务）  ──┤──> query.ts / QueryEngine.ts ──> tool/memory/permission
background agent   ──┤
bridge/remote      ──┘
```

这意味着"本地 REPL 里 Claude 怎么调工具"和"后台自动化任务里 Claude 怎么调工具"走的是**完全相同的代码路径**，不存在两套实现的行为差异风险。

`query.ts` 输出的是 `AsyncGenerator<StreamEvent>`，调用方（UI 层 or SDK 层）只需要消费这个流：

```typescript
// src/query.ts（伪代码骨架）
export async function* query(
  userMessages: Message[],
  systemPrompt: SystemPrompt,
  toolUseContext: ToolUseContext,
  deps: QueryDeps,
): AsyncGenerator<StreamEvent> {
  let messages = userMessages

  while (true) {
    // 1. 调用模型 API，流式输出
    for await (const event of deps.claudeApi.stream(messages, systemPrompt)) {
      yield event
    }

    // 2. 检查是否有 tool_use 需要执行
    const toolUseBlocks = extractToolUseBlocks(messages)
    if (toolUseBlocks.length === 0) break

    // 3. 执行工具，按并发安全性分批
    for await (const update of runTools(toolUseBlocks, ...)) {
      yield update
    }

    // 4. 工具结果追加到 messages -> 下一轮循环
    messages = appendToolResults(messages, toolResults)

    // 5. 会话管理：compact 检查、hook 执行、memory 更新
    await executePostSamplingHooks(messages, toolUseContext)
    if (shouldAutoCompact(messages)) await compactConversation(messages, ...)
  }
}
```

---

## 3. 亮点二：Memory 不是黑盒

**文件**：[`src/memdir/memdir.ts`](../src/memdir/memdir.ts)，[`src/services/SessionMemory/sessionMemory.ts`](../src/services/SessionMemory/sessionMemory.ts)，[`src/tools/AgentTool/agentMemory.ts`](../src/tools/AgentTool/agentMemory.ts)

Memory 系统的独特之处：把所有记忆做成**文件系统上可读写的 Markdown 文件**，而不是黑盒数据库。

`isAutoMemoryEnabled()` 真实源码（[`src/memdir/paths.ts:30`](../src/memdir/paths.ts)）展示了它的控制优先级：

```typescript
// src/memdir/paths.ts
/**
 * 优先级从高到低：
 * 1. CLAUDE_CODE_DISABLE_AUTO_MEMORY 环境变量
 * 2. CLAUDE_CODE_SIMPLE (--bare 模式) → 关闭
 * 3. 远程模式无持久存储时 → 关闭
 * 4. settings.json 中的 autoMemoryEnabled 字段
 * 5. 默认：开启
 */
export function isAutoMemoryEnabled(): boolean {
  const envVal = process.env.CLAUDE_CODE_DISABLE_AUTO_MEMORY
  if (isEnvTruthy(envVal))          return false   // 显式关闭
  if (isEnvDefinedFalsy(envVal))    return true    // 显式开启
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) return false  // --bare 模式
  if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) &&
      !process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR)   return false  // 无持久存储的远程
  const settings = getInitialSettings()
  if (settings.autoMemoryEnabled !== undefined) return settings.autoMemoryEnabled
  return true  // 默认开启
}
```

`MEMORY.md` 的索引截断保护（防止无限增长）：

```typescript
// src/memdir/memdir.ts
export const MAX_ENTRYPOINT_LINES = 200    // 最多 200 行
export const MAX_ENTRYPOINT_BYTES = 25_000 // 最多 25KB

export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  const lines = raw.trim().split('\n')
  let truncated = lines.length > MAX_ENTRYPOINT_LINES
    ? lines.slice(0, MAX_ENTRYPOINT_LINES).join('\n')
    : raw.trim()
  if (truncated.length > MAX_ENTRYPOINT_BYTES) {
    const cutAt = truncated.lastIndexOf('\n', MAX_ENTRYPOINT_BYTES)
    truncated = truncated.slice(0, cutAt > 0 ? cutAt : MAX_ENTRYPOINT_BYTES)
  }
  return { content: truncated + '\n\n> WARNING: MEMORY.md is truncated...', ... }
}
```

这让 Memory 系统具备三个关键属性：

| 属性 | 实现方式 |
|------|----------|
| 透明 | 所有记忆是磁盘上的 `.md` 文件，用户可以直接 `cat` 查看 |
| 可治理 | 四层 memory（Auto/Session/Agent/Team）各自有独立开关 |
| 防越界 | `truncateEntrypointContent()` 硬截断，防止 prompt 爆炸 |

---

## 4. 亮点三：权限系统是主干，不是补丁

**文件**：[`src/Tool.ts`](../src/Tool.ts)，[`src/utils/permissions/permissionSetup.ts`](../src/utils/permissions/permissionSetup.ts)

权限系统从 `Tool` 接口设计之初就是一等公民，不是事后加的过滤层：

```typescript
// src/Tool.ts —— Tool 接口的安全相关字段（精选）
interface Tool {
  isConcurrencySafe(input: unknown): boolean  // 声明是否并发安全
  isReadOnly():   boolean                      // 声明是否只读
  isDestructive(): boolean                     // 声明是否破坏性
  checkPermissions(input: unknown, context: ToolUseContext): PermissionResult
  preparePermissionMatcher(input: unknown): PermissionMatcher
  requiresUserInteraction(): boolean           // 是否需要 UI 交互（如 ask）
}
```

`buildTool()` 的默认值策略体现了**保守安全原则**（fail-closed）：

```typescript
// src/Tool.ts
export function buildTool<T>(spec: ToolSpec<T>): Tool {
  return {
    isConcurrencySafe:  spec.isConcurrencySafe  ?? (() => false),  // 默认不并发
    isReadOnly:         spec.isReadOnly          ?? (() => false),  // 默认非只读
    isDestructive:      spec.isDestructive       ?? (() => false),  // 默认非破坏性
    checkPermissions:   spec.checkPermissions    ?? defaultAllow,   // 默认允许（由外层权限系统控制）
    // ...
  }
}
```

Auto mode 进入时，系统会自动剥离危险权限规则：

```typescript
// src/utils/permissions/permissionSetup.ts
export function stripDangerousPermissionsForAutoMode(
  context: ToolPermissionContext,
): ToolPermissionContext {
  const dangerousPermissions = findDangerousClassifierPermissions(rules, [])
  // 将危险规则从 context 中移除，但保存在 strippedDangerousRules 中
  return {
    ...removeDangerousPermissions(context, dangerousPermissions),
    strippedDangerousRules: stripped,  // 退出 auto mode 时可以恢复
  }
}

export function restoreDangerousPermissions(
  context: ToolPermissionContext,
): ToolPermissionContext {
  // 退出 auto mode 时，把之前保存的危险规则恢复回来
  const stash = context.strippedDangerousRules
  for (const [source, ruleStrings] of Object.entries(stash)) {
    result = applyPermissionUpdate(result, { type: 'addRules', rules: ..., destination: source })
  }
  return { ...result, strippedDangerousRules: undefined }
}
```

---

## 5. 亮点四：多 Agent 协作成熟度高

**文件**：[`src/tools/AgentTool/runAgent.ts`](../src/tools/AgentTool/runAgent.ts)，[`src/utils/swarm/backends/registry.ts`](../src/utils/swarm/backends/registry.ts)

后端选择由 `detectAndGetBackend()` 自动检测（[`src/utils/swarm/backends/registry.ts:131`](../src/utils/swarm/backends/registry.ts)）：

```typescript
// src/utils/swarm/backends/registry.ts
export async function detectAndGetBackend(): Promise<BackendDetectionResult> {
  await ensureBackendsRegistered()
  if (cachedDetectionResult) return cachedDetectionResult  // 缓存结果

  const insideTmux = await isInsideTmux()
  const inITerm2 = isInITerm2()

  // 优先级：tmux > iTerm2 native pane > in-process
  if (insideTmux) {
    return { backend: createTmuxBackend(), isNative: true, needsIt2Setup: false }
  }
  if (inITerm2) {
    if (!check_it2_installed()) {
      return { backend: null, isNative: false, needsIt2Setup: true }  // 需要安装 it2
    }
    return { backend: createITermBackend(), isNative: true, needsIt2Setup: false }
  }
  // 回退到 in-process（不需要额外依赖）
  return { backend: createInProcessBackend(), isNative: false, needsIt2Setup: false }
}
```

这个架构允许：

- **in-process**：轻量级，子 agent 和父 agent 共享进程
- **tmux pane**：每个 teammate 在独立 tmux 窗格中运行，支持并行可视化
- **iTerm2 pane**：利用 iTerm2 原生 API 创建标签页

---

## 6. 亮点五：长会话治理做成体系

**文件**：[`src/services/compact/compact.ts`](../src/services/compact/compact.ts)

`compactConversation()` 不是简单的"历史截断"，而是完整的会话压缩流程：

```typescript
// src/services/compact/compact.ts:387
export async function compactConversation(
  messages: Message[],
  context: ToolUseContext,
  cacheSafeParams: CacheSafeParams,
  suppressFollowUpQuestions: boolean,
  customInstructions?: string,
  isAutoCompact: boolean = false,
): Promise<CompactionResult> {
  // 1. 预处理：移除图片和大型附件，防止摘要请求本身超 Token 上限
  const strippedMessages = stripImagesFromMessages(
    stripReinjectedAttachments(messages)
  )

  // 2. 执行 PreCompact hooks（可自定义压缩策略）
  const hookResult = await executePreCompactHooks({
    trigger: isAutoCompact ? 'auto' : 'manual',
    customInstructions: customInstructions ?? null,
  }, context.abortController.signal)

  // 3. 尝试用 Session Memory 直接充当摘要（避免额外 API 调用）
  const smResult = await trySessionMemoryCompaction(messages, context)
  if (smResult) return smResult

  // 4. 调用模型生成摘要
  // ...（调用 claude.ts，生成 summary message）

  // 5. PTL（Prompt Too Long）重试保护
  // 若摘要请求本身也超限，truncateHeadForPTLRetry() 从头部裁去几轮历史
}
```

```typescript
// src/services/compact/compact.ts —— 压缩后能力复灌
export function buildPostCompactMessages(result: CompactionResult): Message[] {
  // compact 后，模型的 tool schema 和 MCP 工具描述都被清空了
  // 这个函数负责重建：文件附件 + Plans 清单 + 工具能力声明
  return [
    ...createPostCompactFileAttachments(result),  // 恢复文件上下文
    getDeferredToolsDeltaAttachment(result),       // 重新声明所有工具能力
  ]
}
```

compact 体系包含四种策略：

| 策略 | 触发条件 | 核心文件 |
|------|----------|----------|
| 手动 compact | 用户执行 `/compact` | `compact.ts` |
| 自动 compact | Token 超过阈值 | `autoCompact.ts` |
| Session Memory compact | 有 Session Memory 文件时 | `sessionMemoryCompact.ts` |
| Micro compact | 轻量缩减（实验性） | `reactiveCompact.ts` |

---

## 7. 总体架构判断

| 特征 | 实现证据 |
|------|----------|
| 统一执行内核 | `query()` 异步生成器，所有运行形态共用 |
| 文件化 memory | Markdown 文件 + `MEMORY.md` 索引，`truncateEntrypointContent()` 防越界 |
| 权限系统主干化 | `Tool.ts` 接口内建 `checkPermissions`，`buildTool()` fail-closed 默认值 |
| 多 agent runtime | `detectAndGetBackend()` 自动选 tmux/iTerm2/in-process 后端 |
| 长会话体系化 | `compactConversation()` + Session Memory + `buildPostCompactMessages()` 复灌 |

如果只用一句话总结：**Claude Code 不是"把 Claude API 包一层 CLI 皮肤"，而是一套有独立 query 内核、文件化 memory、主干化权限系统和多 agent runtime 的本地 agent 平台。**
