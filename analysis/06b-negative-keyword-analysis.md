# 第十五章：负面关键词检测与挫败感信号机制

[返回总目录](../README.md)

## 1. 本章导读

这一章专门回答一个看起来很小、但其实很产品化的问题：

**为什么源码里会有一个用正则匹配 `wtf`、`this sucks`、`damn it` 之类词汇的函数？**

先给结论：

这段代码不是安全过滤器，也不是内容审查器，更不是用来拦截用户输入的。

它的主要作用是：

1. 在用户提交 prompt 时，快速给输入打一个“负面情绪/不满”标签
2. 把这个标签写进埋点事件 `tengu_input_prompt`
3. 作为产品分析和体验评估的轻量信号，辅助识别“用户是否在烦躁、是否在不满意”

更进一步地看，这段代码并不是孤立存在的，它和另一个更高层的产品链路形成配合：

- 一层是**输入级轻量关键词打标**
- 一层是**会话级 frustration detection**
- 最终可能导向**反馈问卷 / transcript sharing**

所以它的真实定位不是“文本处理技巧”，而是 **product telemetry + frustration sensing** 的一部分。

本章主要依据这些实现：

- [`src/utils/userPromptKeywords.ts`](../src/utils/userPromptKeywords.ts)
- [`src/utils/processUserInput/processTextPrompt.ts`](../src/utils/processUserInput/processTextPrompt.ts)
- [`src/utils/processUserInput/processSlashCommand.tsx`](../src/utils/processUserInput/processSlashCommand.tsx)
- [`src/screens/REPL.tsx`](../src/screens/REPL.tsx)
- [`src/components/FeedbackSurvey/useFeedbackSurvey.tsx`](../src/components/FeedbackSurvey/useFeedbackSurvey.tsx)
- [`src/components/FeedbackSurvey/submitTranscriptShare.ts`](../src/components/FeedbackSurvey/submitTranscriptShare.ts)
- [`src/commands/insights.ts`](../src/commands/insights.ts)

## 2. 先看原始函数：它到底做了什么

相关实现：

- [`src/utils/userPromptKeywords.ts`](../src/utils/userPromptKeywords.ts)

原始函数如下：

```typescript
export function matchesNegativeKeyword(input: string): boolean {
  const lowerInput = input.toLowerCase()

  const negativePattern =
    /\b(wtf|wth|ffs|omfg|shit(ty|tiest)?|dumbass|horrible|awful|piss(ed|ing)? off|piece of (shit|crap|junk)|what the (fuck|hell)|fucking? (broken|useless|terrible|awful|horrible)|fuck you|screw (this|you)|so frustrating|this sucks|damn it)\b/

  return negativePattern.test(lowerInput)
}
```

如果只从函数体看，它做的事情非常简单：

```text
输入字符串
  -> 转小写
  -> 用一组负面词/抱怨词正则匹配
  -> 返回 true / false
```

也就是说，它本身没有：

- 改写 prompt
- 阻止 prompt 发送
- 降权模型回答
- 触发任何权限拒绝
- 直接弹出用户界面

它只是一个布尔判定器。

## 3. 这段代码真正被用在什么地方

相关实现：

- [`src/utils/processUserInput/processTextPrompt.ts`](../src/utils/processUserInput/processTextPrompt.ts)

这个函数的唯一直接调用点在 `processTextPrompt()`：

```typescript
const isNegative = matchesNegativeKeyword(userPromptText)
const isKeepGoing = matchesKeepGoingKeyword(userPromptText)
logEvent('tengu_input_prompt', {
  is_negative: isNegative,
  is_keep_going: isKeepGoing,
})
```

这段代码非常关键，因为它说明：

1. `matchesNegativeKeyword()` 发生在**用户输入刚进入系统**的时候
2. 它的输出只被写进一个 analytics 事件
3. 与它并列的还有 `matchesKeepGoingKeyword()`

这说明作者把它视为一种**输入意图标签**，而不是控制流逻辑。

### 3.1 它所处的位置

`processTextPrompt()` 这条链路大致是：

```text
用户输入文本
  -> 生成 promptId
  -> 记录 OTEL user_prompt
  -> 计算 is_negative / is_keep_going
  -> logEvent('tengu_input_prompt', ...)
  -> 构造 UserMessage
  -> 正常进入 query
```

也就是说，关键词匹配发生得很早，但它并不会改变后续 query 逻辑。

### 3.2 slash command 反而没有这个标签

这是一个很有意思的细节。

在 [`src/utils/processUserInput/processSlashCommand.tsx`](../src/utils/processUserInput/processSlashCommand.tsx) 里，slash command 也会记录同名事件：

```typescript
logEvent('tengu_input_prompt', {});
```

但这里没有 `is_negative` 和 `is_keep_going`。

这说明产品假设是：

- 自然语言 prompt 值得做情绪与意图打标
- slash command 本身更像结构化命令，没必要做这种文本情绪分析

