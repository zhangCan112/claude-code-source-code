# 05 - 服务层：API / 压缩 / MCP / 分析

## 1. 概述

`src/services/` 是 Claude Code 的业务逻辑核心，包含 36 个子目录，覆盖 API 交互、上下文压缩、MCP 协议、工具执行、遥测分析、插件管理、OAuth 认证、LSP 集成、语音模式等关键子系统。

```
services/
├── api/          (20文件) Claude API 流式客户端
├── analytics/    (9文件)  遥测分析 + Feature Flags
├── compact/      (11文件) 上下文压缩
├── mcp/          (23文件) Model Context Protocol 集成
├── tools/        (4文件)  工具执行引擎
├── plugins/      (3文件)  插件管理
├── oauth/        (5文件)  OAuth 2.0 认证
├── lsp/          (7文件)  LSP 集成
├── voice/        (3文件)  语音模式
├── autoDream/    (2文件)  后台自动思考
├── SessionMemory/(3文件)  会话记忆
├── extractMemories/(2文件) 记忆提取
├── PromptSuggestion/     推测建议
├── MagicDocs/            文档处理
└── ...其他
```

---

## 2. Claude API 客户端（services/api/）

### 2.1 流式调用架构

`services/api/claude.ts`（3,419行）是与 Anthropic API 交互的核心。它将 Claude SDK 的流式接口封装为 `AsyncGenerator`：

```
消息构建流水线:
  normalizeMessagesForAPI()  → 移除 UI-only 字段，配对 tool_use/tool_result
  toolToAPISchema()          → Zod Schema → JSON Schema 转换
  prependUserContext()       → 注入 CLAUDE.md、日期等用户上下文
  appendSystemContext()      → 追加 git status、分支等系统上下文
  buildSystemPrompt()        → 组装完整系统提示词

API 调用:
  client.beta.messages.create({
    stream: true,
    messages: [...],
    tools: [...],
    system: fullSystemPrompt,
    model: currentModel,
    thinking: thinkingConfig,    // thinking/reasoning 配置
    betas: [...betaHeaders],     // API beta 特性标记
    metadata: { userId, sessionId }
  })
```

### 2.2 Beta Headers 系统

API 调用携带多个 Beta Headers（`src/constants/betas.ts`），启用实验性特性：

| Beta Header | 功能 |
|-------------|------|
| `prompt-caching-2024-07-31` | 提示词缓存（减少 token 消耗） |
| `max-tokens-3-5-sonnet-2024-07-15` | 扩展输出 token 限制 |
| `interleaved-thinking-2025-05-14` | 交错思考模式 |
| `code-execution-2025-05-22` | 代码执行能力 |
| `computer-use-2025-01-24` | 计算机使用能力 |

### 2.3 错误分类与重试

`services/api/errors.ts` 实现 20+ 种错误分类：

```
classifyAPIError(error)
  ├── 429 Rate Limit     → 指数退避 + 抖动重试
  ├── 529 Overload       → 最多 3 次后触发 FallbackTriggeredError（模型降级）
  ├── 400 Prompt Too Long → 触发上下文压缩
  ├── 401/403 Auth Error  → 刷新 OAuth token 后重试
  ├── 400 Tool Mismatch   → 日志 + 用户恢复提示
  ├── ECONNRESET          → 重试（禁用 keep-alive）
  └── 其他                → CannotRetryError（不可恢复）
```

`withRetry`（`services/api/withRetry.ts:170`）封装重试逻辑为 AsyncGenerator，重试等待期间 yield 心跳消息（`SystemAPIErrorMessage`），让上层感知重试状态。

### 2.4 模型降级

当主模型持续 529 过载时，`FallbackTriggeredError` 触发模型降级链（`src/query.ts:894`）：

```
claude-sonnet-4-20250514
    ↓ 529 过载
claude-sonnet-4-20250514（重试 3 次）
    ↓ 仍然 529
FallbackTriggeredError → 切换到 fallback 模型
```

