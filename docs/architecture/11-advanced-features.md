# 11 - 高级功能模块深度分析

> 本文档补充分析 Claude Code CLI 中之前架构文档未覆盖的 12 个重要模块，涵盖计算机操作控制、推测执行、后台记忆整理、安全分类器、定时任务等高级功能。

---

## 目录

1. [Computer Use -- 计算机操作控制](#1-computer-use----计算机操作控制)
2. [Speculation -- 推测执行](#2-speculation----推测执行)
3. [AutoDream -- 后台记忆整理](#3-autodream----后台记忆整理)
4. [MagicDocs -- 魔法文档](#4-magicdocs----魔法文档)
5. [Classifier Approvals -- 安全分类器](#5-classifier-approvals----安全分类器)
6. [Conversation Recovery -- 对话恢复](#6-conversation-recovery----对话恢复)
7. [Deep Link -- 深度链接](#7-deep-link----深度链接)
8. [Teleport -- 远程环境传送](#8-teleport----远程环境传送)
9. [Code Attribution -- 代码归属追踪](#9-code-attribution----代码归属追踪)
10. [Cron -- 定时任务系统](#10-cron----定时任务系统)
11. [Secure Storage -- 安全存储](#11-secure-storage----安全存储)
12. [Context Collapse -- 上下文折叠](#12-context-collapse----上下文折叠)

---

## 1. Computer Use -- 计算机操作控制

**目录**: `src/utils/computerUse/` (15 个文件)

### 概述

Computer Use 是 Claude Code 的桌面控制功能，允许 AI 通过程序化方式操控用户的 macOS 桌面：截屏、移动鼠标、点击、键盘输入、拖拽等。该功能内部代号 **Chicago**，当前仅支持 macOS，依赖两个原生模块：

- **`@ant/computer-use-input`** (Rust/enigo) -- 鼠标和键盘控制、获取前台应用信息
- **`@ant/computer-use-swift`** -- SCContentFilter 截图、NSWorkspace 应用管理、TCC 权限检查

核心包 `@ant/computer-use-mcp` 定义了 `ComputerExecutor` 接口和 MCP 工具协议，CLI 实现了该接口的完整 CLI 变体。

### 特性门控 (`gates.ts`)

门控配置通过 GrowthBook 特性标志 `tengu_malort_pedway` 进行远程控制：

```
ChicagoConfig = {
  enabled: boolean           // 总开关
  pixelValidation: boolean   // 点击前像素校验
  clipboardPasteMultiline    // 剪贴板粘贴多行
  mouseAnimation: boolean    // 拖拽动画
  hideBeforeAction: boolean  // 操作前隐藏其他窗口
  autoTargetDisplay: boolean // 自动选择目标显示器
  clipboardGuard: boolean    // 剪贴板保护
  coordinateMode: 'pixels'   // 坐标模式（冻结在首次读取）
}
```

关键约束：
- 需要 Max 或 Pro 订阅 (`hasRequiredSubscription()`)
- Anthropic 内部员工可绕过订阅检查但有 monorepo 环境检测
- 坐标模式在首次读取后冻结，防止 mid-session 配置翻转导致坐标不一致

### 执行器 (`executor.ts`)

`createCliExecutor()` 是工厂函数，返回完整的 `ComputerExecutor` 实例。与桌面版 Cowork 的关键差异（CLI deltas）：

| 差异 | 原因 |
|------|------|
| 无 `withClickThrough` | CLI 是终端，无窗口需要穿透 |
| 终端作为代理宿主 | 通过 `getTerminalBundleId()` 检测宿主终端，在截图和窗口管理中排除 |
| 剪贴板通过 `pbcopy`/`pbpaste` | 无 Electron clipboard 模块 |

核心操作实现模式：

- **CFRunLoop 泵** (`drainRunLoop.ts`): Swift 的 `@MainActor` 方法和 Rust 的 `key()`/`keys()` 都向 `DispatchQueue.main` 派发。在 libuv 下（Node/bun）主队列永远不会排空，Promise 会挂起。`drainRunLoop()` 通过引用计数的 1ms `setInterval` 调用 `_drainMainRunLoop()` 来泵送 CFRunLoop。30 秒超时保底。

- **鼠标动画** (`animatedMove`): 仅用于拖拽，60fps ease-out-cubic，距离比例时长，上限 0.5s。

- **剪贴板粘贴** (`typeViaClipboard`): 保存剪贴板 -> 写入文本 -> 回读校验 -> Cmd+V -> 等待 100ms -> 恢复剪贴板。整个过程在 `finally` 中保护。

- **修饰键安全** (`withModifiers`): 按键追踪哪些修饰键实际按下，`finally` 反序释放，避免卡键。

### 会话锁 (`computerUseLock.ts`)

使用 `O_EXCL` 原子创建锁文件实现跨进程独占。同一时间只有一个 Claude 会话可以操控计算机。PID 存活检测用于回收崩溃会话遗留的锁。

### MCP 服务器 (`mcpServer.ts`, `setup.ts`)

Computer Use 工具通过 MCP 协议暴露。`mcp__computer-use__*` 工具名会触发 API 后端注入 CU 可用性提示到系统提示中。`setup.ts` 构建动态 MCP 配置，将所有 CU 工具加入 `allowedTools` 绕过常规权限弹窗。

### 清理 (`cleanup.ts`)

每轮结束时：
1. 取消隐藏被 `prepareForAction` 隐藏的应用（5 秒超时）
2. 注销 ESC 热键
3. 释放会话锁，发送系统通知 "Claude is done using your computer"

> **Agent 开发启示**: Computer Use 的架构展示了如何安全地将底层系统控制暴露给 AI。关键设计模式包括：(1) 多层门控（远程配置 + 订阅级别 + 环境检测）；(2) 会话级独占锁防止并发冲突；(3) `finally` 块保证所有资源释放（按键、剪贴板、隐藏窗口）；(4) 通过 MCP 协议标准化工具暴露。对于构建 Agent 系统时集成外部 native 能力，这是一个完整的参考实现。

---

## 2. Speculation -- 推测执行

**文件**: `src/services/PromptSuggestion/speculation.ts`, `src/state/AppStateStore.ts`

### 概述

推测执行（Speculation）是 Claude Code 的性能优化功能：当 AI 向用户给出建议（prompt suggestion）后，在用户思考期间，系统**提前**以该建议为输入运行一个分叉 Agent，执行只读操作和安全的文件编辑。如果用户接受建议，已完成的工作可以直接注入对话，节省等待时间。

当前仅对 Anthropic 内部用户 (`USER_TYPE === 'ant'`) 启用。

### 状态机

```
AppState.speculation: SpeculationState
  | { status: 'idle' }
  | { status: 'active',
      id: string,              // 8字符 UUID
      abort: () => void,       // 中断函数
      messagesRef,             // 可变引用: 已产生的消息
      writtenPathsRef,         // 可变引用: overlay 中已写入的路径
      boundary: CompletionBoundary | null,  // 停止边界
      toolUseCount: number,
      isPipelined: boolean,    // 是否为流水线推测
      pipelinedSuggestion?,    // 流水线中的下一个建议
    }
```

### 执行流程

1. **触发**: `startSpeculation()` 在 prompt suggestion 生成后调用
2. **分叉 Agent**: 通过 `runForkedAgent()` 启动独立 Agent，模拟用户接受建议后的执行
3. **工具安全过滤** (`canUseTool` 回调):
   - **安全只读工具** (Read, Glob, Grep, LSP 等) -- 允许
   - **写工具** (Edit, Write) -- 需要权限模式 >= `acceptEdits`，写入 overlay 目录
   - **只读 Bash** -- 通过 `checkReadOnlyConstraints` 校验后允许
   - **其他工具** -- 拒绝，设置 boundary，中止推测
4. **Overlay 文件隔离**: 所有文件写操作重定向到 `$CLAUDE_TEMP/speculation/<pid>/<id>/` 目录。读操作对已写入的文件也重定向到 overlay（copy-on-write 语义）。
5. **接受** (`acceptSpeculation`): 将 overlay 文件复制回主工作区，清理消息（过滤 thinking 块、失败的工具调用、中断消息），注入到主对话。
6. **中止** (`abortSpeculation`): 用户输入新内容时触发，清理 overlay，重置状态。

### 流水线推测

当推测完整完成后，`generatePipelinedSuggestion()` 会基于推测结果立即生成下一个建议。如果用户接受第一个建议，流水线建议会被提升为当前建议，并启动新一轮推测 -- 形成连续的推测链。

### CompletionBoundary 类型

推测在以下边界停止：
- `bash`: 遇到非只读 Bash 命令
- `edit`: 遇到文件编辑但权限不足
- `denied_tool`: 遇到不支持的工具
- `complete`: 推测完整执行完毕

### 限制

- 最多 20 轮对话、100 条消息
- 只能写 cwd 内的文件
- 不支持非只读 Bash、WebFetch、Agent 等工具

> **Agent 开发启示**: 推测执行是 Agent 系统的一个重要优化方向。关键洞察包括：(1) 使用 overlay 目录实现 copy-on-write 文件隔离，让推测性写操作可安全回滚；(2) 分叉 Agent 的 `canUseTool` 回调实现了细粒度的工具安全过滤，是沙箱策略的优雅体现；(3) 流水线推测展示了如何将 AI 的空闲等待时间转化为有效工作。实操建议：在任何需要"乐观执行"的 Agent 场景中，overlay + canUseTool 过滤的模式都值得借鉴。

---

## 3. AutoDream -- 后台记忆整理

**目录**: `src/services/autoDream/` (4 个文件)

### 概述

AutoDream 是后台记忆整理机制（代号 "Dream"），在用户正常使用 Claude Code 期间，定期在后台启动一个分叉子 Agent，执行 `/dream` 流程 -- 回顾近期会话记录，将新学到的知识整理进持久化记忆文件。

### 三层门控（cheapest first）

`autoDream.ts` 的 `runAutoDream()` 按成本从低到高依次检查：

1. **时间门控**: `readLastConsolidatedAt()` 读取锁文件的 mtime（一次 stat），检查距上次整理是否超过 `minHours`（默认 24 小时）
2. **会话数门控**: `listSessionsTouchedSince()` 扫描项目目录下的会话文件，过滤当前会话，检查是否有 >= `minSessions`（默认 5）个会话有更新。**扫描节流**: 10 分钟内不重复扫描。
3. **锁门控**: `tryAcquireConsolidationLock()` 原子写入 PID 到锁文件，PID 存活校验，二次读取校验（防竞争）

### 锁机制 (`consolidationLock.ts`)

锁文件位于记忆目录下 `.consolidate-lock`，其 **mtime 就是 lastConsolidatedAt**。

- 获取锁: 写入 PID -> 二次读取校验 PID 归属 -> 返回旧 mtime（供回滚用）
- 成功完成: mtime 停留在获取时刻（= now），无需额外操作
- 失败回滚: `rollbackConsolidationLock(priorMtime)` 重置 mtime 到获取前的值
- 崩溃恢复: mtime 卡住但 PID 已死 -> 下个进程可回收（1 小时超时保底）

### 整理 Prompt (`consolidationPrompt.ts`)

`buildConsolidationPrompt()` 生成四阶段指令：

| 阶段 | 行为 |
|------|------|
| Phase 1 -- Orient | ls 记忆目录，读取入口文件，浏览已有主题文件 |
| Phase 2 -- Gather | 从日志、现有记忆、会话 transcript 中搜集新信息 |
| Phase 3 -- Consolidate | 合并新信号到现有主题文件，转换相对日期，删除矛盾事实 |
| Phase 4 -- Prune & Index | 更新入口索引文件，保持在行数和大小限制内 |

AutoDream 运行的子 Agent 的工具被限制为只读 Bash 和记忆文件的读写。

### 与 extractMemories 的区别

- **extractMemories**: 每轮结束后即时提取当前对话中的关键记忆片段（reactive，微粒度）
- **AutoDream**: 周期性回顾多个会话的积累，进行整合、去重、修正（proactive，宏观整理）

> **Agent 开发启示**: AutoDream 展示了如何为长期运行的 Agent 实现"反思"能力。关键设计要点：(1) 三层门控从廉价到昂贵依次评估，避免不必要的开销；(2) 利用文件 mtime 作为时间戳，免去额外的状态存储；(3) 锁的回滚机制确保失败不会阻塞后续尝试；(4) 子 Agent 的工具受限（只读 bash + 记忆文件编辑），是最小权限原则的体现。实操建议：对于任何需要"知识累积"的 Agent，定期的后台整理机制比纯粹的"追加"策略更有效。

---

## 4. MagicDocs -- 魔法文档

**目录**: `src/services/MagicDocs/` (2 个文件)

### 概述

Magic Docs 是一个自动文档维护系统。在代码库中标记了 `# MAGIC DOC: [标题]` 头部的 Markdown 文件会被自动跟踪，并在对话空闲时由后台子 Agent 自动更新，将对话中的新发现融入文档。

当前仅限 Anthropic 内部用户 (`USER_TYPE === 'ant'`)。

### 工作流程

1. **检测**: `registerFileReadListener` 监听所有文件读取事件。当读取的文件内容匹配 `^# MAGIC DOC: (.+)$` 时，将该文件路径注册到 `trackedMagicDocs` Map 中。
2. **触发**: `updateMagicDocs` 注册为 `postSamplingHook`，在每轮 API 采样完成后执行。仅在以下条件满足时更新：
   - 来源是 `repl_main_thread`
   - 最后一轮助手消息中**没有**工具调用（即对话处于空闲状态）
3. **更新**: 对每个跟踪的文档调用 `updateMagicDoc()`，通过 `runAgent()` 启动一个子 Agent：
   - Agent 类型 `magic-docs`，模型 `sonnet`
   - 仅允许使用 `Edit` 工具，且只能编辑该特定文件
   - 上下文包含当前完整对话（`forkContextMessages`）

### 自定义指令

文件头后的第一行如果是斜体文本（`_instructions_` 或 `*instructions*`），会被解析为文档专属的更新指令，优先级高于通用规则。

### Prompt 架构 (`prompts.ts`)

更新 Prompt 通过模板变量系统（`{{varName}}`）构建，支持用户自定义 prompt 文件 `~/.claude/magic-docs/prompt.md`。核心规则：

- 保持文档**当前状态**，不追加历史记录
- 就地更新信息，删除过时内容
- 高信号密度：聚焦架构、模式、入口点，避免重复代码中已显而易见的内容
- 头部和自定义指令行必须原样保留

> **Agent 开发启示**: Magic Docs 展示了"被动触发的后台 Agent"模式。关键设计：(1) 通过文件读取事件进行隐式注册，无需用户手动配置；(2) 空闲检测（无工具调用时才触发）避免干扰主流程；(3) 最小权限工具集（仅 Edit，仅目标文件）；(4) 支持用户自定义 prompt 模板，实现可扩展性。实操建议：在需要"活文档"的场景中，"检测标记 -> 空闲更新 -> 受限写入"的模式是安全且高效的。

---

## 5. Classifier Approvals -- 安全分类器

**文件**: `src/utils/classifierApprovals.ts`, `src/utils/classifierApprovalsHook.ts`, `src/utils/permissions/yoloClassifier.ts`, `src/utils/permissions/classifierShared.ts`

### 概述

Classifier Approvals 系统为 Bash 命令和 auto-mode（也称 YOLO mode）下的工具调用提供安全分类。系统有两种分类器，核心区别如下：

| 分类器 | 标识 | 机制 | 用途 |
|--------|------|------|------|
| Bash Classifier | `bash` | 规则匹配 + 语义分类 | 对 Bash 命令进行安全性分类 |
| YOLO/Auto-Mode Classifier | `auto-mode` | LLM side-query | auto-mode 下判断工具调用是否安全 |

### Bash Classifier

特性门控: `BASH_CLASSIFIER`。`bashClassifier.ts` 提供语义匹配的允许/拒绝规则描述。分类结果通过 `setClassifierApproval(toolUseID, matchedRule)` 存储。

### YOLO/Auto-Mode Classifier

特性门控: `TRANSCRIPT_CLASSIFIER`。`yoloClassifier.ts` 实现了一个完整的 LLM-as-classifier：

- 使用 `sideQuery()` 发起独立 API 调用（非主对话流）
- 携带完整对话历史 + 系统提示 + 自定义权限规则
- 权限规则来自 `auto_mode_system_prompt.txt` + `permissions_external.txt`（或内部版本）
- 返回结构化的 `YoloClassifierResult`（通过 Zod schema 校验）
- 分类结果通过 `setYoloClassifierApproval(toolUseID, reason)` 存储

### UI 集成 (`classifierApprovalsHook.ts`)

React hook `useIsClassifierChecking(toolUseID)` 通过 `useSyncExternalStore` 订阅分类器状态变化，在工具执行 UI 中显示 "classifier checking" 状态。

### 状态管理

```typescript
// 全局 Map：toolUseID -> { classifier: 'bash'|'auto-mode', matchedRule?, reason? }
const CLASSIFIER_APPROVALS = new Map<string, ClassifierApproval>()
// 正在分类中的工具调用集合
const CLASSIFIER_CHECKING = new Set<string>()
```

信号系统（`createSignal`）实现了 React 外部存储的订阅模式。

> **Agent 开发启示**: 分类器系统展示了 Agent 安全决策的两种范式：(1) 规则匹配（快速、确定性、适合已知模式）；(2) LLM-as-classifier（灵活、处理长尾场景、但有延迟和成本）。在 auto-mode 下，为每个工具调用发起独立 LLM 分类请求的做法虽然开销大，但提供了对未知命令的零样本安全判断能力。实操建议：混合使用规则引擎（快路径）和 LLM 分类器（慢路径）是平衡安全与性能的有效策略。

---

## 6. Conversation Recovery -- 对话恢复

**文件**: `src/utils/conversationRecovery.ts`

### 概述

Conversation Recovery 负责在 `--continue` 和 `--resume` 场景下加载、反序列化和修复历史对话，使 Claude Code 能从崩溃、中断或正常退出后恢复之前的工作。

### 核心函数

**`loadConversationForResume(source, sourceJsonlFile)`** -- 统一入口：

- `source === undefined`: 加载最近会话（跳过活跃的 `--bg`/daemon 会话）
- `source` 是 string: 按 session ID 加载
- `sourceJsonlFile`: 按 JSONL 文件路径加载（跨目录恢复）

返回的结构包含：消息列表、中断状态、文件历史快照、attribution 快照、content 替换记录、上下文折叠快照、session 元数据（agent 名称、颜色、worktree、PR 信息等）。

**`deserializeMessagesWithInterruptDetection()`** -- 反序列化管线：

1. 迁移旧版 attachment 类型（`new_file` -> `file`）
2. 验证 `permissionMode` 值合法性
3. 过滤未解决的 tool_use（无匹配 tool_result）
4. 过滤孤立的 thinking-only 助手消息
5. 过滤纯空白助手消息
6. 检测中断类型并生成恢复策略

### 中断检测

`detectTurnInterruption()` 分析对话末端，识别三种状态：

| 最后消息类型 | 判定 | 恢复策略 |
|-------------|------|----------|
| assistant (非 API 错误) | `none` -- 正常完成 | 无需恢复 |
| user (tool_result, 非终端工具) | `interrupted_turn` | 注入 "Continue from where you left off." |
| user (纯文本) | `interrupted_prompt` | 保留用户消息，等待处理 |
| attachment | `interrupted_turn` | 注入继续消息 |

特殊处理：Brief mode 的 `SendUserMessage` 等终端工具的 tool_result 被正确识别为已完成轮次。

### Skill 状态恢复

`restoreSkillStateFromMessages()` 从 `invoked_skills` attachment 消息中恢复 skill 状态，确保 skill 在多次 compaction 后仍能存活。同时处理 `skill_listing` 去重。

> **Agent 开发启示**: 对话恢复是 Agent 持久化的核心挑战。关键设计：(1) 反序列化管线通过多层过滤器逐步修复数据一致性问题（孤立工具调用、空白消息等）；(2) 中断检测区分"用户消息未处理"和"Agent 执行被打断"两种场景并采用不同恢复策略；(3) 通过 session 元数据完整保存了会话的所有上下文（agent 设置、worktree、PR 信息等）。实操建议：为 Agent 实现恢复机制时，重点关注数据一致性修复和中断类型区分。

---

## 7. Deep Link -- 深度链接

**目录**: `src/utils/deepLink/` (6 个文件)

### 概述

Deep Link 系统实现了 `claude-cli://` 自定义 URL 协议，允许从浏览器、编辑器或其他应用中一键打开 Claude Code 并传递参数。

### URI 格式

```
claude-cli://open?q=<prompt>&cwd=<path>&repo=<owner/name>
```

参数均可选。`q` 预填充 prompt，`cwd` 设置工作目录，`repo` 通过 MRU 克隆路径查找本地仓库。

### 安全处理 (`parseDeepLink.ts`)

作为用户交互的入口，安全处理非常严格：

- ASCII 控制字符检测（防命令分隔符注入）
- Unicode 净化（`partiallySanitizeUnicode` 去除 ASCII smuggling）
- `repo` 格式白名单（`[\w.-]+/[\w.-]+`）
- `q` 长度限制 5000 字符、`cwd` 限制 4096 字符
- 绝对路径校验

### 协议注册 (`registerProtocol.ts`)

三平台实现：

| 平台 | 机制 |
|------|------|
| macOS | 在 `~/Applications` 创建 .app 跳板，Info.plist 包含 CFBundleURLTypes，二进制通过符号链接指向已签名的 claude |
| Linux | 创建 .desktop 文件 + `xdg-mime default` 注册 |
| Windows | 写入 `HKCU\Software\Classes\claude-cli` 注册表 |

`ensureDeepLinkProtocolRegistered()` 在每次会话启动时检查注册状态，自动修复（通过读取注册 artifact 而非缓存标志，确保跨机器和路径变更时自愈）。失败有 24 小时退避。

### 终端启动 (`terminalLauncher.ts`)

协议处理器在无 TTY 的无头上下文中运行（由 OS 直接启动），需要检测并启动用户的终端模拟器。

支持的终端：macOS 6 种（iTerm2, Ghostty, Kitty, Alacritty, WezTerm, Terminal.app），Linux 10 种，Windows 3 种。

关键安全区分：
- **纯 argv 路径**（Ghostty, Kitty, Linux 全部, Windows Terminal）-- 用户输入作为独立 argv 元素传递，无 shell 解释
- **Shell-string 路径**（iTerm2, Terminal.app, PowerShell, cmd.exe）-- 用户输入经过 `shellQuote()`/`psQuote()`/`cmdQuote()` 转义

### 终端偏好记忆 (`terminalPreference.ts`)

在交互式会话期间，从 `TERM_PROGRAM` 环境变量捕获终端偏好并持久化到全局配置，供后续无头启动时使用。

### 安全横幅 (`banner.ts`)

外部链接打开的会话显示警告横幅，提示用户：
- 当前工作目录（关系到加载的 CLAUDE.md）
- 如果有 repo slug，显示本地克隆和 FETCH_HEAD 新鲜度
- 如果 prompt 超过 1000 字符，提示用户"滚动查看完整 prompt"

> **Agent 开发启示**: Deep Link 是 Agent 与外部生态集成的重要通道。安全处理是最核心的设计考量：(1) 深度链接中的 prompt 必须在用户确认后才提交，不能自动执行；(2) 纯 argv 路径优于 shell-string 路径，因为前者的安全性由操作系统保证，后者依赖 quoting 正确性；(3) 三平台协议注册的差异化处理和自愈机制值得参考。实操建议：任何接受外部输入的 Agent 入口都需要类似的安全分层设计。

---

## 8. Teleport -- 远程环境传送

**目录**: `src/utils/teleport/` (4 个文件)

### 概述

Teleport 是 Claude Code 的远程会话（CCR -- Claude Code Remote）支持系统，允许将本地代码库"传送"到远程云端沙箱中执行。支持 Anthropic 托管云环境、BYOC (Bring Your Own Cloud) 和 Bridge 模式。

### API 层 (`api.ts`)

基于 Anthropic Sessions API (`/v1/sessions`)，主要操作：

- **`fetchCodeSessionsFromSessionsAPI()`**: 列出远程会话，将 `SessionResource` 转换为 `CodeSession` 格式
- **`sendEventToRemoteSession()`**: 向远程会话发送用户消息事件
- **`fetchSession()` / `updateSessionTitle()`**: 会话管理

认证要求 Claude.ai OAuth token + Organization UUID。所有请求携带 `anthropic-beta: ccr-byoc-2025-07-29` 头部。

内置重试机制（`axiosGetWithRetry`）：4 次重试，指数退避 2s/4s/8s/16s，仅重试瞬态网络错误和 5xx。

### Git Bundle 传送 (`gitBundle.ts`)

`createAndUploadGitBundle()` 将本地仓库打包为 git bundle 上传到 Files API，供远程沙箱初始化：

**三级回退策略**：
1. `--all`: 包含所有分支和标签
2. `HEAD`: 仅当前分支历史
3. `squashed`: 单次无父提交的树快照

每级在超过大小限制（默认 100MB，可通过 GrowthBook 配置）时降级到下一级。

**WIP（在途工作）捕获**：
- `git stash create` 创建一个悬空提交（不改变工作区或 refs/stash）
- 写入 `refs/seed/stash` 使其可达
- bundle 包含该 ref
- 清理：`finally` 块中删除 `refs/seed/stash` 和 `refs/seed/root`

### 环境管理 (`environments.ts`, `environmentSelection.ts`)

- **`fetchEnvironments()`**: 从 `/v1/environment_providers` 获取可用环境列表
- **`createDefaultCloudEnvironment()`**: 为无环境的用户创建默认 `anthropic_cloud` 环境
- **`getEnvironmentSelectionInfo()`**: 合并设置源优先级，确定选中的环境

环境类型：`anthropic_cloud`（Anthropic 托管）、`byoc`（用户自己的云）、`bridge`（桥接）

> **Agent 开发启示**: Teleport 展示了 Agent 从本地到远程的完整迁移路径。关键设计：(1) git bundle 的三级回退策略在大仓库场景下保证了可用性；(2) WIP 捕获通过 stash create（不修改工作区）实现了对未提交更改的无损打包；(3) 认证和环境选择的分层设计（OAuth + org UUID + 环境 provider）适合多租户场景。实操建议：远程 Agent 执行场景中，代码传送的大小控制和 WIP 保护是常见挑战，git bundle 方案是成熟的解法。

---

## 9. Code Attribution -- 代码归属追踪

**文件**: `src/utils/attribution.ts`

### 概述

Code Attribution 系统追踪 AI 对文件的修改贡献，生成 commit 和 PR 的归属信息。功能包括基于 `Co-Authored-By` 的 commit 署名、基于贡献比例的增强 PR 描述，以及 transcript 分析。

### Commit 归属 (`getAttributionTexts()`)

生成两种文本：
- **commit**: `Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>` -- 模型名动态获取，非内部仓库的未知模型回退到 "Claude Opus 4.6"
- **pr**: `Generated with Claude Code` + 产品链接

支持用户自定义（`settings.attribution.commit` / `settings.attribution.pr`）和向后兼容（`includeCoAuthoredBy`）。

远程模式下返回 session URL 作为归属。

### 增强 PR 归属 (`getEnhancedPRAttribution()`)

计算并展示详细的贡献统计：

```
Generated with Claude Code (93% 3-shotted by claude-opus-4-5, 2 memories recalled)
```

数据来源：
- **Claude 贡献百分比**: 从 `AppState.attribution.fileStates` 通过 `calculateCommitAttribution()` 计算
- **N-shotted**: 从 transcript 的 post-compaction 部分计数用户 prompt（排除 tool_result、terminal output、sidechain 消息）
- **memories recalled**: 从 transcript 计数针对记忆文件的 Read/Grep/Glob/Edit/Write 操作

### Transcript 分析 (`getTranscriptStats()`)

使用 `readTranscriptForLoad()` 高效读取 transcript（跳过占 84% 字节的 attribution-snapshot 行），仅分析最后一个 compact_boundary 之后的 entries。

对于内部仓库，还通过动态导入 `attributionTrailer.ts` 生成 git trailer 行，在 squash merge 时存活到最终的 squash commit 中。

> **Agent 开发启示**: 代码归属是 AI 辅助开发中的透明度保障。关键设计：(1) 区分 commit 和 PR 两种归属场景，粒度不同；(2) "N-shotted" 指标量化了人类参与度，是 Agent 自治程度的有意义度量；(3) 记忆访问计数揭示了 Agent 利用历史知识的程度；(4) 考虑了 squash merge 场景下归属信息的存活。实操建议：为 Agent 的输出附加归属元数据是负责任 AI 的重要实践。

---

## 10. Cron -- 定时任务系统

**文件**: `src/utils/cron.ts`, `src/utils/cronScheduler.ts`, `src/utils/cronTasks.ts`, `src/utils/cronTasksLock.ts`, `src/utils/cronJitterConfig.ts`

### 概述

Cron 系统提供了完整的定时任务调度能力，支持 5 字段 cron 表达式，任务存储在项目目录 `.claude/scheduled_tasks.json` 中。任务分为一次性（fire-once）和循环（recurring）两种。

### Cron 表达式解析 (`cron.ts`)

`parseCronExpression(expr)`: 解析标准 5 字段 cron（minute hour dayOfMonth month dayOfWeek），支持通配符、步进、范围、列表。所有时间解释为本地时区。

`computeNextCronRun(fields, from)`: 逐分钟前进查找下一匹配时间（上限 366 天）。遵循标准 cron 语义：当 dayOfMonth 和 dayOfWeek 同时受限时，**任一**匹配即可（OR 语义）。正确处理 DST 跳转。

`cronToHuman(cron, opts)`: 将 cron 表达式转为人类可读文本，支持 UTC->本地时区转换。

### 任务存储 (`cronTasks.ts`)

```typescript
type CronTask = {
  id: string         // 8字符 UUID
  cron: string       // 5字段 cron 表达式
  prompt: string     // 触发时注入的 prompt
  createdAt: number  // 创建时间戳
  lastFiredAt?: number  // 上次触发时间（循环任务）
  recurring?: boolean   // 是否循环
  permanent?: boolean   // 是否免于过期（assistant mode 内置任务）
  durable?: boolean     // 运行时：false=仅进程内存
  agentId?: string      // 运行时：所属 teammate
}
```

任务有两种存储方式：
- **File-backed** (durable): 写入 `.claude/scheduled_tasks.json`
- **Session-only** (non-durable): 仅在 `bootstrap/state.ts` 的进程内存中

### 调度器 (`cronScheduler.ts`)

`createCronScheduler(options)` 创建调度器实例，核心循环：

1. **启用轮询**: 等待 `setScheduledTasksEnabled(true)` 或检测到任务文件存在
2. **锁获取**: `tryAcquireSchedulerLock()` -- 同一项目目录只有一个调度器所有者
3. **文件监视**: chokidar 监视 `scheduled_tasks.json`，变化时重新加载
4. **1 秒检查循环** (`check()`): 遍历任务，到时触发

### Jitter 防雷群 (`cronJitterConfig.ts`)

防止大量用户在相同 wall-clock 时间（如整点）同时触发推理请求：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `recurringFrac` | 0.1 | 循环任务延迟 = 间隔 * frac |
| `recurringCapMs` | 15min | 循环延迟上限 |
| `oneShotMaxMs` | 90s | 一次性任务最大提前量 |
| `oneShotFloorMs` | 0 | 一次性任务最小提前量 |
| `oneShotMinuteMod` | 30 | 仅对 :00/:30 分钟 jitter |
| `recurringMaxAgeMs` | 7 天 | 循环任务自动过期 |

Jitter 值通过 `taskId` 的前 8 个十六进制字符确定性计算（稳定、均匀分布）。GrowthBook 远程配置支持运维人员在事故期间实时调整 jitter 参数。

### 锁机制 (`cronTasksLock.ts`)

与 computerUseLock 同模式：O_EXCL 原子创建、PID 存活检测、过期锁回收。非所有者每 5 秒探测一次锁。

### 错过任务处理

启动时扫描所有任务，对 `nextCronRunMs(cron, createdAt) < now` 的一次性任务：
1. 生成通知消息
2. 要求模型先用 `AskUserQuestion` 确认是否执行
3. 从文件中删除已通知的任务

> **Agent 开发启示**: Cron 系统展示了如何为 Agent 实现可靠的定时调度。关键设计：(1) Jitter 机制是大规模 Agent 部署的必要基础设施，防止推理服务过载；(2) 文件级锁 + PID 存活检测解决了多进程共享任务文件的竞争问题；(3) session-only 任务支持了临时性调度需求；(4) 错过任务的"先确认再执行"策略是 Agent 安全性的体现。实操建议：任何有定时触发需求的 Agent 系统都应考虑 jitter 和 missed-task 处理。

---

## 11. Secure Storage -- 安全存储

**目录**: `src/utils/secureStorage/` (6 个文件)

### 概述

Secure Storage 提供凭据和 token 的安全持久化存储，采用分层 fallback 策略。

### 存储层级

```
SecureStorage 接口
├── macOS: Keychain (primary) + PlainText (fallback)
├── Linux: PlainText（TODO: 计划添加 libsecret）
└── Windows: PlainText
```

### macOS Keychain 集成 (`macOsKeychainStorage.ts`)

通过 `security` 命令行工具与 macOS Keychain 交互：

- **读取**: `security find-generic-password -a <user> -w -s <service>`
- **写入**: 优先使用 `security -i`（stdin 模式，进程监控工具只看到 "security -i"，不暴露 payload）。payload 转 hex 编码。当 payload 超过 stdin 4096 字节缓冲区限制时回退到 argv 模式。
- **删除**: `security delete-generic-password`

缓存策略：
- TTL-based 缓存，避免频繁 subprocess spawn
- **Stale-while-error**: 读取失败时返回缓存的旧值（而非 null），防止单次瞬态故障导致 "Not logged in" 状态传播
- Async 读取去重：`readInFlight` promise 复用
- Keychain 锁定检测（SSH 场景，exit code 36），缓存进程生命周期

### Fallback 策略 (`fallbackStorage.ts`)

`createFallbackStorage(primary, secondary)` 的行为：

| 操作 | 逻辑 |
|------|------|
| read | primary -> secondary |
| update | primary 成功时删除 secondary（迁移）；primary 失败时写 secondary 并删除 primary（防 stale shadow） |
| delete | 两者都删 |

关键修复：当 primary 写入失败但 primary 仍持有旧数据时，`read()` 会优先返回 primary 的旧数据（shadow 问题），导致 refresh token 循环。解法是在 secondary 写入成功后删除 primary 的旧条目。

### PlainText 存储 (`plainTextStorage.ts`)

文件路径 `~/.claude/.credentials.json`，权限 `0o600`。作为最后的 fallback。

> **Agent 开发启示**: 安全存储是 Agent 认证系统的基础。关键设计：(1) Stale-while-error 缓存策略将可用性优先于一致性，在 Agent 场景中更合理；(2) stdin 模式避免在进程参数中暴露敏感数据（CrowdStrike 等安全工具会扫描进程参数）；(3) Fallback 写入时主动清理 primary 的旧数据防止 shadow 读取是一个值得注意的边界情况。实操建议：Agent 系统的凭据管理应优先使用操作系统原生安全存储，并实现完善的 fallback 和缓存策略。

---

## 12. Context Collapse -- 上下文折叠

**文件**: `src/utils/collapseBackgroundBashNotifications.ts`, `src/utils/collapseHookSummaries.ts`, `src/utils/collapseReadSearch.ts`, `src/utils/collapseTeammateShutdowns.ts`

### 概述

Context Collapse 是一组渲染优化工具，将连续的同类消息折叠为简洁的摘要，减少 UI 噪音。这些函数作用于渲染层（`RenderableMessage[]`），不改变底层对话数据。

### collapseBackgroundBashNotifications

**作用**: 将连续的已完成后台 Bash 任务通知折叠为 "N background commands completed"。

**条件**: 仅在全屏模式 + 非 verbose 时激活。仅折叠 `status=completed` 且 summary 以 `BACKGROUND_BASH_SUMMARY_PREFIX` 开头的任务。失败/killed 任务不折叠。

### collapseHookSummaries

**作用**: 将并行工具调用各自产生的同名 hook 摘要折叠为一条。

**合并逻辑**: `hookCount` 累加、`hookInfos` 和 `hookErrors` 展平、`totalDurationMs` 取最大值（并行执行，max 最接近实际耗时）、`preventedContinuation` 取 OR。

### collapseReadSearch

**作用**: 最复杂的折叠器，将连续的搜索/读取操作（Read, Grep, Glob, REPL, ToolSearch, Snip 等）折叠为摘要组。

`SearchOrReadResult` 细分了多种操作类别：
- `isSearch`: Grep/Glob 等搜索操作
- `isRead`: Read 文件操作
- `isList`: 列表操作
- `isREPL`: REPL 工具
- `isMemoryWrite`: 写入记忆文件的 Write/Edit
- `isAbsorbedSilently`: Snip/ToolSearch 等元操作（不计入计数）

折叠组（`CollapsedReadSearchGroup`）汇总操作计数和文件路径，在 UI 中显示为如 "Read 3 files, searched 2 patterns" 的紧凑摘要。

### collapseTeammateShutdowns

**作用**: 将连续的已完成 teammate 关闭通知折叠为 `teammate_shutdown_batch`（含 count）。

### 共同模式

所有折叠器遵循相同的算法结构：
1. 线性扫描消息列表
2. 识别连续的可折叠消息组
3. 单条保留，多条合成摘要消息
4. 非目标消息直接 pass-through

> **Agent 开发启示**: 上下文折叠是 Agent UI 可用性的关键优化。关键设计：(1) 折叠仅在渲染层进行，不修改底层对话数据，保持了数据完整性；(2) verbose 模式可以绕过折叠，方便调试；(3) 折叠粒度要精准 -- 失败消息不折叠、不同类型不混合。实操建议：在 Agent 产生大量中间步骤的场景中（如连续文件搜索、批量工具调用），渲染层折叠是提升用户体验的低成本手段。
