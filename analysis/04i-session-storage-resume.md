# 第十一章：Session Storage / Transcript / Resume 持久化机制

[返回总目录](../README.md)

## 1. 本章导读

这一章直接回答一个核心问题：

**Claude Code 的会话不是“内存里聊完就算”，而是被实现成一套 append-only transcript 日志系统。`/resume` 也不是简单把旧消息数组重新塞回 REPL，而是要经历“日志加载 -> 元数据恢复 -> 链路修复 -> UI 重新接管”这一整条恢复流水线。**

本章重点讲清楚：

1. transcript 实际存在哪里
2. 写入为什么采用 append-only JSONL
3. 哪些内容进入 transcript，哪些不会
4. title / tag / agent / mode / worktree / PR 这些元数据如何持久化
5. 本地 transcript、远端 ingress、subagent sidechain 之间是什么关系
6. `/resume` 时如何从日志重建出一条可继续运行的会话链

本章主要依据这些实现：

- [`src/utils/sessionStorage.ts`](../src/utils/sessionStorage.ts)
- [`src/utils/sessionStoragePortable.ts`](../src/utils/sessionStoragePortable.ts)
- [`src/utils/conversationRecovery.ts`](../src/utils/conversationRecovery.ts)
- [`src/screens/ResumeConversation.tsx`](../src/screens/ResumeConversation.tsx)
- [`src/services/api/sessionIngress.ts`](../src/services/api/sessionIngress.ts)

先给结论：

Claude Code 的会话持久化不是数据库快照模型，而是下面这套分层结构：

```text
一、主 transcript
   - 每个 session 一个 .jsonl
   - user / assistant / attachment / system 以 append-only 方式写入

二、附加元数据条目
   - summary / custom-title / tag / agent-setting / mode / worktree-state / pr-link
   - 与正文同样写进 transcript，但单独按类型恢复

三、subagent sidechain transcript
   - 每个 agent 独立 .jsonl
   - 用于 fork / teammate / subagent 的恢复

四、远端 ingress 副本
   - 主 transcript 的远端 append 链
   - 用于 hydrate 和跨进程恢复

五、resume 恢复流水线
   - 读取 JSONL
   - 修复 compact / snip / progress / parallel tool result 造成的链路问题
   - 恢复 metadata / fileHistory / contextCollapse / worktree / agent 状态
   - 最终再交回 REPL
```

换句话说，这个项目把“会话状态”拆成了两层：

- **写入层尽量简单**：只追加，不原地改写
- **恢复层负责复杂性**：读取时补齐、修复、重建

## 2. 存储模型：核心不是消息数组，而是 append-only JSONL 事件流

相关实现：

- [`src/utils/sessionStorage.ts`](../src/utils/sessionStorage.ts)

`sessionStorage.ts` 最关键的设计，不是“把消息写到文件”，而是**把 transcript 当成事件流日志，而不是可变快照**。

### 2.1 哪些东西算 transcript

源码把“什么算 transcript message”写得非常明确：

