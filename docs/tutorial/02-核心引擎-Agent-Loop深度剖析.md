# 02 - 核心引擎：Agent Loop 深度剖析

## 1. Agent Loop 概述

### 什么是 Agent Loop

Agent Loop 是现代 AI Agent 系统的核心执行模式，遵循 **LLM 调用 → 工具执行 → 结果回传 → 循环** 的经典范式。每一轮迭代中，大语言模型分析当前上下文、决定是否调用工具，工具执行完毕后将结果注入上下文，模型再次推理，直到产出最终回答。

### Claude Code 的实现规模

Claude Code 的 Agent Loop 核心实现在 `src/query.ts` 中，该文件约 **1729 行**，是整个项目中最大的单文件模块。其设计并非简单的 while 循环，而是一个融合了流式处理、并发工具调度、多级上下文压缩、模型降级回退、Hook 系统等机制的精密状态机。围绕它构建的支撑模块包括：

| 模块 | 行数 | 职责 |
|------|------|------|
| `src/query.ts` | 1729 | Agent Loop 主循环 |
| `src/QueryEngine.ts` | 1295 | 会话生命周期管理 |
| `src/services/tools/StreamingToolExecutor.ts` | 530 | 流式工具并发执行器 |
| `src/services/tools/toolOrchestration.ts` | 188 | 工具编排与分区 |
| `src/services/tools/toolExecution.ts` | 1460+ | 单工具执行流水线 |
| `src/services/tools/toolHooks.ts` | 650 | 工具前后置 Hook |
| `src/services/api/withRetry.ts` | 822 | API 重试策略 |

### 与其他 Agent 框架的设计差异

与 LangChain 的链式组合（Chain）或 AutoGPT 的递归子任务不同，Claude Code 采用了 **AsyncGenerator 驱动的单循环状态机**：

- **LangChain**：通过 Chain/Agent 抽象层层嵌套，每次调用返回完整结果
- **AutoGPT**：基于独立进程和文件系统通信，循环粒度粗
- **Claude Code**：`query()` 是一个 `AsyncGenerator`，每一轮迭代实时 `yield` 流式事件，调用者（如 `QueryEngine`）可以逐事件消费，实现真正的端到端流式体验

## 2. query() 函数深度解析

### 函数签名和 AsyncGenerator 设计

```typescript
// src/query.ts:219
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
>
```

`query()` 返回一个 `AsyncGenerator`，其 yield 类型涵盖流式事件、消息、墓碑消息（用于 UI 消除）和工具摘要。Terminal 返回值标记循环终止原因。这种设计使得上层消费者（`QueryEngine.submitMessage()`）可以用 `for await...of` 逐事件处理，实现 **流式响应的端到端贯穿**。

核心参数 `QueryParams`（`src/query.ts:181`）包含消息数组、系统提示词、工具权限回调、上下文信息等，其中 `deps` 字段支持依赖注入，便于测试。

### 主循环结构

```
// src/query.ts:307
while (true) { ... }
```

`queryLoop()`（`src/query.ts:241`）采用 `while(true)` 无限循环，通过以下方式退出：

| 退出路径 | 位置 | 触发条件 |
|---------|------|---------|
| `return { reason: 'completed' }` | `:1357` | 模型 `end_turn` 且无工具调用 |
| `return { reason: 'aborted_*' }` | `:1051`, `:1515` | 用户中断 |
| `return { reason: 'blocking_limit' }` | `:647` | Token 超限阻塞 |
| `return { reason: 'model_error' }` | `:996` | API 不可恢复错误 |
| `return { reason: 'max_turns' }` | `:1711` | 达到最大轮次 |
| `continue`（状态更新） | 多处 | 工具执行后继续下一轮 |

每次 `continue` 都会重新构建 `State` 对象（`src/query.ts:204`），携带消息、上下文、压缩追踪等跨迭代状态。

### 消息规范化

`normalizeMessagesForAPI()` 负责将内部 `Message[]` 转换为符合 Anthropic API 规范的消息格式。核心处理包括：

- 确保 `tool_use` 和 `tool_result` 严格配对
- 过滤不需要发送给 API 的消息类型（如 attachment、progress）
- 处理 thinking block 的签名保留规则（参见 `src/query.ts:151-163` 的 "rules of thinking" 注释）

### stop_reason 处理分支

流式响应完成后，`query()` 根据不同状态走不同路径：

**① tool_use → 工具执行**