---

## 3. 上下文压缩（services/compact/）

### 3.1 三层压缩策略

```
┌──────────────────────────────────────────────────────┐
│ Layer 1: MicroCompact（每次 API 调用前）                │
│ 触发：每轮 API 调用前自动执行                           │
│ 策略：字符串替换（旧工具结果 → 占位符）                   │
│ 开销：极低（无 LLM 调用）                               │
│ 效果：清理旧 Read/Bash/Grep 等工具的冗长输出              │
├──────────────────────────────────────────────────────┤
│ Layer 2: AutoCompact（token > 阈值）                   │
│ 触发：tokenCount >= effectiveWindow - 13,000           │
│ 策略：LLM 生成对话摘要                                  │
│ 开销：中等（一次 LLM 调用）                              │
│ 效果：完整对话压缩为摘要，释放上下文空间                    │
├──────────────────────────────────────────────────────┤
│ Layer 3: ReactiveCompact（API 返回 prompt-too-long）   │
│ 触发：API 返回 400 + "prompt is too long"               │
│ 策略：紧急压缩 + 重试                                   │
│ 开销：高（压缩 + 额外 API 调用）                         │
│ 效果：从不可恢复错误中恢复                                │
└──────────────────────────────────────────────────────┘
```

### 3.2 AutoCompact 详细流程

```
autoCompactIfNeeded()（autoCompact.ts:72）
  │
  ├── 熔断器检查：连续失败 >= 3 次 → 跳过
  ├── 递归保护：compact/session_memory querySource → 跳过
  │
  └── shouldAutoCompact()
      │ tokenCount >= effectiveWindow - 13,000
      │
      ├── trySessionMemoryCompaction()
      │     ├── 提取关键信息为"会话记忆"
      │     └── 更轻量，优先尝试
      │
      └── compactConversation()
            ├── stripImagesFromMessages()      // 移除图片
            ├── analyzeContext()                // 分析上下文结构
            ├── executePreCompactHooks()        // 前置 Hook
            ├── 构建压缩 prompt → 调用 LLM      // 生成摘要
            ├── buildPostCompactMessages()       // 构建后压缩消息
            │     ├── compactBoundary（系统标记）
            │     ├── summaryMessage（摘要）
            │     ├── attachments（关键文件、计划、技能）
            │     └── hookResults（Hook 产物）
            └── executePostCompactHooks()        // 后置 Hook
```

### 3.3 压缩保留策略

压缩后保留的关键信息（POST_COMPACT 常量）：
- 最多 5 个文件，每个 5000 tokens
- 技能预算 25000 tokens，每个 5000 tokens
- 计划文件内容完整保留
- 已调用的 Skill 内容保留（防止压缩丢失技能上下文）

### 3.4 Token 预算管理

`src/query/tokenBudget.ts` 实现 Token 预算决策：

```typescript
function checkTokenBudget(tracker, agentId, budget, globalTurnTokens): TokenBudgetDecision

// 决策逻辑：
// - 子 agent 或无预算 → stop
// - 未达 90% 阈值 → continue（附 nudge 消息）
// - 收益递减（连续 3 轮 delta < 500 tokens）→ stop
```

---

## 4. MCP 协议集成（services/mcp/）

### 4.1 Model Context Protocol 概述

MCP 是 Anthropic 推出的标准协议，让 AI 模型能与外部工具和数据源交互。Claude Code 作为 MCP 客户端，可以连接任意 MCP 服务器。

### 4.2 连接管理

`MCPConnectionManager`（`services/mcp/MCPConnectionManager.tsx`）管理所有 MCP 连接：

```
支持的传输类型：
  ├── stdio     → 本地子进程（最常见）
  ├── SSE       → HTTP Server-Sent Events
  ├── HTTP      → HTTP 流式
  ├── WebSocket → WS 连接
  └── SDK       → 进程内 SDK 传输（无网络）

生命周期：
  配置加载 → 连接建立 → 工具发现 → 运行时调用 → 关闭清理
```

