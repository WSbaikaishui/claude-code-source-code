# 08 - System Prompt 构建链路与消息压缩策略

> 深入分析 Claude Code CLI 的系统提示构建管线和多层消息压缩体系

---

## 目录

- [Part 1: System Prompt 构建链路](#part-1-system-prompt-构建链路)
  - [1.1 总体架构](#11-总体架构)
  - [1.2 System Prompt 完整组成](#12-system-prompt-完整组成)
  - [1.3 CLAUDE.md 发现与加载](#13-claudemd-发现与加载)
  - [1.4 动态上下文注入](#14-动态上下文注入)
  - [1.5 缓存与大小控制](#15-缓存与大小控制)
- [Part 2: 多层消息压缩策略](#part-2-多层消息压缩策略)
  - [2.1 压缩体系总览](#21-压缩体系总览)
  - [2.2 Layer 0: Tool Result Budget](#22-layer-0-tool-result-budget)
  - [2.3 Layer 1: Snip Compact](#23-layer-1-snip-compact)
  - [2.4 Layer 2: Microcompact](#24-layer-2-microcompact)
  - [2.5 Layer 3: Context Collapse](#25-layer-3-context-collapse)
  - [2.6 Layer 4: Auto Compact](#26-layer-4-auto-compact)
  - [2.7 Layer 5: Reactive Compact](#27-layer-5-reactive-compact)
  - [2.8 Session Memory Compact](#28-session-memory-compact)
  - [2.9 Token 预算管理](#29-token-预算管理)

---

## Part 1: System Prompt 构建链路

### 1.1 总体架构

System Prompt 的构建分为三个独立的上下文层，分别对应 API 调用中的不同注入位置：

```
┌─────────────────────────────────────────────────────────────────┐
│                     API 请求最终结构                              │
├─────────────────────────────────────────────────────────────────┤
│  system:  systemPrompt + systemContext                          │
│           (作为 API 的 system 参数)                               │
│                                                                  │
│  messages[0]: userContext (作为 <system-reminder> 注入首条消息)     │
│  messages[1..N]: 实际对话消息                                     │
└─────────────────────────────────────────────────────────────────┘
```

**核心入口文件：**

| 文件 | 职责 |
|------|------|
| `src/constants/prompts.ts` | `getSystemPrompt()` — 生成默认系统 prompt 数组 |
| `src/context.ts` | `getSystemContext()` / `getUserContext()` — 上下文层 |
| `src/utils/queryContext.ts` | `fetchSystemPromptParts()` — 组装入口 |
| `src/QueryEngine.ts` | 最终组装与 API 调用 |

**构建流程图：**

```
QueryEngine.submitMessage()
  │
  ├─► fetchSystemPromptParts()              [src/utils/queryContext.ts:44]
  │     │
  │     ├─► getSystemPrompt()               [src/constants/prompts.ts:444]
  │     │     ├── 静态部分 (intro, system, tasks, actions, tools, tone, efficiency)
  │     │     ├── SYSTEM_PROMPT_DYNAMIC_BOUNDARY (全局缓存分界线)
  │     │     └── 动态部分 (registry-managed sections)
  │     │           ├── session_guidance
  │     │           ├── memory (loadMemoryPrompt)
  │     │           ├── env_info_simple
  │     │           ├── language
  │     │           ├── output_style
  │     │           ├── mcp_instructions
  │     │           ├── scratchpad
  │     │           ├── frc (Function Result Clearing)
  │     │           └── summarize_tool_results
  │     │
  │     ├─► getUserContext()                 [src/context.ts:155]
  │     │     ├── claudeMd (CLAUDE.md 内容)
  │     │     └── currentDate
  │     │
  │     └─► getSystemContext()              [src/context.ts:116]
  │           ├── gitStatus (分支, 状态, 最近提交)
  │           └── cacheBreaker (ant-only)
  │
  ├─► 组装 systemPrompt = [...defaultSystemPrompt, ...appendSystemPrompt]
  │
  ├─► 注入 userContext → messages[0] 作为 <system-reminder>
  │     [src/utils/api.ts:449 prependUserContext()]
  │
  └─► 注入 systemContext → 追加到 system 参数末尾
        [src/utils/api.ts:437 appendSystemContext()]
```

### 1.2 System Prompt 完整组成

`getSystemPrompt()` 返回一个字符串数组，每个元素是一个独立的 prompt 段。最终结构分为 **静态部分** 和 **动态部分**，由 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 分隔：

#### 静态部分（跨组织可缓存）

```
src/constants/prompts.ts

1. getSimpleIntroSection()      — 身份声明和 CYBER_RISK_INSTRUCTION
2. getSimpleSystemSection()     — 系统行为规则（工具权限、system-reminder 标签等）
3. getSimpleDoingTasksSection() — 编码风格、任务执行规范
4. getActionsSection()          — 操作安全指南（不可逆操作确认等）
5. getUsingYourToolsSection()   — 工具使用指南（Read/Edit/Write/Glob/Grep 优先于 Bash）
6. getSimpleToneAndStyleSection() — 语气和样式规范
7. getOutputEfficiencySection() — 输出效率要求（简洁、直接）
────── SYSTEM_PROMPT_DYNAMIC_BOUNDARY ──────
```

#### 动态部分（通过 `systemPromptSection` 注册管理）

```
src/constants/prompts.ts → resolveSystemPromptSections()

8.  session_guidance       — Agent 工具、技能搜索、验证代理等
9.  memory                 — loadMemoryPrompt() (MEMORY.md / 自动记忆)
10. ant_model_override     — Anthropic 内部模型覆盖
11. env_info_simple        — 环境信息（cwd、平台、shell、模型名等）
12. language               — 语言偏好（如 "Always respond in 中文"）
13. output_style           — 输出样式配置
14. mcp_instructions       — MCP 服务器指令
15. scratchpad             — Scratchpad 目录说明
16. frc                    — Function Result Clearing 指引
17. summarize_tool_results — 工具结果摘要建议
18. numeric_length_anchors — 数字长度锚点 (ant-only)
19. token_budget           — Token 预算指引
20. brief                  — Brief/Kairos 模式
```

**缓存机制（`src/constants/systemPromptSections.ts`）：**

- `systemPromptSection(name, compute)` — 计算一次后缓存，直到 `/clear` 或 `/compact` 触发 `clearSystemPromptSections()` 清除
- `DANGEROUS_uncachedSystemPromptSection(name, compute, reason)` — 每轮重新计算，会破坏 prompt cache

### 1.3 CLAUDE.md 发现与加载

#### 文件来源分类

`src/utils/claudemd.ts` 定义了 4 层记忆文件层次，按优先级从低到高加载：

```
加载顺序（越靠后优先级越高，模型关注度越高）:

1. Managed Memory  — /etc/claude-code/CLAUDE.md
   │                 + /etc/claude-code/.claude/rules/*.md
   │                 （全局管理策略，对所有用户生效）
   │
2. User Memory     — ~/.claude/CLAUDE.md
   │                 + ~/.claude/rules/*.md
   │                 （用户私有全局指令）
   │
3. Project Memory  — 从项目根目录向上遍历直到文件系统根:
   │                 每级目录检查:
   │                   ├── CLAUDE.md
   │                   ├── .claude/CLAUDE.md
   │                   └── .claude/rules/*.md
   │                 （项目级指令，已签入代码库）
   │
4. Local Memory    — CLAUDE.local.md (每级目录)
                     （用户私有项目级指令，不签入）
```

#### 发现路径

```
getMemoryFiles()  [src/utils/claudemd.ts:790]
  │
  ├─ 1. Managed: getMemoryPath('Managed') → /etc/claude-code/CLAUDE.md
  │    + getManagedClaudeRulesDir() → /etc/claude-code/.claude/rules/*.md
  │
  ├─ 2. User (if userSettings enabled):
  │    getMemoryPath('User') → ~/.claude/CLAUDE.md
  │    + getUserClaudeRulesDir() → ~/.claude/rules/*.md
  │
  ├─ 3. Project + Local: 从 CWD 向上遍历到根目录
  │    for each dir (从根到 CWD，距离 CWD 越近优先级越高):
  │      ├── Project: CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md
  │      └── Local:   CLAUDE.local.md
  │
  ├─ 4. Additional dirs (--add-dir): 如 CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1
  │
  ├─ 5. AutoMem: MEMORY.md (if isAutoMemoryEnabled())
  │
  └─ 6. TeamMem: 团队记忆 (if feature('TEAMMEM'))
```

#### 文件处理管线

每个记忆文件经过以下处理：

```
processMemoryFile()
  │
  ├─ readFile → rawContent
  ├─ parseFrontmatterPaths() → 提取 paths 元数据（glob 匹配规则）
  ├─ stripHtmlComments() → 移除 <!-- ... --> 注释
  ├─ extractIncludePathsFromTokens() → 解析 @include 指令
  │    支持: @path, @./relative, @~/home, @/absolute
  ├─ truncateEntrypointContent() → 截断 MEMORY.md (AutoMem/TeamMem)
  └─ 返回 MemoryFileInfo { path, type, content, globs? }
```

#### 注入格式

`getClaudeMds()` 将所有文件组装为以下格式注入 `userContext.claudeMd`：

```
Codebase and user instructions are shown below. Be sure to adhere to these
instructions. IMPORTANT: These instructions OVERRIDE any default behavior and
you MUST follow them exactly as written.

Contents of /etc/claude-code/CLAUDE.md (user's private global instructions for all projects):

[文件内容]

Contents of /project/CLAUDE.md (project instructions, checked into the codebase):

[文件内容]
```

#### 禁用控制

- `CLAUDE_CODE_DISABLE_CLAUDE_MDS=1` — 完全禁用
- `--bare` 模式 — 跳过自动发现，但仍尊重 `--add-dir` 显式指定的目录
- `MAX_MEMORY_CHARACTER_COUNT = 40000` — 单文件推荐上限

### 1.4 动态上下文注入

#### Git Status (`src/context.ts:36`)

`getGitStatus()` 并行获取以下信息：

```
branch      ← git rev-parse --abbrev-ref HEAD
mainBranch  ← 默认分支名
status      ← git status --short (截断到 MAX_STATUS_CHARS=2000)
log         ← git log --oneline -n 5
userName    ← git config user.name
```

生成格式：
```
This is the git status at the start of the conversation...
Current branch: feature/xxx
Main branch: main
Git user: user
Status: [短状态]
Recent commits: [5条记录]
```

**注意：** Git status 在 CCR (Claude Code Remote) 或禁用 git 指令时跳过。会话期间缓存（memoize），不会更新。

#### 当前日期

```typescript
// src/context.ts:186
currentDate: `Today's date is ${getLocalISODate()}.`
```

#### 环境信息 (`computeSimpleEnvInfo`)

```
<env>
Primary working directory: /path/to/project
Is directory a git repo: Yes
Platform: darwin
Shell: zsh
OS Version: Darwin 24.6.0
</env>
You are powered by the model named Claude Opus 4.6. The exact model ID is claude-opus-4-6[1m].
Assistant knowledge cutoff is May 2025.
```

### 1.5 缓存与大小控制

#### 三层缓存策略

```
┌──────────────────────────────────────────────────────┐
│              System Prompt 缓存架构                    │
├──────────────────────────────────────────────────────┤
│                                                       │
│  1. 全局缓存 (cacheScope: 'global')                    │
│     SYSTEM_PROMPT_DYNAMIC_BOUNDARY 之前的静态部分       │
│     跨组织/跨会话共享缓存                               │
│                                                       │
│  2. Section 缓存 (systemPromptSections)                │
│     动态部分按 section name 缓存                        │
│     在 /clear 或 /compact 时清除                        │
│                                                       │
│  3. memoize 缓存                                       │
│     getSystemContext / getUserContext                   │
│     会话级别缓存，compact 后 clear                      │
│                                                       │
└──────────────────────────────────────────────────────┘
```

#### 大小控制

- Git status 截断到 `MAX_STATUS_CHARS = 2000` 字符
- CLAUDE.md 单文件推荐 `MAX_MEMORY_CHARACTER_COUNT = 40000` 字符
- MEMORY.md (AutoMem/TeamMem) 经过 `truncateEntrypointContent()` 截断
- Custom system prompt 覆盖时跳过默认 system prompt 和 system context

---

## Part 2: 多层消息压缩策略

### 2.1 压缩体系总览

Claude Code 实现了一个 **6 层递进式消息压缩体系**，每层在不同时机触发，处理不同粒度的上下文压力。执行顺序在 `src/query.ts` 的 `query()` 生成器函数中编排：

```
                        ┌───────────────────────────────┐
                        │    query() 每轮执行管线        │
                        └───────────────────────────────┘
                                      │
                  ┌───────────────────┴───────────────────┐
                  │                                       │
         ┌────── ▼ ──────┐                     ┌──────── ▼ ────────┐
         │  请求前处理     │                     │    请求后恢复       │
         └───────────────┘                     └───────────────────┘
                  │                                       │
    ┌─────────────┼─────────────┐              ┌──────────┤
    ▼             ▼             ▼              ▼          ▼
 Layer 0       Layer 1       Layer 2       Layer 5     Layer 5
 Tool Result   Snip          Microcompact   Reactive   (413 恢复)
 Budget        Compact       (缓存编辑)     Compact
    │             │             │
    ▼             ▼             ▼
 Layer 3       Layer 4
 Context       Auto
 Collapse      Compact
```

**完整执行顺序（`src/query.ts:365-467`）：**

```
1. applyToolResultBudget()       — Layer 0: 单消息工具结果大小预算
2. snipCompactIfNeeded()         — Layer 1: 历史消息裁剪 (feature: HISTORY_SNIP)
3. microcompactMessages()        — Layer 2: 工具结果微压缩
4. applyCollapsesIfNeeded()      — Layer 3: 上下文折叠 (feature: CONTEXT_COLLAPSE)
5. autoCompactIfNeeded()         — Layer 4: 自动压缩（调用模型摘要）
────── API 调用 ──────
6. tryReactiveCompact()          — Layer 5: 413 错误响应式压缩
```

### 2.2 Layer 0: Tool Result Budget

**文件:** `src/utils/toolResultStorage.ts`

**触发条件:** 每轮请求前自动运行（如果 `contentReplacementState` 存在）

**策略:** 对单条消息中的工具结果施加字节大小预算。超过预算的工具结果内容被替换为磁盘引用：

```typescript
// src/utils/toolResultStorage.ts:769
export async function enforceToolResultBudget(
  messages: Message[],
  state: ContentReplacementState,
  skipToolNames: ReadonlySet<string>,
): Promise<{ messages: Message[]; newlyReplaced: ToolResultReplacementRecord[] }>
```

**关键特性：**
- 逐消息检查，按 `perMessageBudgetLimit` 阈值
- 已处理的工具结果通过 `seenIds` 跟踪，避免重复处理
- 替换结果可持久化到会话记录，支持 `--resume`
- 对没有 `maxResultSizeChars` 限制的工具跳过检查

### 2.3 Layer 1: Snip Compact

**文件:** `src/services/compact/snipCompact.ts` (feature-gated: `HISTORY_SNIP`, 源码编译后不可见)

**触发条件:** 每轮请求前自动运行

**策略:** 裁剪历史消息中较早的部分，释放 token 空间。不调用模型，纯规则裁剪。

```typescript
// src/query.ts:401-410
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
  if (snipResult.boundaryMessage) {
    yield snipResult.boundaryMessage
  }
}
```

**关键特性：**
- 返回 `{ messages, tokensFreed, boundaryMessage? }`
- `tokensFreed` 传递给 autoCompact 以修正 token 阈值判断
- 可能生成 boundary message（系统消息标记裁剪边界）
- 与 microcompact 不互斥，两者可同时运行

### 2.4 Layer 2: Microcompact

**文件:** `src/services/compact/microCompact.ts`

**触发条件:** 每轮请求前自动运行，在 snip 之后

**策略:** 清除旧的工具调用结果内容，保留最近 N 个结果。分为三种子路径：

#### 2.4.1 Time-Based Microcompact（基于时间的微压缩）

**配置:** `src/services/compact/timeBasedMCConfig.ts`

```typescript
const TIME_BASED_MC_CONFIG_DEFAULTS = {
  enabled: false,
  gapThresholdMinutes: 60,  // 1小时后触发
  keepRecent: 5,             // 保留最近5个结果
}
```

**触发条件:** 距离上一条 assistant 消息超过 `gapThresholdMinutes` 分钟（服务端 prompt cache 已过期）

**行为:** 直接修改消息内容，将旧工具结果替换为 `[Old tool result content cleared]`

```
evaluateTimeBasedTrigger()
  │
  ├─ 检查 config.enabled && isMainThreadSource(querySource)
  ├─ 计算 gapMinutes = (now - lastAssistantTimestamp) / 60000
  ├─ if gapMinutes >= config.gapThresholdMinutes:
  │    ├─ 收集所有 COMPACTABLE_TOOLS 的 tool_use ID
  │    ├─ 保留最近 keepRecent 个
  │    └─ 清除其余工具结果内容
  └─ 短路：不执行 Cached MC
```

#### 2.4.2 Cached Microcompact（缓存编辑式微压缩）

**前提:** feature(`CACHED_MICROCOMPACT`) 开启，模型支持 cache editing

**行为:** 不修改本地消息内容，而是通过 API 的 `cache_edits` 机制在服务端删除旧的工具结果：

```
cachedMicrocompactPath()
  │
  ├─ 注册新工具结果到 cachedMCState
  ├─ getToolResultsToDelete() 基于 count-threshold 计算需删除的工具
  ├─ createCacheEditsBlock() 生成 cache_edits 指令
  ├─ 设置 pendingCacheEdits（API 层注入）
  └─ 消息本身不变
```

**可压缩的工具类型（`COMPACTABLE_TOOLS`）：**
- FileReadTool, Shell tools, GrepTool, GlobTool
- WebSearchTool, WebFetchTool, FileEditTool, FileWriteTool

### 2.5 Layer 3: Context Collapse

**文件:** `src/services/contextCollapse/index.ts` (feature-gated: `CONTEXT_COLLAPSE`, 源码编译后不可见)

**触发条件:** 每轮请求前，在 microcompact 之后、autocompact 之前

**策略:** 将较早的对话轮次"折叠"为摘要，维持粒度高于全量压缩但低于完整保留的中间状态。

```typescript
// src/query.ts:440-447
if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(
    messagesForQuery,
    toolUseContext,
    querySource,
  )
  messagesForQuery = collapseResult.messages
}
```

**设计要点：**
- 采用 commit log 模式 — 折叠结果存在独立的 collapse store 中，不修改 REPL 的消息数组
- `projectView()` 在每轮入口重放 commit log，生成折叠后的视图
- 上下文使用率约 90% 时开始提交折叠，95% 时触发阻塞式折叠生成
- 与 autocompact **互斥** — 当 contextCollapse 启用时，`shouldAutoCompact()` 返回 false
- 拥有独立的 413 恢复路径 `recoverFromOverflow()`，优先于 reactiveCompact

### 2.6 Layer 4: Auto Compact

**文件:** `src/services/compact/autoCompact.ts` + `src/services/compact/compact.ts`

**触发条件:** 上下文 token 用量超过 `autoCompactThreshold`

#### 阈值计算

```typescript
// src/services/compact/autoCompact.ts:72
function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  // effectiveContextWindow = contextWindow - min(maxOutputTokens, 20000)
  return effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS  // 13,000
}
```

```
  上下文窗口 (如 200K)
  ─ max_output_tokens 预留 (max 20K)
  ═ 有效上下文窗口 (effectiveContextWindow)
  ─ AUTOCOMPACT_BUFFER_TOKENS (13K)
  ═ autoCompactThreshold (触发阈值)
```

#### 执行流程

```
autoCompactIfNeeded()
  │
  ├─ 检查断路器: consecutiveFailures >= 3 ? 跳过
  ├─ shouldAutoCompact() 检查:
  │    ├─ querySource 非 session_memory / compact / marble_origami
  │    ├─ isAutoCompactEnabled() (非 DISABLE_COMPACT / DISABLE_AUTO_COMPACT)
  │    ├─ 非 reactiveCompact-only 模式
  │    ├─ 非 contextCollapse 模式
  │    └─ tokenCount - snipTokensFreed >= threshold
  │
  ├─ 优先尝试 Session Memory Compact
  │    └─ trySessionMemoryCompaction() → 如果成功，直接返回
  │
  └─ 回退到 Legacy Compact
       └─ compactConversation() → 调用模型生成摘要
```

#### compactConversation 详细流程

```
compactConversation()  [src/services/compact/compact.ts:387]
  │
  ├─ 执行 PreCompact Hooks
  │
  ├─ 准备摘要请求:
  │    ├─ getCompactPrompt(customInstructions)
  │    │    └─ NO_TOOLS_PREAMBLE + BASE_COMPACT_PROMPT + NO_TOOLS_TRAILER
  │    └─ stripImagesFromMessages() — 移除图片避免摘要请求过大
  │
  ├─ 调用模型生成摘要 (streamCompactSummary):
  │    ├─ 使用 forked agent 或 streaming 调用
  │    ├─ 如果遇到 prompt-too-long:
  │    │    └─ truncateHeadForPTLRetry() — 丢弃最早的 API 轮次，最多重试 3 次
  │    └─ formatCompactSummary() — 剥离 <analysis> 块，格式化 <summary>
  │
  ├─ 重建上下文:
  │    ├─ 清除 readFileState 和 loadedNestedMemoryPaths 缓存
  │    ├─ createPostCompactFileAttachments() — 恢复最近 5 个读取过的文件
  │    │    (POST_COMPACT_MAX_FILES_TO_RESTORE=5, 每文件最多 5000 tokens)
  │    ├─ createPlanAttachmentIfNeeded() — 恢复计划上下文
  │    ├─ createSkillAttachmentIfNeeded() — 恢复已调用的技能
  │    ├─ getDeferredToolsDeltaAttachment() — 重新注入延迟工具
  │    ├─ getAgentListingDeltaAttachment() — 重新注入 agent 列表
  │    └─ getMcpInstructionsDeltaAttachment() — 重新注入 MCP 指令
  │
  ├─ 执行 SessionStart Hooks (compact 后等同新会话)
  ├─ 执行 PostCompact Hooks
  │
  └─ 返回 CompactionResult:
       ├─ boundaryMarker — 压缩边界标记
       ├─ summaryMessages — 包含摘要的用户消息
       ├─ attachments — 恢复的文件/计划/技能附件
       └─ hookResults — 钩子结果消息
```

#### 摘要 Prompt 结构

```
NO_TOOLS_PREAMBLE:
  "CRITICAL: Respond with TEXT ONLY. Do NOT call any tools."

BASE_COMPACT_PROMPT:
  要求生成 9 个章节:
  1. Primary Request and Intent
  2. Key Technical Concepts
  3. Files and Code Sections
  4. Errors and fixes
  5. Problem Solving
  6. All user messages
  7. Pending Tasks
  8. Current Work
  9. Optional Next Step

NO_TOOLS_TRAILER:
  "REMINDER: Do NOT call any tools."
```

#### 断路器机制

```typescript
// src/services/compact/autoCompact.ts:70
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

连续 3 次 autocompact 失败后停止重试，避免对不可恢复的上下文溢出场景无限循环。

### 2.7 Layer 5: Reactive Compact

**文件:** `src/services/compact/reactiveCompact.ts` (feature-gated: `REACTIVE_COMPACT`, 源码编译后不可见)

**触发条件:** API 返回 `prompt_too_long` (413) 错误后

**策略:** 事后补救 — 在 API 拒绝请求后，从尾部剥离消息轮次并重试。

```typescript
// src/query.ts:1119-1121
if ((isWithheld413 || isWithheldMedia) && reactiveCompact) {
  const compacted = await reactiveCompact.tryReactiveCompact({
    hasAttempted: hasAttemptedReactiveCompact,
    querySource,
    aborted: toolUseContext.abortController.signal.aborted,
    messages: messagesForQuery,
    cacheSafeParams: { ... },
  })
}
```

**关键特性：**
- 在 stream loop 中拦截 413 错误（`isWithheldPromptTooLong`），不立即暴露给用户
- 与 Context Collapse 的恢复路径协作：先尝试 collapse drain，再回退到 reactive compact
- 也处理 media size error（图片/文档过大）
- `hasAttemptedReactiveCompact` 防止无限重试

### 2.8 Session Memory Compact

**文件:** `src/services/compact/sessionMemoryCompact.ts`

**触发时机:** 作为 autoCompact 的优先替代路径

**策略:** 使用后台持续提取的 Session Memory 替代模型调用摘要。不调用模型，直接使用已有的结构化记忆。

```
trySessionMemoryCompaction()
  │
  ├─ 检查 shouldUseSessionMemoryCompaction()
  │    └─ tengu_session_memory && tengu_sm_compact 功能开关
  │
  ├─ initSessionMemoryCompactConfig() — 从 GrowthBook 加载配置
  │    DEFAULT: { minTokens: 10000, minTextBlockMessages: 5, maxTokens: 40000 }
  │
  ├─ waitForSessionMemoryExtraction() — 等待后台提取完成
  │
  ├─ getSessionMemoryContent() — 获取 Session Memory 内容
  │    └─ 如果为空或仅模板 → 返回 null, 回退到 legacy compact
  │
  ├─ calculateMessagesToKeepIndex() — 计算保留范围:
  │    ├─ 从 lastSummarizedMessageId 开始
  │    ├─ 向后扩展直到满足 minTokens + minTextBlockMessages
  │    ├─ 不超过 maxTokens
  │    └─ adjustIndexToPreserveAPIInvariants() 保证 tool_use/tool_result 配对
  │
  ├─ truncateSessionMemoryForCompact() — 截断过大的记忆段
  │
  └─ createCompactionResultFromSessionMemory() → CompactionResult
```

**优势：** 无需模型调用，延迟极低，且保留最近的原始消息。

### 2.9 Token 预算管理

#### 关键常量

```typescript
// src/services/compact/autoCompact.ts
AUTOCOMPACT_BUFFER_TOKENS      = 13_000   // 触发 autocompact 的缓冲
WARNING_THRESHOLD_BUFFER_TOKENS = 20_000   // 显示警告的缓冲
ERROR_THRESHOLD_BUFFER_TOKENS   = 20_000   // 显示错误的缓冲
MANUAL_COMPACT_BUFFER_TOKENS    = 3_000    // 阻塞限制的缓冲
MAX_OUTPUT_TOKENS_FOR_SUMMARY   = 20_000   // 摘要输出预留

// src/services/compact/compact.ts
POST_COMPACT_MAX_FILES_TO_RESTORE = 5      // 压缩后恢复文件数
POST_COMPACT_TOKEN_BUDGET       = 50_000   // 压缩后附件总预算
POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000   // 单文件恢复上限
POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000  // 单技能恢复上限
POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000  // 技能恢复总预算

// src/services/compact/sessionMemoryCompact.ts
DEFAULT_SM_COMPACT_CONFIG = {
  minTokens: 10_000,              // 保留消息最低 token 数
  minTextBlockMessages: 5,        // 保留消息最低条数
  maxTokens: 40_000,              // 保留消息最高 token 数
}
```

#### Token 预算状态机

```
calculateTokenWarningState(tokenUsage, model)

  ┌─────────────────────────────────────────────────────────┐
  │                   effectiveContextWindow                 │
  │                                                          │
  │  ┌─── normalZone ───┬── warningZone ──┬── errorZone ──┐ │
  │  │                  │                 │               │ │
  │  │  正常运行         │  显示黄色警告     │  显示红色警告  │ │
  │  │                  │                 │               │ │
  │  ├──────────────────┼─────────────────┼───────────────┤ │
  │  0          warning(-20K)     error(-20K)    blocking  │
  │                                             (-3K)      │
  │                                                          │
  │  autoCompactThreshold = effectiveWindow - 13K            │
  │  (如开启 autocompact, 在此阈值自动触发压缩)              │
  └─────────────────────────────────────────────────────────┘
```

#### 环境变量覆盖

| 变量 | 作用 |
|------|------|
| `DISABLE_COMPACT` | 禁用所有压缩 |
| `DISABLE_AUTO_COMPACT` | 仅禁用自动压缩，保留手动 `/compact` |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 百分比覆盖阈值 (0-100) |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | 覆盖上下文窗口大小 |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | 覆盖阻塞限制 |
| `ENABLE_CLAUDE_CODE_SM_COMPACT` | 强制启用 Session Memory Compact |
| `DISABLE_CLAUDE_CODE_SM_COMPACT` | 强制禁用 Session Memory Compact |

---

## 总结

### System Prompt 构建特点

1. **分层设计** — 静态/动态分离，通过 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 实现全局缓存和会话缓存的共存
2. **Section 注册制** — 动态 section 通过 `systemPromptSection()` 注册，支持缓存和按需重算
3. **多源记忆合并** — CLAUDE.md 从 4 个层级加载，支持 glob 匹配、@include 指令、frontmatter 元数据
4. **严格大小控制** — git status 截断 2K，记忆文件推荐上限 40K，AutoMem 额外截断

### 消息压缩体系特点

1. **6 层递进** — 从轻量级的 Tool Result Budget 到重量级的 Reactive Compact，逐层升级
2. **互斥与协作** — Context Collapse 和 Auto Compact 互斥；Snip 和 Microcompact 可并行
3. **0 模型调用优先** — Snip、Microcompact、Session Memory Compact 均不调用模型，只有 Auto Compact 和 Reactive Compact 需要模型摘要
4. **断路器保护** — 连续 3 次 autocompact 失败后停止重试
5. **Post-compact 恢复** — 压缩后自动恢复最近文件、计划、技能、MCP 指令等关键上下文