当模型输出包含 `tool_use` 块时，`needsFollowUp` 置为 true（`src/query.ts:834`），循环进入工具执行阶段（`src/query.ts:1363`）。

**② end_turn → 正常结束**

无工具调用时，进入停止 Hook 处理（`src/query.ts:1267`），检查 Stop Hook 是否阻止继续，最终 `return { reason: 'completed' }`。

**③ max_output_tokens → 上下文压缩触发**

当模型输出被截断（`isWithheldMaxOutputTokens`，`src/query.ts:175`），错误被暂缓 yield，进入恢复流程：

```
escalate（8k→64k，:1199）→ recovery（续写消息，:1223）→ MAX_OUTPUT_TOKENS_RECOVERY_LIMIT(3次)
```

**④ prompt_too_long → 被暂缓处理**

`isWithheldPromptTooLong` 机制（`src/query.ts:801-810`）暂缓向 SDK 消费者暴露错误，先尝试 context collapse drain 或 reactive compact 恢复。

**⑤ refusal → 拒绝处理**

`getErrorMessageIfRefusal()`（`src/services/api/errors.ts:1184`）生成合规提示消息。

### 流式事件处理

`deps.callModel()` 返回的事件流遵循 Anthropic SDK 的流式协议：

```
message_start → content_block_start → content_block_delta[] → content_block_stop → ... → message_delta → message_stop
```

关键处理逻辑：
- 每个 `content_block_stop` 对应一个 `assistant` 消息的 yield
- `tool_use` 类型的 content block 被收集到 `toolUseBlocks` 数组
- 若启用 `StreamingToolExecutor`，工具在流式接收阶段即开始排队执行

### 错误处理与重试逻辑

API 层的重试由 `withRetry`（`src/services/api/withRetry.ts:170`）封装：

```
withRetry() AsyncGenerator
  ├── 529 过载：最多 3 次 → FallbackTriggeredError（触发模型降级）
  ├── 429 限速：指数退避 + 抖动（BASE_DELAY_MS * 2^attempt）
  ├── 401/403：刷新 OAuth token 后重试
  ├── 连接错误：重试（ECONNRESET 时禁用 keep-alive）
  └── 不可恢复错误：抛出 CannotRetryError
```

在 `query()` 层面（`src/query.ts:894`），`FallbackTriggeredError` 触发模型降级：清除已有 assistant 消息、切换到 fallback model、重新发起 API 调用。

### 自动压缩(autoCompact)触发条件

每次循环迭代开始时（`src/query.ts:454`），`deps.autocompact()` 检查是否需要压缩：

```
autoCompactIfNeeded() [autoCompact.ts:241]
  ├── 熔断器：连续失败 >= 3 次 → 跳过
  ├── shouldAutoCompact() → tokenCount >= threshold (effectiveWindow - 13000)
  ├── trySessionMemoryCompaction() → 优先尝试会话记忆压缩
  └── compactConversation() → LLM 摘要压缩
```

### Turn 限制和预算检查

- **maxTurns**：`src/query.ts:1705` 检查 `nextTurnCount > maxTurns`
- **tokenBudget**：`src/query.ts:1308` 调用 `checkTokenBudget()`，基于 90% 阈值和收益递减检测（连续 3 轮 delta < 500 tokens）决定是否继续
- **USD 预算**：`QueryEngine` 层面检查 `getTotalCost() >= maxBudgetUsd`（`src/QueryEngine.ts:972`）

## 3. QueryEngine 生命周期

### submitMessage() 方法

`QueryEngine.submitMessage()`（`src/QueryEngine.ts:209`）是 SDK/Headless 模式的核心入口。一次完整的调用流程：

```
1. setCwd(cwd)                          // 设置工作目录
2. fetchSystemPromptParts()              // 构建系统提示词
3. processUserInput(prompt)              // 处理用户输入（含斜杠命令）
4. mutableMessages.push(...messages)     // 追加新消息
5. recordTranscript(messages)            // 持久化到会话文件
6. yield buildSystemInitMessage(...)     // 发射系统初始化消息
7. for await (message of query({...}))   // 驱动 Agent Loop
8.   → normalizeMessage(message)         // 标准化后 yield 给 SDK
9. yield { type: 'result', ... }         // 发射最终结果
```

### 会话持久化：mutableMessages 数组