## 4. 这段正则为什么会存在：从产品角度看

这类代码最容易被误解成“模型行为控制逻辑”，其实它更像体验分析基础设施。

它出现的原因，基本可以归纳成下面四个工程目的。

### 4.1 识别明显的挫败/抱怨输入

像下面这类词：

- `wtf`
- `wth`
- `this sucks`
- `so frustrating`
- `damn it`
- `piece of shit`

都不是普通任务描述，而更像用户对产品状态的即时反馈。

这类信号对产品团队非常有价值，因为它们往往意味着：

- 用户觉得 Claude Code 刚刚做错了
- 用户觉得当前交互很卡、很烦
- 用户正在失去耐心

而这些信号不一定会体现在显式反馈按钮中。

### 4.2 给后端分析一个廉价而稳定的标签

比起每次都用模型去分类“用户是否生气”，这个正则方案有几个现实优势：

1. 成本低
   不需要额外 API 调用。
2. 延迟低
   本地同步计算即可完成。
3. 稳定
   同样输入总是同样结果，不受模型波动影响。
4. 易统计
   可以直接在埋点里按布尔字段聚合。

所以这更像是一个**廉价的 frustration heuristic**。

### 4.3 作为更复杂体验系统的前置信号

单看 `matchesNegativeKeyword()`，它只是打标。

但在整个产品里，“用户是否 frustrated”显然是一个更大的分析维度。

例如在 [`src/commands/insights.ts`](../src/commands/insights.ts) 里，满意度标签中就有：

```typescript
frustrated: 'Frustrated',
dissatisfied: 'Dissatisfied',
likely_satisfied: 'Likely Satisfied',
satisfied: 'Satisfied',
happy: 'Happy',
```

而 facet 提取 prompt 里还明确举例：

```text
"this is broken", "I give up" → frustrated
```

这说明“frustrated 用户”是一个正式的分析维度，不是临时拼出来的。

### 4.4 为反馈收集和质量改进提供触发条件

在 REPL 里可以看到一句很关键的注释：

```typescript
// Frustration detection: show transcript sharing prompt after detecting frustrated messages
const frustrationDetection = useFrustrationDetection(...)
```

随后在 UI 渲染里又有：

```typescript
{frustrationDetection.state !== 'closed' && <FeedbackSurvey ... />}
```

这说明系统里明确存在一条产品链路：

```text
检测到挫败感
  -> 弹出反馈 / 转录分享提示
  -> 用户允许时上传 transcript
  -> 用于改进 Claude Code
```

虽然在当前提取出的 `src/` 中没有找到 `useFrustrationDetection` 的源码文件，但 REPL 接线和 transcript sharing 的 trigger 已经足以证明这条链路存在。

## 5. 它和 transcript sharing 的关系

相关实现：

- [`src/components/FeedbackSurvey/useFeedbackSurvey.tsx`](../src/components/FeedbackSurvey/useFeedbackSurvey.tsx)
- [`src/components/FeedbackSurvey/submitTranscriptShare.ts`](../src/components/FeedbackSurvey/submitTranscriptShare.ts)
- [`src/screens/REPL.tsx`](../src/screens/REPL.tsx)

### 5.1 transcript sharing 是一条正式产品路径

`submitTranscriptShare.ts` 里定义了：

```typescript
export type TranscriptShareTrigger =
  | 'bad_feedback_survey'
  | 'good_feedback_survey'
  | 'frustration'
  | 'memory_survey'
```

注意这里明确有一个 trigger 叫 **`frustration`**。

这不是猜测，而是正式枚举值。

### 5.2 bad/good survey 已经是显式链路

在 `useFeedbackSurvey.tsx` 里，当用户选了 `good` 或 `bad`，会进一步触发 transcript prompt，并在同意后调用：

```typescript
const result = await submitTranscriptShare(messagesRef.current, trigger_0, appearanceId_2);
```

也就是说，产品已经有成熟的“反馈 -> transcript upload”机制。

既然 `TranscriptShareTrigger` 里已经有 `frustration`，再结合 REPL 中明确存在的 `useFrustrationDetection(...)`，就能推断：

- `matchesNegativeKeyword()` 很可能是 frustration signal 的早期轻量输入之一
- 更高层的 frustration detection 再决定是否弹出 transcript sharing

### 5.3 transcript sharing 上传的内容很完整

这也解释了为什么产品会关心 frustration。

`submitTranscriptShare()` 上传的数据不仅包含：

- normalize 后的 transcript

还包含：

- subagent transcripts
- 原始 JSONL transcript（有大小保护）
- trigger
- version
- platform

说明这不是“点一下不满意”这么简单，而是为了复盘一整段出问题的会话。

## 6. 它没有做什么：避免过度解读

这一节很重要，因为这段代码很容易被解读过头。

### 6.1 它不是内容审查

从调用链上看，它不会：