**真实源码** ([`src/utils/sessionStorage.ts:139`](../src/utils/sessionStorage.ts#L139))

```typescript
export function isTranscriptMessage(entry: Entry): entry is TranscriptMessage {
  return (
    entry.type === 'user' ||
    entry.type === 'assistant' ||
    entry.type === 'attachment' ||
    entry.type === 'system'
  )
}
```

对应的注释还明确说明：

- `progress` 不是 transcript message
- `progress` 不能进入 `parentUuid` 主链
- 旧版本把 progress 混进 transcript 后，恢复时会把真实对话链截断

这说明 Claude Code 的 transcript 设计目标很清楚：

- **保留真正影响上下文重建的消息**
- **把高频 UI 状态从持久化层排除**

### 2.2 transcript 路径不是固定死的，而是会跟 session 切换联动

**真实源码** ([`src/utils/sessionStorage.ts:202`](../src/utils/sessionStorage.ts#L202))

```typescript
export function getTranscriptPath(): string {
  const projectDir = getSessionProjectDir() ?? getProjectDir(getOriginalCwd())
  return join(projectDir, `${getSessionId()}.jsonl`)
}
```

**真实源码** ([`src/utils/sessionStorage.ts:247`](../src/utils/sessionStorage.ts#L247))

```typescript
export function getAgentTranscriptPath(agentId: AgentId): string {
  const projectDir = getSessionProjectDir() ?? getProjectDir(getOriginalCwd())
  const sessionId = getSessionId()
  const subdir = agentTranscriptSubdirs.get(agentId)
  const base = subdir
    ? join(projectDir, sessionId, 'subagents', subdir)
    : join(projectDir, sessionId, 'subagents')
  return join(base, `agent-${agentId}.jsonl`)
}
```

这里至少有三个关键点：

1. **主 transcript 是 `sessionId.jsonl`**
2. **subagent transcript 不和主链混写，而是单独 sidechain 文件**
3. **路径解析依赖 `sessionProjectDir` 和当前 sessionId，同一个会话在 resume / branch / switch 后仍能落到正确目录**

所以它不是“当前目录下写一个聊天记录文件”，而是一个带 session 路由能力的日志存储层。

## 3. 写入路径：先做异步批量追加，再按类型分流

相关实现：

- [`src/utils/sessionStorage.ts`](../src/utils/sessionStorage.ts)

### 3.1 底层写盘器先做批量 flush

**真实源码** ([`src/utils/sessionStorage.ts:634`](../src/utils/sessionStorage.ts#L634))

```typescript
private async appendToFile(filePath: string, data: string): Promise<void> {
  try {
    await fsAppendFile(filePath, data, { mode: 0o600 })
  } catch {
    await mkdir(dirname(filePath), { recursive: true, mode: 0o700 })
    await fsAppendFile(filePath, data, { mode: 0o600 })
  }
}
```

**真实源码** ([`src/utils/sessionStorage.ts:645`](../src/utils/sessionStorage.ts#L645))

```typescript
private async drainWriteQueue(): Promise<void> {
  for (const [filePath, queue] of this.writeQueues) {
    const batch = queue.splice(0)
    let content = ''
    for (const { entry } of batch) {
      const line = jsonStringify(entry) + '\n'
      ...
      content += line
    }
    await this.appendToFile(filePath, content)
  }
}
```

这段代码说明：

- session storage 并不是每来一条就同步 `writeFile`
- 它先进入内存队列，再由 `drainWriteQueue()` 批量 flush
- 文件和目录权限也明确固定为 `0600 / 0700`

这是一种典型的日志系统设计：

- 追加简单
- 崩溃恢复容易
- 不要求每次都重写整份 transcript

### 3.2 `appendEntry()` 才是真正的“分流器”

`appendEntry()` 是整个持久化系统的主入口。它决定一条 entry 应该：

- 写入主 transcript
- 写入 sidechain transcript
- 只更新 metadata
- 是否还要同步发到远端 ingress

可以把它改写成下面这段伪代码：

```typescript
async function appendEntry(entry, sessionId) {
  if (当前 session 文件还没 materialize) {
    pendingEntries.push(entry)
    return
  }

  if (entry 是 summary/title/tag/mode/worktree/pr-link 等 metadata) {
    enqueueWrite(mainSessionFile, entry)
    return
  }

  if (entry.type === 'content-replacement') {
    target = entry.agentId ? agentSidechainFile : mainSessionFile
    enqueueWrite(target, entry)
    return
  }

  // 剩下的就是 transcript message
  target = entry.isSidechain ? agentSidechainFile : mainSessionFile

  if (target 是 sidechain 本地文件) {
    // 允许写入与主链重复 UUID，保证 fork 后的上下文完整
    enqueueWrite(target, entry)
    return
  }

  if (uuid 之前没写过) {
    enqueueWrite(mainSessionFile, entry)
    messageSet.add(entry.uuid)
    persistToRemote(sessionId, entry)
  }
}
```

这个设计很关键，因为它解决了两个互相冲突的问题：

1. **主 transcript 不能重复写同一个 UUID**
   否则 resume 会遇到重复链路和远端 409 冲突
2. **sidechain 又必须允许继承消息重复出现**
   否则 fork / subagent 恢复时会丢掉继承来的父上下文

源码里对此写得非常直白：

**真实源码** ([`src/utils/sessionStorage.ts:1212`](../src/utils/sessionStorage.ts#L1212))

```typescript
const isAgentSidechain =
  entry.isSidechain && entry.agentId !== undefined
const targetFile = isAgentSidechain
  ? getAgentTranscriptPath(asAgentId(entry.agentId!))
  : sessionFile

const isNewUuid = !messageSet.has(entry.uuid)
if (isAgentSidechain || isNewUuid) {
  void this.enqueueWrite(targetFile, entry)

  if (!isAgentSidechain) {
    messageSet.add(entry.uuid)
    if (isTranscriptMessage(entry)) {
      await this.persistToRemote(sessionId, entry)
    }
  }
}
```

结论很明确：**主链去重，sidechain 保真，远端只跟主链走。**

## 4. 元数据持久化：不是独立数据库，而是“同日志写入 + 尾部重挂”

相关实现：

- [`src/utils/sessionStorage.ts`](../src/utils/sessionStorage.ts)

很多项目会把 title、tag、mode、PR 关联这些元数据单独放进 SQLite 或 JSON sidecar。Claude Code 没这么做，它还是写回 transcript，但加了一层很重要的机制：

**metadata 会被周期性重挂到 transcript 尾部。**

### 4.1 为什么要重挂到尾部

源码注释把原因写得很清楚：

**真实源码** ([`src/utils/sessionStorage.ts:686`](../src/utils/sessionStorage.ts#L686))

```typescript
/**
 * Re-append cached session metadata to the end of the transcript file.
 * This ensures metadata stays within the tail window that readLiteMetadata
 * reads during progressive loading.
 */
```

也就是说，系统有两种读取方式：

1. **完整恢复**
   读取 transcript 全量内容
2. **lite 列表读取**
   只读头尾窗口，快速展示 session 列表、title、tag、firstPrompt

如果 title / tag 太早写进 transcript，后面会被越来越长的对话“挤出 tail window”，列表页就看不到它们。所以它必须反复重挂到 EOF。

### 4.2 `reAppendSessionMetadata()` 的真实逻辑

它不是盲目把缓存重写一遍，而是先吸收 tail 里更新过的值，再重挂：

```typescript
function reAppendSessionMetadata(skipTitleRefresh = false) {
  tail = readFileTailSync(sessionFile)

  // 先从 tail 吸收外部 SDK 改过的新 title/tag，避免本地缓存把它覆盖回旧值
  refreshTitleAndTagFromTail(tail)

  append(last-prompt)
  append(custom-title)
  append(tag)
  append(agent-name)
  append(agent-color)
  append(agent-setting)
  append(mode)
  append(worktree-state)
  append(pr-link)
}
```

这个机制的意义有三层：

1. **让渐进式 session 列表加载能从 tail 快速读到关键 metadata**
2. **让 resume 后尚未发送新消息的 session，也能在退出时把 metadata 落盘**
3. **兼容外部 SDK / 其他进程对 title、tag 的修改**

### 4.3 resume 时为什么还要专门 `adoptResumedSessionFile()`

**真实源码** ([`src/utils/sessionStorage.ts:1511`](../src/utils/sessionStorage.ts#L1511))

```typescript
export function adoptResumedSessionFile(): void {
  const project = getProject()
  project.sessionFile = getTranscriptPath()
  project.reAppendSessionMetadata(true)
}
```

这一步专门解决一个很实际的问题：

- 用户 resume 了旧 session
- 改了标题或其他 metadata
- 但还没发新消息就退出

如果此时 `sessionFile` 仍然是 `null`，退出清理逻辑虽然有缓存，但不会真正写盘。`adoptResumedSessionFile()` 就是在 resume 后立刻把“当前持久化目标”绑定到旧 transcript，让后续 metadata 重挂有落点。

### 4.4 metadata 恢复不是重新解析 UI，而是恢复进内存缓存

**真实源码** ([`src/utils/sessionStorage.ts:2758`](../src/utils/sessionStorage.ts#L2758))

```typescript
export function restoreSessionMetadata(meta: {
  customTitle?: string
  tag?: string
  agentName?: string
  agentColor?: string
  agentSetting?: string
  mode?: 'coordinator' | 'normal'
  worktreeSession?: PersistedWorktreeSession | null
  prNumber?: number
  prUrl?: string
  prRepository?: string
}): void
```

它恢复的不是“页面状态”，而是 `Project` 里的当前 session 缓存。后面 agent banner、mode、worktree、退出时 metadata 重挂，都依赖这个缓存。

## 5. transcript 不是只有本地文件，还有远端 ingress 副本

相关实现：

- [`src/services/api/sessionIngress.ts`](../src/services/api/sessionIngress.ts)
- [`src/utils/sessionStorage.ts`](../src/utils/sessionStorage.ts)

Claude Code 的 session persistence 不是纯本地行为。主 transcript message 在本地 append 的同时，还可以增量发到远端 ingress。

### 5.1 远端不是上传整个文件，而是 append 链

**真实源码** ([`src/services/api/sessionIngress.ts:57`](../src/services/api/sessionIngress.ts#L57))

```typescript
async function appendSessionLogImpl(sessionId, entry, url, headers) {
  const lastUuid = lastUuidMap.get(sessionId)
  if (lastUuid) {
    requestHeaders['Last-Uuid'] = lastUuid
  }

  const response = await axios.put(url, entry, { ... })
}
```

这表明远端协议不是“上传整个 transcript 文件”，而是：

- 每次 PUT 一条 entry
- 用 `Last-Uuid` 做乐观并发控制
- 服务端返回 409 时，客户端会尝试吸收服务器头部的最新 UUID，再继续重试

### 5.2 同一个 session 的远端 append 必须串行化

**真实源码** ([`src/services/api/sessionIngress.ts:24`](../src/services/api/sessionIngress.ts#L24))

```typescript
const sequentialAppendBySession = new Map(...)
```

对应的 `getOrCreateSequentialAppend()` 会给每个 session 建一条顺序执行队列，避免同一 session 并发写远端造成链头竞争。

这和本地 append-only 设计是配套的：

- 本地允许高频异步批量追加
- 远端则要求单 session 串行顺序

### 5.3 resume/hydrate 时可以反向把远端 transcript 拉回本地

**真实源码** ([`src/utils/sessionStorage.ts:1587`](../src/utils/sessionStorage.ts#L1587))

```typescript
export async function hydrateRemoteSession(sessionId: string, ingressUrl: string) {
  const remoteLogs = (await sessionIngress.getSessionLogs(sessionId, ingressUrl)) || []
  const sessionFile = getTranscriptPathForSession(sessionId)
  const content = remoteLogs.map(e => jsonStringify(e) + '\n').join('')
  await writeFile(sessionFile, content, ...)
}
```

也就是说，远端 ingress 不只是备份，还承担了 hydrate source 的作用。另一个变体是 `hydrateFromCCRv2InternalEvents()`，它甚至会把 foreground transcript 和 subagent transcript 分别写回本地文件。

结论：**本地 transcript 是运行时主副本，远端 ingress 是可回灌的增量副本。**

## 6. 快速列表与大文件加载：读取层不是全量 parse，而是分层优化

相关实现：

- [`src/utils/sessionStoragePortable.ts`](../src/utils/sessionStoragePortable.ts)
- [`src/utils/sessionStorage.ts`](../src/utils/sessionStorage.ts)

### 6.1 session 列表不是每次都全量 parse transcript

`sessionStoragePortable.ts` 明确提供了一套 lite reader：

**真实源码** ([`src/utils/sessionStoragePortable.ts:17`](../src/utils/sessionStoragePortable.ts#L17))

```typescript
export const LITE_READ_BUF_SIZE = 65536
```

它只读文件头尾 64KB，并从中提取：

- first prompt
- custom title
- tag
- 其他用于列表展示的轻量元数据

`extractFirstPromptFromHead()` 的实现也很务实：

- 跳过 `tool_result`
- 跳过 `isMeta`
- 跳过 compact summary
- 跳过 `<command-name>` 包装和系统自动注入片段

所以 session 列表页不是靠“反序列化整段对话”实现的，而是靠**头尾窗口 + 字段提取**实现的。

### 6.2 大 transcript 的完整恢复也不是无脑整文件读入

**真实源码** ([`src/utils/sessionStorage.ts:3511`](../src/utils/sessionStorage.ts#L3511))

```typescript
if (size > SKIP_PRECOMPACT_THRESHOLD) {
  const scan = await readTranscriptForLoad(filePath, size)
  buf = scan.postBoundaryBuf
  hasPreservedSegment = scan.hasPreservedSegment
  if (scan.boundaryStartOffset > 0) {
    metadataLines = await scanPreBoundaryMetadata(filePath, scan.boundaryStartOffset)
  }
}
```

这里的意思是：

- 如果 transcript 很大，不直接把整份文件都交给 JSON parser
- 先在文件级别扫描 compact boundary
- 把 boundary 之前的废弃历史尽量裁掉
- 但又单独保留 boundary 前的 metadata 行，避免 title / mode / agent-setting 丢失

`readTranscriptForLoad()` 本身就是专门干这件事的：

**真实源码** ([`src/utils/sessionStoragePortable.ts:717`](../src/utils/sessionStoragePortable.ts#L717))

```typescript
export async function readTranscriptForLoad(filePath, fileSize): Promise<{
  boundaryStartOffset: number
  postBoundaryBuf: Buffer
  hasPreservedSegment: boolean
}>
```

这说明恢复层已经不是“读文件 -> parseJSONL”这么简单，而是：

```typescript
if (文件很大) {
  buf = 仅读取 boundary 之后仍有效的部分
  metadata = 从 boundary 之前补扫 session-scoped metadata
} else {
  buf = readFile(filePath)
}
```

## 7. 完整恢复：`loadTranscriptFile()` 不是读日志，而是在重建会话图

相关实现：

- [`src/utils/sessionStorage.ts`](../src/utils/sessionStorage.ts)

`loadTranscriptFile()` 是这一章最核心的函数。它不是单纯地“把 JSONL 解析成数组”，而是在做下面这些事：

1. 按 entry type 分流到不同 Map
2. 把旧版 progress 链桥接掉
3. 应用 compact / preserved segment / snip 修复
4. 收集 file history、attribution、content replacement、context collapse
5. 重新计算 leaf UUID

可以把它概括成下面的伪代码：

```typescript
async function loadTranscriptFile(filePath) {
  messages = new Map()
  summaries = new Map()
  titles = new Map()
  tags = new Map()
  agentSettings = new Map()
  contentReplacements = new Map()
  contextCollapseCommits = []

  buf = maybeReadPostCompactRegionOnly(filePath)
  metadataLines = maybeScanPreBoundaryMetadata(filePath)

  for (line of metadataLines) {
    restoreSessionScopedMaps(line)
  }

  progressBridge = new Map()
  for (entry of parseJSONL(buf)) {
    if (entry is legacy progress) {
      progressBridge[entry.uuid] = resolvedParent
      continue
    }

    if (entry is transcript message) {
      if (entry.parentUuid 指向 legacy progress) {
        entry.parentUuid = progressBridge[parent]
      }
      messages.set(entry.uuid, entry)
      continue
    }

    switch (entry.type) {
      case 'summary': ...
      case 'custom-title': ...
      case 'tag': ...
      case 'agent-setting': ...
      case 'content-replacement': ...
      case 'marble-origami-commit': ...
    }
  }

  applyPreservedSegmentRelinks(messages)
  applySnipRemovals(messages)
  leafUuids = recomputeLeaves(messages)

  return allMapsAndLeafs
}
```

这说明 Claude Code 的 transcript loader 实际上是在重建一张“会话图”，而不是还原一份简单聊天记录。

## 8. resume 修复：为什么读取后还要再做链路补救

相关实现：

- [`src/utils/sessionStorage.ts`](../src/utils/sessionStorage.ts)
- [`src/utils/conversationRecovery.ts`](../src/utils/conversationRecovery.ts)

append-only 日志的优点是写入简单，但代价是读取侧必须更强健。Claude Code 为此专门加了几层恢复修复逻辑。

### 8.1 `applySnipRemovals()` 会删消息，还会重新接 parentUuid

**真实源码** ([`src/utils/sessionStorage.ts:1982`](../src/utils/sessionStorage.ts#L1982))

它处理的是这种问题：

- 某段消息被 snip 掉了
- 但幸存消息的 `parentUuid` 仍然指向被删区间中的某条消息

如果只删除不重连，恢复链会直接断。这个函数会沿着被删消息自己的 `parentUuid` 继续向前走，直到找到仍然存在的祖先，再把幸存消息挂回去。

这不是普通聊天产品会做的事，说明 transcript 在这里已经是可修复的图结构，不是静态数组。

### 8.2 `buildConversationChain()` 不是简单回溯，它还会补 parallel tool result

**真实源码** ([`src/utils/sessionStorage.ts:2069`](../src/utils/sessionStorage.ts#L2069))

```typescript
export function buildConversationChain(messages, leafMessage) {
  ...
  transcript.reverse()
  return recoverOrphanedParallelToolResults(messages, transcript, seen)
}
```

后面的 `recoverOrphanedParallelToolResults()` 专门处理一类很棘手的情况：

- assistant 一次输出多个并行 tool_use
- streaming 过程中这些块会被拆成多个 assistant message
- tool_result 分别挂到不同 assistant block 上
- 单纯按单 parent 链逆推，只会保留其中一支

所以恢复时必须再做一次“补兄弟节点 + 补孤立 tool_result”的后处理。

这也是为什么 resume 逻辑不能只依赖 `parentUuid` 单链遍历。

### 8.3 `checkResumeConsistency()` 还会做恢复一致性审计

**真实源码** ([`src/utils/sessionStorage.ts:2224`](../src/utils/sessionStorage.ts#L2224))

它会拿最新 checkpoint 里记录的 `messageCount`，和当前恢复出的 chain 长度位置做比较，专门监控“写入时显示的会话”和“恢复后读出来的会话”是否发生漂移。

这说明作者已经明确把 resume drift 当成线上风险看待，而不是纯理论问题。

## 9. `/resume` 主链：conversationRecovery 负责把“日志”变回“可继续运行的消息流”

相关实现：

- [`src/utils/conversationRecovery.ts`](../src/utils/conversationRecovery.ts)

`loadConversationForResume()` 是 resume 入口，不管来源是：

- 最近一次会话
- 指定 sessionId
- 指定 `.jsonl` 路径
- 已经加载好的 `LogOption`

最终都会汇聚到同一套恢复逻辑。

### 9.1 它做的不是单步读取，而是整套恢复编排

可以改写成下面的伪代码：

```typescript
async function loadConversationForResume(source) {
  log = resolveSourceToLogOrJsonl(source)

  if (log is lite log) {
    log = loadFullLog(log)
  }

  sessionId = resolveSessionId(log)
  copyPlanForResume(log, sessionId)
  copyFileHistoryForResume(log)

  messages = log.messages
  checkResumeConsistency(messages)

  restoreSkillStateFromMessages(messages)

  deserialized = deserializeMessagesWithInterruptDetection(messages)
  messages = deserialized.messages

  hookMessages = processSessionStartHooks('resume', { sessionId })
  messages.push(...hookMessages)

  return {
    messages,
    turnInterruptionState,
    fileHistorySnapshots,
    attributionSnapshots,
    contentReplacements,
    contextCollapseCommits,
    session metadata...
  }
}
```

这里有几个容易被忽略但很重要的点：

1. **resume 前会恢复 invoked skills 状态**
   否则再次 compact 时可能把之前的 skill 状态丢掉
2. **会过滤 unresolved tool uses、orphaned thinking-only messages、纯空白 assistant**
   这是为了保证恢复出来的 transcript 对 API 仍然合法
3. **会检测中断 turn，并在必要时注入 `Continue from where you left off.`**
   这是把“被中断的旧会话”重新变成“可继续跑的当前会话”
4. **会重新跑 session start hooks**
   说明 resume 不是纯静态回放，而是一次新的运行时接管

## 10. UI Resume：真正把恢复结果接回 REPL 的是 `ResumeConversation`

相关实现：

- [`src/screens/ResumeConversation.tsx`](../src/screens/ResumeConversation.tsx)

`ResumeConversation.tsx` 不是简单调一下 `loadConversationForResume()` 就完事，它负责把“恢复出来的逻辑状态”重新接回当前进程。

核心流程可以概括为：

```typescript
result = await loadConversationForResume(log)

if (result.sessionId && !forkSession) {
  switchSession(result.sessionId)
  renameRecordingForSession()
  resetSessionFilePointer()
  restoreCostStateForSession(result.sessionId)
}

restoreAgentFromSession(...)
restoreSessionMetadata(result)
restoreWorktreeForResume(result.worktreeSession)

if (result.sessionId) {
  adoptResumedSessionFile()
}

restoreContextCollapse(...)

render(<REPL initialMessages={result.messages} ... />)
```

这段逻辑说明 resume 真正恢复的不只是消息：

- sessionId
- asciicast 录制文件名
- cost tracker
- agent identity
- session metadata cache
- worktree 状态
- context collapse 持久化状态
- 最终 REPL 初始消息集

所以 `/resume` 本质上是一次**运行时状态接管**，而不是“把旧 transcript 打开看看”。

## 11. 技术结论：写入路径故意做简单，复杂性全部压到恢复路径

这一套实现的技术取向非常明确：

### 11.1 优点

- 写入简单，append-only，崩溃后更容易保留证据
- transcript、metadata、subagent、remote ingress 可以增量同步
- 大文件有 lite reader 和 pre-compact skip，不必每次全量 parse
- resume 具备较强的兼容能力，能修复旧版 progress、snip、parallel tool result、interrupted turn

### 11.2 代价

- 读取路径明显比写入路径复杂得多
- transcript 已经不是线性消息数组，而是带修复规则的图结构
- compact / preserved segment / sidechain / content replacement / context collapse 全都让恢复链路更难维护

### 11.3 最关键的判断

Claude Code 的 Session Storage 不是“本地聊天记录文件”，而是：

**一个以 JSONL 为底层介质、支持 metadata 尾部重挂、支持 sidechain、支持 remote ingress、支持恢复修复与运行时接管的会话日志系统。**

这也是为什么 `sessionStorage.ts` 会非常大。它承担的不是单一 IO 功能，而是整个 agent runtime 的长期状态底座。