`mutableMessages`（`src/QueryEngine.ts:186`）是会话的内存状态，跨 turn 累积所有消息。每次 `query()` yield 的 assistant/user/compact_boundary 消息都被 push 到此数组，并异步写入 transcript 文件（`recordTranscript`）。

关键优化：compact boundary 触发后（`src/QueryEngine.ts:927`），`mutableMessages` 被 splice 截断，释放压缩前的消息供 GC 回收。

### 中断处理

`abortController`（`src/QueryEngine.ts:203`）通过 `createAbortController()` 创建，传递到 `query()` 内部的每一层：

- 用户调用 `engine.interrupt()`（`:1158`）触发 `abort()`
- `query()` 检测到 abort 后：先 consume `StreamingToolExecutor.getRemainingResults()`（生成合成错误块），再 yield 中断消息并返回

### ask() 便捷函数

`ask()`（`src/QueryEngine.ts:1186`）是对 `QueryEngine` 的一次性封装，每次调用创建新的 engine 实例，适用于 `-p` 模式等单次查询场景。它通过 `setReadFileCache` 将文件状态缓存回传给调用者。

## 4. StreamingToolExecutor 详解

### 设计模式：观察者 + 状态机

`StreamingToolExecutor`（`src/services/tools/StreamingToolExecutor.ts:40`）是一个融合了并发控制和有序输出 guarantee 的执行器。每个工具在生命周期中经历四种状态：

```
queued → executing → completed → yielded
```

核心数据结构 `TrackedTool`（`:21`）跟踪每个工具的状态、并发安全分类、结果缓冲和进度消息。

### addTool() 方法：工具排队与并发分类

```typescript
// src/services/tools/StreamingToolExecutor.ts:76
addTool(block: ToolUseBlock, assistantMessage: AssistantMessage): void
```

当流式响应中收到 `tool_use` block 时立即调用。核心步骤：

1. 查找工具定义，不存在则立即生成错误结果
2. 解析输入参数，调用 `tool.isConcurrencySafe(parsedInput.data)` 判断并发安全性
3. 创建 `TrackedTool` 条目（status: `queued`）
4. 触发 `processQueue()` 尝试立即执行

### 并发安全工具 vs 独占工具的分区策略

```typescript
// src/services/tools/StreamingToolExecutor.ts:129
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}
```

规则：
- **无工具执行中**：任何工具可启动
- **有并发安全工具执行中**：只有并发安全工具可并行加入
- **有独占工具执行中**：不允许任何新工具启动

这保证了独占工具（如 Bash、文件写入）永远不会与其他工具并发执行。

### 结果缓冲和顺序发射

`getCompletedResults()`（`:412`）按工具添加顺序遍历，只有前面的工具都已 yielded 后，后续工具的结果才会被发射。这确保了工具结果的全局有序性，同时不阻塞并发安全工具的实际执行。

Progress 消息不受此约束——它们通过 `pendingProgress` 数组独立缓冲，可以立即 yield。

### discard() 方法：流式回退清理

`discard()`（`:69`）在流式回退（streaming fallback）时调用，将所有排队/执行中的工具标记为废弃。执行中的工具会收到合成的 `streaming_fallback` 错误消息，避免孤儿 `tool_result` 污染上下文。

### 子进程清理机制

当 Bash 工具错误时（`:359`），`siblingAbortController` 被触发（`abort('sibling_error')`），所有正在运行的子进程收到信号立即终止。这避免了 `mkdir` 失败后后续命令继续执行的无意义行为。

## 5. 工具编排(toolOrchestration)

### partitionToolCalls() 算法

```typescript
// src/services/tools/toolOrchestration.ts:91
function partitionToolCalls(
  toolUseMessages: ToolUseBlock[],
  toolUseContext: ToolUseContext,
): Batch[]
```

`reduce` 遍历工具调用列表，产出 `Batch[]`：

```
输入: [Read(a), Read(b), Bash(c), Read(d), Bash(e)]
输出: [
  { isConcurrencySafe: true,  blocks: [Read(a), Read(b)] },
  { isConcurrencySafe: false, blocks: [Bash(c)] },
  { isConcurrencySafe: true,  blocks: [Read(d)] },
  { isConcurrencySafe: false, blocks: [Bash(e)] },
]
```

关键规则：连续的并发安全工具合并为一个批次，不安全的工具各自独占一个批次。

### Batch 分区与并发控制

