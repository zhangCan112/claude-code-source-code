# 06 - 终端 UI：Ink + React 渲染体系

## 1. 概述

Claude Code 的终端 UI 是工程史上最复杂的 React 终端应用之一。它没有使用标准的命令行库（如 inquirer），而是**自定制了整个 Ink 渲染引擎**（48个文件），用 React 组件树驱动终端输出。

这个选择的代价是巨大的工程复杂度，但换来的是：
- 实时流式文本渲染（逐字显示 LLM 响应）
- 并行工具进度的动态布局
- 虚拟滚动（万行消息不卡顿）
- Vim 模式的完整文本编辑体验

---

## 2. 自定制 Ink 引擎（src/ink/）

### 2.1 为什么定制 Ink

标准 Ink 库面向简单终端应用（几个 Box + Text）。Claude Code 需要：

| 需求 | 标准 Ink | 定制 Ink |
|------|---------|---------|
| 鼠标事件 | 不支持 | 支持（点击、选择） |
| 文本选择 | 不支持 | 支持（可视化选择） |
| 焦点管理 | 简单 Tab | 完整焦点链 |
| 虚拟滚动 | 不支持 | 支持（万行不卡顿） |
| Flexbox 布局 | 基础 | 完整 Yoga 布局 |
| 快速键输入 | 单键 | 完整 key sequence 解析 |

### 2.2 架构层次

```
React 组件树
    │
    ▼
React Reconciler（ink/reconciler.ts）
    │ 将 React 虚拟 DOM 转换为终端元素树
    ▼
布局引擎（ink/layout/）
    │ Yoga Flexbox 引擎计算每个元素的位置和大小
    ▼
渲染器（ink/render-to-screen.ts）
    │ 将布局结果转换为 ANSI 转义序列
    ▼
终端输出（ink/terminal.ts）
    │ 管理终端的原始输入/输出流
    ▼
终端 I/O（ink/termio.ts）
    │ 处理 stdin/stdout 的原始模式
    ▼
用户终端（xterm/iTerm2/Windows Terminal...）
```

### 2.3 关键组件

**布局组件**（`ink/components/`）：
- `Box` — Flexbox 容器（类似 HTML div）
- `Text` — 文本渲染（支持颜色、加粗、斜体等）
- `Button` — 可聚焦按钮
- `Newline` — 换行
- `Spacer` — 弹性空白

**输入处理**（`ink/hooks/`）：
- `useInput` — 键盘输入 Hook
- `useStdin` — 原始标准输入访问
- `useSelection` — 文本选择状态
- `useAnimationFrame` — 动画帧（类似浏览器 requestAnimationFrame）

**文本处理**：
- `wrap-text.ts` — 终端宽度感知的文本换行
- `wrapAnsi.ts` — ANSI 转义序列安全的换行
- `parse-keypress.ts` — 完整的终端按键序列解析器
- `stringWidth.ts` — 终端字符宽度计算（处理 CJK 宽字符）

---

## 3. 主题与设计系统（src/components/design-system/）

### 3.1 主题提供者

```
ThemeProvider
  ├── 颜色方案（浅色/深色跟随系统）
  ├── 语义颜色（primary, success, warning, error）
  └── 字体样式（monospace, bold, italic）
```

### 3.2 核心组件

| 组件 | 文件 | 用途 |
|------|------|------|
| `ThemedBox` | `ThemedBox.tsx` | 主题感知的容器 |
| `ThemedText` | `ThemedText.tsx` | 主题感知的文本 |
| `Dialog` | `Dialog.tsx` | 模态对话框（权限请求、确认） |
| `FuzzyPicker` | `FuzzyPicker.tsx` | 模糊搜索选择器 |
| `Tabs` | `Tabs.tsx` | 标签页切换 |
| `ProgressBar` | `ProgressBar.tsx` | 进度条 |

---

## 4. PromptInput 系统（src/components/PromptInput/）

### 4.1 架构

`PromptInput`（21 个文件）是用户输入的核心组件——Claude Code 的"聊天框"：

```
PromptInput
  ├── 输入区域
  │     ├── 多行文本编辑
  │     ├── Vim 模式支持（src/vim/）
  │     ├── 语法高亮（Markdown）
  │     ├── Tab 补全（文件路径、命令）
  │     └── 图片粘贴处理
  │
  ├── 建议系统
  │     ├── 斜杠命令补全
  │     ├── 文件路径补全
  │     ├── 工具名称补全
  │     └── 历史搜索（Ctrl+R）
  │
  ├── 状态指示器
  │     ├── 模型选择器
  │     ├── 权限模式指示器
  │     ├── 语音模式指示器
  │     └── 排队命令预览
  │
  └── 底部面板
        ├── 任务面板
        ├── 团队面板
        ├── Bridge 状态
        └── 模式切换
```