- 屏蔽脏话
- 拒绝请求
- 替换用户输入
- 触发安全策略

如果它是 moderation 或 safety classifier，调用点不会只是 `logEvent(...)`。

### 6.2 它不是模型提示增强

这段布尔值没有被注入 prompt，也没有进入 tool permission 分支。

所以它不是在告诉模型：

- “用户现在很生气”
- “请更加谨慎”
- “请切换更安抚的回复风格”

至少从当前源码证据看，没有这层用途。

### 6.3 它不是完整的 frustration detection 系统

它只是一个轻量输入级 heuristic。

真正的 frustration detection 很可能还会结合：

- 会话消息序列
- 近期 assistant 输出
- 是否已有 survey 在显示
- 当前是否有 active prompt

从 REPL 里 `useFrustrationDetection(messages, isLoading, hasActivePrompt, ...)` 这一签名就能看出来，它显然比单个正则复杂得多。

## 7. 为什么用正则而不是模型分类

从工程角度讲，这里用正则是合理的。

### 7.1 这是实时输入路径

`processTextPrompt()` 处在用户提交 prompt 的主链路上。

如果这里还要做模型推理来判断“你是不是不高兴”，会带来：

- 额外延迟
- 额外成本
- 隐私面扩大
- 结果不稳定

而正则可以做到零成本近实时。

### 7.2 产品只需要粗信号，不需要语义完美

这里想解决的问题不是 NLP 精准分类，而是：

“这个用户输入里是不是很明显地带有负面情绪词？”

对于这种粗粒度问题，正则足够了。

### 7.3 它与 `matchesKeepGoingKeyword()` 形成对照

同文件里还有：

```typescript
export function matchesKeepGoingKeyword(input: string): boolean {
  ...
  const keepGoingPattern = /\b(keep going|go on)\b/
  return keepGoingPattern.test(lowerInput)
}
```

这说明这个文件本身的定位就是：

**把用户 prompt 中少量高价值意图，提取成简单标签。**

一个标签表示负面情绪，
一个标签表示继续执行意图。

这是一种非常典型的 product instrumentation 设计。

## 8. 用伪代码还原整条产品链路

下面给出一个更接近源码语义的伪代码。

### 8.1 输入打标层

```text
on user text prompt:
    userPromptText = extractText(input)

    isNegative = matchesNegativeKeyword(userPromptText)
    isKeepGoing = matchesKeepGoingKeyword(userPromptText)

    logEvent("tengu_input_prompt", {
        is_negative: isNegative,
        is_keep_going: isKeepGoing
    })

    continue normal query flow
```

### 8.2 更高层的 frustration 链路

```text
during REPL runtime:
    frustrationDetection = useFrustrationDetection(messages, ...)

    if frustrationDetection decides user looks frustrated:
        show feedback survey / transcript share prompt

    if user agrees:
        submitTranscriptShare(messages, trigger="frustration", ...)
```

### 8.3 整体结构图

```text
用户输入文本
   |
   v
matchesNegativeKeyword()
   |
   +--> is_negative = true/false
   |
   v
logEvent('tengu_input_prompt', { is_negative })
   |
   v
正常进入 query / 会话
   |
   v
更高层 frustration detection
   |
   +--> 如果识别到用户明显 frustrated
           |
           v
      FeedbackSurvey / TranscriptSharePrompt
           |
           v
      submitTranscriptShare(trigger='frustration')
```

## 9. 这套设计的优点与代价

### 9.1 优点

1. **简单**
   只靠本地正则就能拿到一个很有价值的产品信号。

2. **便宜**
   零额外模型成本，零额外网络请求。

3. **足够稳定**
   对“明显带脏话或抱怨语气”的输入，匹配结果可预测。

4. **方便统计**
   `is_negative` 很适合作为漏斗、留存、失败会话、反馈转化率的分层维度。

### 9.2 代价

1. **召回有限**
   用户可能表达不满但不用这些词，例如“这完全不对”，就未必会命中。

2. **误报存在**
   某些带引号、代码、引用上下文中的词也可能触发。

3. **语言覆盖差**
   这套正则几乎只覆盖英语口语和脏话。

4. **不能代替真正的会话级判断**
   单条输入中的负面词，不一定等于用户对整段会话真实 frustrated。

## 10. 本章小结

源码里出现 `matchesNegativeKeyword()`，本质上不是因为 Claude Code 想“审查用户措辞”，而是因为它需要一个低成本、实时的**负面体验信号**。

从代码证据看，可以明确确认三点：

1. 这个函数当前直接用于 `processTextPrompt()` 中的埋点打标
2. 它不会阻断输入，也不会直接改变模型行为
3. 它和更高层的 frustration / transcript sharing 产品链路在概念上是连贯的

所以，这段函数存在的真正原因可以概括成一句话：

**Claude Code 需要尽早识别“用户是不是正在骂它、是不是已经烦了”，并把这类信号变成产品分析与反馈收集体系的一部分。**