```typescript
// src/services/tools/toolOrchestration.ts:30
for (const { isConcurrencySafe, blocks } of partitionToolCalls(...)) {
  if (isConcurrencySafe) {
    yield* runToolsConcurrently(blocks, ...)   // 并行执行
  } else {
    yield* runToolsSerially(blocks, ...)        // 串行执行
  }
}
```

- **并行批次**：使用 `all()` 工具函数（`src/utils/generators.js`），受 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` 环境变量控制（默认 10）
- **串行批次**：逐个执行，每个工具的 context modifier 在执行后立即应用到 `currentContext`

上下文修改器（`contextModifier`）在并发批次中延迟应用（`toolOrchestration.ts:54-62`），在串行批次中立即应用，避免了并发工具间的上下文竞争。

## 6. Claude API 交互层

### 流式调用实现

`queryModelWithStreaming()`（`src/services/api/claude.ts`）封装了与 Anthropic API 的流式交互：

```
消息构建:
  normalizeMessagesForAPI()  → API 格式消息
  toolToAPISchema()          → 工具 schema 转换
  prependUserContext()       → 注入用户上下文
  appendSystemContext()      → 追加系统上下文

API 调用:
  client.beta.messages.create({
    stream: true,
    messages: [...],
    tools: [...],
    system: fullSystemPrompt,
    model: currentModel,
    ...
  })
```

流式事件通过 `for await (const event of stream)` 逐个处理，每个 `content_block_delta` 被包装为 `StreamEvent` yield 给调用者。

### 错误分类与重试策略

`errors.ts`（`src/services/api/errors.ts:965`）的 `classifyAPIError()` 将错误分为 20+ 种类型：

| 类型 | 条件 | 处理策略 |
|------|------|---------|
| `rate_limit` | 429 | 指数退避重试 |
| `server_overload` | 529 | 最多 3 次后降级 |
| `prompt_too_long` | 400/413 + "too long" | 触发压缩 |
| `auth_error` | 401/403 | 刷新凭证重试 |
| `tool_use_mismatch` | 400 + tool_use/tool_result | 日志 + 用户恢复提示 |

`withRetry`（`src/services/api/withRetry.ts:170`）是 API 调用的通用重试包装器，作为 `AsyncGenerator` 实现，在重试等待期间 yield `SystemAPIErrorMessage`，让上层感知重试状态。

对于持久会话（`CLAUDE_CODE_UNATTENDED_RETRY`），重试无限进行，退避上限 5 分钟，每 30 秒 yield 心跳消息防止会话被标记为空闲。

### API 超时和中断处理

`AbortSignal` 贯穿整个调用链：`QueryEngine.abortController` → `query()` → `deps.callModel()` → SDK `stream`。中断时：
1. SDK 抛出 `APIUserAbortError`
2. `withRetry` 捕获并终止重试循环
3. `query()` 进入 abort 清理流程

## 7. 上下文压缩(Context Compaction)

### 为什么需要压缩

LLM 的上下文窗口有限（200k tokens），而 Agent Loop 每轮迭代都会累积消息。Claude Code 采用三层压缩策略，在 token 预算耗尽前主动管理上下文大小。

### tokenBudget.ts 的预算计算逻辑

```typescript
// src/query/tokenBudget.ts:45
function checkTokenBudget(tracker, agentId, budget, globalTurnTokens): TokenBudgetDecision
```

核心逻辑：
- **跳过条件**：子 agent（有 agentId）或无预算设置 → `stop`
- **继续条件**：未达 90% 阈值 且 非收益递减 → `continue`（附带 nudge 消息）
- **收益递减检测**：连续 3 轮 delta < 500 tokens → `stop`
- **完成事件**：包含统计数据用于分析

### autoCompact.ts 的自动触发条件

```typescript
// src/services/compact/autoCompact.ts:72
getAutoCompactThreshold(model) = effectiveContextWindow - 13,000
```

当 `tokenCountWithEstimation(messages) >= threshold` 时触发。额外保护：
- **熔断器**：连续失败 3 次后停止尝试（`:70`）
- **递归保护**：`compact`/`session_memory` querySource 跳过（避免死锁）
- **特征门控**：reactive compact 和 context collapse 模式下自动抑制主动压缩

### compact.ts 的核心压缩算法

`compactConversation()`（`src/services/compact/compact.ts`）执行完整的压缩流程：

```
1. stripImagesFromMessages()           // 移除图片减少 token
2. analyzeContext()                    // 分析上下文结构
3. executePreCompactHooks()            // 前置 Hook
4. 构建压缩 prompt → 调用 LLM 生成摘要
5. 构建后压缩消息:
   - compactBoundary（系统消息标记边界）
   - summaryMessage（摘要内容）
   - attachments（关键文件、计划、技能）
   - hookResults（Hook 产物）