### 4.3 工具发现与注册

MCP 服务器的工具通过 `MCPTool` 包装为标准 `Tool` 接口：

```
MCP 服务器声明工具（JSON Schema）
    ↓
MCPTool 适配器：
  ├── inputSchema → Zod Schema 转换
  ├── name → mcp__serverName__toolName 前缀
  ├── call() → 转发到 MCP 服务器
  ├── isConcurrencySafe → 依赖服务器声明
  ├── isMcp = true → 标记为 MCP 工具
  └── mcpInfo → { serverName, toolName }
```

### 4.4 认证

`services/mcp/auth.ts` 实现 MCP 服务器的 OAuth 认证：
- 支持标准 OAuth 2.0 授权码流程
- `xaa.ts` 实现 Cross-App Access 认证
- `channelPermissions.ts` / `channelAllowlist.ts` 管理通道权限

### 4.5 Elicitation

`services/mcp/elicitationHandler.ts` 处理 MCP 服务器的用户提示请求（elicitation）——MCP 服务器可以请求用户输入，Claude Code 展示对话框并回传结果。

---

## 5. 遥测分析（services/analytics/）

### 5.1 双 Sink 管道

```
事件产生
  │
  ├── 1st-party Sink（Anthropic 官方）
  │     ├── OpenTelemetry + Protobuf
  │     ├── POST api.anthropic.com/api/event_logging/batch
  │     ├── 批量发送（最多 200 事件，10s 刷新）
  │     └── 失败持久化到 ~/.claude/telemetry/ + 指数退避重试
  │
  └── 3rd-party Sink（Datadog）
        ├── 64 种批准的事件类型
        └── POST http-intake.logs.us5.datadoghq.com
```

### 5.2 GrowthBook Feature Flags

`services/analytics/growthbook.ts` 实现 Feature Flags 系统：
- 从 GrowthBook CDN 拉取特性配置
- 支持用户级别定向（订阅层级、地区等）
- 运行时特性开关（无需更新代码）
- `feature()` 编译时检查 + GrowthBook 运行时检查 的双层模式

### 5.3 环境指纹

每个事件携带环境指纹（`services/analytics/metadata.ts`）：
- 平台、架构、Node 版本、终端类型
- 包管理器、CI/CD 检测、WSL 版本
- VCS 类型、版本字符串
- 进程指标（uptime、rss、heap、CPU）
- 用户追踪（model、session ID、user ID、device ID）

---

## 6. 工具执行引擎（services/tools/）

### 6.1 StreamingToolExecutor

已在 02 篇详解。核心职责：
- 流式接收 tool_use 块时即开始排队执行
- 并发安全/独占分区
- 结果按序发射
- 错误级联取消

### 6.2 toolOrchestration

已在 02 篇详解。核心职责：
- `partitionToolCalls()` 分区算法
- 并行批次（`runToolsConcurrently`）vs 串行批次（`runToolsSerially`）
- 上下文修改器的延迟/立即应用

### 6.3 toolExecution

单个工具的完整执行流水线：
```
validateInput()
  → PreToolUse Hooks
  → checkPermissions()
  → canUseTool()（交互式权限对话框）
  → tool.call()
  → PostToolUse Hooks
  → 结果格式化
```

### 6.4 toolHooks

工具执行前后的 Hook 集成：
- `PreToolUse`：可以在工具执行前修改输入或阻止执行
- `PostToolUse`：可以在工具执行后处理结果

---

## 7. 插件管理（services/plugins/）

### 7.1 PluginInstallationManager

`services/plugins/PluginInstallationManager.ts` 管理插件生命周期：
- 发现 → 安装 → 启用 → 运行 → 禁用 → 卸载
- 支持本地插件目录和 Marketplace 插件
- 插件可以贡献：工具、命令、Hook、MCP 服务器、输出样式

### 7.2 插件类型