### 4.2 Vim 模式

`src/vim/`（5 个文件）实现了完整的 Vim 编辑模式：

```
motions.ts     → h,j,k,l,w,b,e,0,$,G,g...
operators.ts   → d,c,y,p,x...
textObjects.ts → iw,aw,i",a",i(,a(...
transitions.ts → 状态机 (normal → insert → visual → command)
types.ts       → 类型定义
```

### 4.3 虚拟滚动

`useVirtualScroll`（`src/hooks/useVirtualScroll.ts`）实现消息列表的虚拟化：
- 只渲染可见区域的消息（±buffer）
- 滚动位置跟踪
- 自动滚到底部（新消息到达时）
- 滚动时暂停后台刷新（`markScrollActivity()` 在 `bootstrap/state.ts:798`）

---

## 5. 消息渲染系统（src/components/messages/）

### 5.1 消息类型与渲染映射

每种 `Message` 类型都有对应的渲染组件：

| 消息类型 | 渲染组件 | 文件 |
|---------|---------|------|
| 助手文本 | `AssistantTextMessage` | `AssistantTextMessage.tsx` |
| 工具调用 | `UserToolResultMessage` | `UserToolResultMessage/index.tsx` |
| 压缩边界 | `CompactBoundaryMessage` | `CompactBoundaryMessage.tsx` |
| 工具组 | `GroupedToolUseContent` | `GroupedToolUseContent.tsx` |
| 系统消息 | `SystemMessage` | `SystemMessage.tsx` |
| 进度 | 进度 Spinner | `Spinner.tsx` |

### 5.2 工具结果渲染

每个工具定义了自己的 `renderToolResultMessage()`，提供定制化的结果展示：
- `BashTool`：终端输出 + 错误高亮
- `FileEditTool`：统一 Diff 视图
- `GrepTool/GlobTool`：搜索结果列表
- `AgentTool`：子代理摘要

---

## 6. 权限对话框（src/components/permissions/）

30 个文件实现了所有工具类型的权限请求 UI：

```
权限请求流程:
  useCanUseTool(context, tool, input)
    │
    ├── 自动批准（白名单命令/规则匹配）
    │     └── 直接 ALLOW（无 UI）
    │
    ├── 自动拒绝（deny 规则匹配）
    │     └── 直接 DENY + 错误消息
    │
    └── 需要用户确认
          └── 渲染对应的权限对话框
                ├── BashPermissionRequest     → "Allow command: rm -rf dir?"
                ├── FileEditPermissionRequest → "Allow edit: file.ts?"
                ├── WebFetchPermissionRequest → "Allow fetch: url?"
                ├── AgentPermissionRequest    → "Allow agent: sub-task?"
                └── ...30+ 种权限对话框
```

---

## 7. 输出优化

### 7.1 ANSI 优化器

`ink/optimizer.ts` 实现终端输出的增量更新：
- 比较前后两帧的输出差异
- 只发送变化的行（使用 ANSI 光标移动）
- 避免全屏刷新的闪烁

### 7.2 Scroll Drain

`bootstrap/state.ts:792-824` 的 `scrollDraining` 机制：
- 滚动期间暂停后台刷新（`markScrollActivity()`）
- 滚动停止 150ms 后恢复正常刷新
- 避免滚动和后台更新竞争终端输出

---

## 8. Logo 与品牌（src/components/LogoV2/）

15 个文件实现品牌展示：
- 欢迎屏幕（ASCII Art Logo + 动画）
- Channel 通知
- 版本更新提示

---

## 9. 设计精髓

### React 在终端的优雅适配

将 React 的声明式 UI 模式引入终端是一个大胆的架构选择。它的核心洞察是：**终端 UI 本质上也是一个渲染目标**——就像浏览器渲染 DOM，终端渲染 ANSI 字符。React 的组件模型、状态管理、Hooks 体系在终端场景同样适用。

### 虚拟滚动的必要性

在长时间会话中，消息列表可能达到数千条。直接渲染会导致：
- 终端输出缓冲区溢出
- 布局计算时间随消息数线性增长
- 按键响应延迟

虚拟滚动确保了无论会话多长，渲染性能始终恒定。

### Vim 模式的完整性

Claude Code 的 Vim 模式不是简单的 hjkl 映射，而是实现了完整的 Vim 状态机（normal → insert → visual → command-line）。这体现了对开发者体验的极致追求——让 Vim 用户在 AI 助手中也能保持肌肉记忆。