6. executePostCompactHooks()           // 后置 Hook
```

压缩保留的关键信息（`POST_COMPACT_*` 常量）：
- 最多 5 个文件，每个 5000 tokens
- 技能预算 25000 tokens，每个 5000 tokens
- 计划文件内容

### microCompact.ts 的微压缩策略

微压缩（`src/services/compact/microCompact.ts`）是更轻量的优化，在每轮 API 调用前执行：

- **时间衰减**：旧工具结果被替换为 `[Old tool result content cleared]`
- **可压缩工具范围**：Read、Bash、Grep、Glob、WebSearch、WebFetch、Edit、Write
- **Cache 编辑**（Ant 内部）：通过 `cache_edits` 块直接在 API 侧删除缓存内容，避免重新发送

### 压缩后的消息替换机制

压缩完成后，`buildPostCompactMessages()` 构建新的消息数组（`src/query.ts:528`），包含：
1. 压缩边界消息（compact boundary）
2. 摘要消息
3. 附件消息（文件、Hook 结果等）

`query()` 通过 `messagesForQuery = postCompactMessages` 替换原有消息（`:535`），循环继续使用压缩后的上下文。

## 8. 依赖注入与配置

### query/deps.ts 的依赖注入设计

```typescript
// src/query/deps.ts:21
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

4 个核心依赖通过 `QueryParams.deps` 注入，测试可直接传入 mock。`productionDeps()`（`:33`）返回使用真实实现的生产工厂。这种设计避免了 `spyOn` 跨模块的脆弱性，6-8 个测试文件共享同一套 mock 接口。

### query/config.ts 的配置项

`QueryConfig`（`src/query/config.ts:15`）在 `query()` 入口处一次性快照：

```typescript
{
  sessionId: SessionId,
  gates: {
    streamingToolExecution: boolean,  // Statsig feature gate
    emitToolUseSummaries: boolean,    // env var 控制
    isAnt: boolean,                   // 内部用户标记
    fastModeEnabled: boolean,         // 快速模式开关
  }
}
```

注意：`feature()` 门控不包含在此配置中——它们是 tree-shaking 边界，必须在代码中内联使用以保证死代码消除。

### stopHooks.ts 的轮次结束钩子

`handleStopHooks()`（`src/query/stopHooks.ts:65`）在每轮结束时执行：

```
executeStopHooks()                    // 执行用户定义的 Stop Hook
  ↓
TeammateIdle hooks（如果是 teammate） // 团队协作场景
  ↓
TaskCompleted hooks（如果有进行中任务）
  ↓
executePromptSuggestion()             // 后台生成下轮建议
  ↓
extractMemories()                     // 后台提取记忆
  ↓
executeAutoDream()                    // 后台自动思考
```

大多数后台任务以 fire-and-forget 方式执行，不阻塞主循环。

## 9. 端到端数据流图