`src/types/plugin.ts` 定义 `LoadedPlugin` 类型：
```typescript
type LoadedPlugin = {
  name: string
  rootDir: string
  tools?: Tool[]
  commands?: Command[]
  hooks?: HookMatcher[]
  mcpServers?: MCPServerConfig[]
  outputStyles?: OutputStyleConfig[]
}
```

---

## 8. OAuth 认证（services/oauth/）

```
OAuth 2.0 流程:
  1. 生成 code_verifier + code_challenge（PKCE）
  2. 打开浏览器 → Anthropic 授权页面
  3. auth-code-listener.ts 监听本地回调
  4. 交换 authorization_code → access_token + refresh_token
  5. token 存储到系统 keychain
  6. 自动刷新（jwtUtils.ts 管理 token 生命周期）
```

支持多种认证方式：
- **OAuth**（Claude Pro/Max 订阅）
- **API Key**（直接 API key）
- **AWS Bedrock**（`src/utils/auth.ts` 中 Bedrock 配置）
- **Google Vertex AI**（GCP 认证）

---

## 9. LSP 集成（services/lsp/）

```
LSP 架构:
  LSPServerManager（生命周期管理）
    ├── LSPServerInstance（每个语言一个实例）
    │     ├── 启动 LSP 服务器进程
    │     └── 管理 LSP 协议通信
    ├── LSPClient（协议客户端）
    │     ├── initialize → initialized
    │     ├── textDocument/didOpen
    │     └── textDocument/diagnostics
    └── LSPDiagnosticRegistry（诊断集合管理）
          └── 实时诊断反馈 → 工具提示给 LLM
```

支持的语言服务器：TypeScript、Python、Go、Rust 等（通过插件配置）。

---

## 10. 服务间依赖关系

```
                    ┌─────────────┐
                    │   query()   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌────────────┐
        │ API 层   │ │ 压缩层   │ │ 工具执行层  │
        │ api/     │ │ compact/ │ │ tools/     │
        └────┬─────┘ └──────────┘ └─────┬──────┘
             │                           │
             ▼                           ▼
        ┌──────────┐              ┌────────────┐
        │ 遥测层   │              │ MCP 层     │
        │analytics/│              │ mcp/       │
        └──────────┘              └────────────┘
             │                           │
             ▼                           ▼
        ┌──────────┐              ┌────────────┐
        │GrowthBook│              │ 插件层     │
        │ Feature  │              │ plugins/   │
        │ Flags    │              └────────────┘
        └──────────┘                    │
                                        ▼
                                  ┌────────────┐
                                  │ LSP 层     │
                                  │ lsp/       │
                                  └────────────┘

横向依赖：
  api/ → analytics/（每次 API 调用记录遥测）
  compact/ → api/（压缩需要 LLM 调用）
  tools/ → mcp/（MCP 工具执行）
  plugins/ → mcp/（插件贡献 MCP 服务器）
  oauth/ → api/（认证用于 API 调用）
```

---

## 11. 设计精髓

### 预防优于治疗

三层压缩形成梯度防线：MicroCompact 在每轮自动清理旧数据（预防），AutoCompact 在接近阈值时主动压缩（早期干预），ReactiveCompact 作为最后的紧急恢复（治疗）。压缩时机选择在 API 调用前（`src/query.ts:454`），避免了先触发 API 错误再恢复的昂贵往返。

### 双 Sink 的务实选择

1st-party sink 用于产品分析（Anthropic 了解使用模式），3rd-party Datadog 用于运维监控（错误率、延迟）。这种分离确保了产品数据和运维数据的独立采集和处理。

### MCP 的开放生态

MCP 协议让 Claude Code 的工具系统从"封闭的 40+ 内置工具"扩展为"无限的 MCP 生态"。任何实现了 MCP 协议的服务器都可以作为工具源——数据库查询、API 调用、文件系统操作、浏览器自动化等。`MCPTool` 适配器将异构的 MCP 工具统一为标准 `Tool` 接口，对 Agent Loop 完全透明。