```
用户输入 (prompt)
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│ QueryEngine.submitMessage()                             │
│   processUserInput() → 斜杠命令处理                      │
│   mutableMessages.push() → 会话状态更新                   │
│   recordTranscript() → 持久化                            │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│ query() / queryLoop() — while(true)                     │
│                                                         │
│  ┌───────────────────────────────────────────────┐      │
│  │ 预处理阶段                                     │      │
│  │  applyToolResultBudget() → 工具结果预算裁剪     │      │
│  │  snipCompactIfNeeded()  → 历史裁剪             │      │
│  │  microcompactMessages() → 微压缩               │      │
│  │  applyCollapsesIfNeeded()→ 上下文折叠           │      │
│  │  autoCompactIfNeeded()  → 自动压缩             │      │
│  └───────────────────────┬───────────────────────┘      │
│                          │                               │
│                          ▼                               │
│  ┌───────────────────────────────────────────────┐      │
│  │ API 调用阶段                                   │      │
│  │  deps.callModel() → withRetry() → Anthropic   │      │
│  │    ├── message_start                           │      │
│  │    ├── content_block_delta (text/tool_use)     │      │
│  │    ├── StreamingToolExecutor.addTool() ←── 异步 │      │
│  │    ├── StreamingToolExecutor.getCompletedResults│      │
│  │    └── message_stop                            │      │
│  └───────────────────────┬───────────────────────┘      │
│                          │                               │
│               ┌──────────┴──────────┐                   │
│               │                     │                    │
│         needsFollowUp         !needsFollowUp             │
│               │                     │                    │
│               ▼                     ▼                    │
│  ┌─────────────────────┐  ┌──────────────────┐         │
│  │ 工具执行阶段         │  │ 结束处理          │         │
│  │                     │  │                  │          │
│  │ partitionToolCalls()│  │ handleStopHooks()│          │
│  │  ├─ safe → 并行     │  │ checkTokenBudget()│         │
│  │  └─ unsafe → 串行   │  │ return completed │          │
│  │                     │  └──────────────────┘          │
│  │ runToolUse()        │                                │
│  │  ├─ PreToolUse Hooks│                                │
│  │  ├─ 权限检查        │                                │
│  │  ├─ tool.call()     │                                │
│  │  └─ PostToolUse Hooks│                               │
│  └─────────┬───────────┘                                │
│            │                                             │
│            ▼                                             │
│  ┌───────────────────────┐                               │
│  │ 后处理阶段             │                               │
│  │  附件消息收集          │                               │
│  │  内存预取消费          │                               │
│  │  技能发现注入          │                               │
│  │  maxTurns 检查        │                               │
│  │  state = next         │                               │
│  │  continue ←────────── │─── 回到 while(true) 顶部      │
│  └───────────────────────┘                               │
└─────────────────────────────────────────────────────────┘
```

## 10. 设计精髓提炼

### AsyncGenerator 的优雅之处

`query()` 作为 `AsyncGenerator` 是整个架构最精妙的设计选择。它实现了 **生产者-消费者的解耦**：

- **生产端**：`query()` 内部可以 yield 任何粒度的事件（流式 token、工具进度、压缩边界）
- **消费端**：`QueryEngine` 用 `for await...of` 消费，可以选择性地 yield 给 SDK、记录到 transcript、或忽略
- **错误传播**：`throw` 自然沿 generator 链向上传播，`return` 和 `.return()` 也能正确处理

这种模式比回调（callback）更可控，比 Promise 数组更实时，是 Agent 系统中流式处理的理想原语。

### 流式处理的端到端贯穿

从 API 的 SSE 事件到工具的进度消息，再到 SDK 的流式输出，整个管道是 **零缓冲** 的：

1. API stream 事件 → `query()` yield `StreamEvent`
2. `StreamingToolExecutor` 在流式接收时即开始工具执行
3. 工具进度通过 `pendingProgress` 立即 yield
4. `QueryEngine` 选择性转发到 SDK 消费者

这意味着用户在 CLI 中看到的是真正的实时输出，而非批量刷新。

### 工具并发调度的精巧设计

`StreamingToolExecutor` 和 `partitionToolCalls()` 的组合实现了 **安全且高效的并发**：

- 安全校验：通过 `isConcurrencySafe()` 对每个工具实例级别分类（而非工具类型级别），因为同一工具在不同参数下可能有不同的并发安全性
- 有序保证：结果按工具添加顺序发射，即使实际执行是并发的
- 级联取消：Bash 错误触发 `siblingAbortController`，清理所有兄弟进程
- 流式回退：`discard()` 机制确保模型降级时不会有孤儿工具结果

### 上下文压缩的时机选择

三层压缩形成梯度策略：

| 层级 | 触发时机 | 开销 | 效果 |
|------|---------|------|------|
| microCompact | 每次 API 调用前 | 极低（字符串替换） | 清理旧工具结果 |
| autoCompact | token > 阈值（~93%） | 中（LLM 调用） | 生成对话摘要 |
| reactiveCompact | API 返回 prompt-too-long | 高（压缩+重试） | 紧急恢复 |

关键洞察：**压缩时机选择在 API 调用前而非后**。`autoCompactIfNeeded()` 在 `deps.callModel()` 之前执行（`src/query.ts:454`），这意味着如果压缩成功，模型直接使用精简上下文，避免了先触发 API 错误再恢复的昂贵往返。reactive compact 作为兜底，只在主动压缩失败时才介入。

这种 **预防优于治疗** 的设计哲学贯穿了整个 Agent Loop，使得 Claude Code 在长时间会话中保持了稳定的上下文管理能力。
