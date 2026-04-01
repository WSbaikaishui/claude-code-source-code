# 10 - Memory 记忆系统与数据持久化机制

> **Copyright (c) 2025 Anthropic and Claude. All rights reserved.**
> This document is part of the Claude Code architecture analysis series.

---

## 目录

- [Part 1: Memory 记忆系统](#part-1-memory-记忆系统)
  - [1.1 架构总览](#11-架构总览)
  - [1.2 Memory 类型分类学](#12-memory-类型分类学)
  - [1.3 Memdir 文件系统结构](#13-memdir-文件系统结构)
  - [1.4 Memory 完整生命周期](#14-memory-完整生命周期)
  - [1.5 记忆检索机制](#15-记忆检索机制)
  - [1.6 记忆提取服务 (extractMemories)](#16-记忆提取服务-extractmemories)
  - [1.7 Session Memory 会话记忆](#17-session-memory-会话记忆)
  - [1.8 团队记忆同步 (Team Memory Sync)](#18-团队记忆同步-team-memory-sync)
  - [1.9 Assistant 模式日志记忆 (KAIROS)](#19-assistant-模式日志记忆-kairos)
  - [1.10 Memory UI 与命令](#110-memory-ui-与命令)
- [Part 2: 数据持久化机制](#part-2-数据持久化机制)
  - [2.1 ~/.claude 目录结构](#21-claude-目录结构)
  - [2.2 对话历史 Transcript 持久化](#22-对话历史-transcript-持久化)
  - [2.3 命令历史 (Command History)](#23-命令历史-command-history)
  - [2.4 大内容的 Hash 外部存储](#24-大内容的-hash-外部存储)
  - [2.5 工具结果持久化 (Tool Result Storage)](#25-工具结果持久化-tool-result-storage)
  - [2.6 文件索引系统 (FileIndex)](#26-文件索引系统-fileindex)
  - [2.7 文件持久化 (File Persistence / BYOC)](#27-文件持久化-file-persistence--byoc)
  - [2.8 数据清理与过期策略](#28-数据清理与过期策略)
- [数据流全景图](#数据流全景图)

---

## Part 1: Memory 记忆系统

### 1.1 架构总览

Claude Code 的 Memory 系统是一个**多层文件驱动的持久记忆架构**，允许 AI 助手跨会话记住用户偏好、项目上下文和工作反馈。系统由以下核心模块组成：

```
Memory 系统架构
┌──────────────────────────────────────────────────────────────────┐
│                        System Prompt                             │
│  loadMemoryPrompt() → 注入 MEMORY.md 索引 + 行为指令              │
└────────────────────────────┬─────────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌───────────────┐  ┌─────────────────┐  ┌──────────────────┐
│  Auto Memory  │  │  Team Memory    │  │  Session Memory  │
│  (个人记忆)    │  │  (团队共享记忆)  │  │  (会话级笔记)    │
│               │  │                 │  │                  │
│ ~/.claude/    │  │ ~/.claude/      │  │ {projectDir}/    │
│ projects/     │  │ projects/       │  │ {sessionId}/     │
│ {slug}/       │  │ {slug}/         │  │ session-memory/  │
│ memory/       │  │ memory/team/    │  │ summary.md       │
└───────┬───────┘  └────────┬────────┘  └────────┬─────────┘
        │                   │                    │
        ▼                   ▼                    ▼
┌───────────────┐  ┌─────────────────┐  ┌──────────────────┐
│ extractMem.   │  │ teamMemSync     │  │ autoCompact      │
│ (后台子代理)   │  │ (API 同步)      │  │ (压缩时注入)     │
│ 会话结束时提取  │  │ pull/push/watch │  │                  │
└───────────────┘  └─────────────────┘  └──────────────────┘
```

**核心文件路径映射：**

| 模块 | 关键文件 |
|------|---------|
| Memdir 核心 | `src/memdir/memdir.ts` |
| 路径管理 | `src/memdir/paths.ts` |
| 类型分类 | `src/memdir/memoryTypes.ts` |
| 记忆扫描 | `src/memdir/memoryScan.ts` |
| 相关性检索 | `src/memdir/findRelevantMemories.ts` |
| 新鲜度管理 | `src/memdir/memoryAge.ts` |
| 团队路径 | `src/memdir/teamMemPaths.ts` |
| 团队 Prompt | `src/memdir/teamMemPrompts.ts` |
| 记忆提取 | `src/services/extractMemories/extractMemories.ts` |
| 提取 Prompt | `src/services/extractMemories/prompts.ts` |
| 会话记忆 | `src/services/SessionMemory/sessionMemory.ts` |
| 团队同步 | `src/services/teamMemorySync/index.ts` |
| 同步 Watcher | `src/services/teamMemorySync/watcher.ts` |
| 秘密扫描 | `src/services/teamMemorySync/secretScanner.ts` |
| Memory 命令 | `src/commands/memory/memory.tsx` |
| 通知组件 | `src/components/memory/MemoryUpdateNotification.tsx` |
| Memory 类型 | `src/utils/memory/types.ts` |

---

### 1.2 Memory 类型分类学

Memory 系统使用封闭的**四类型分类法** (4-type taxonomy)，所有记忆必须属于以下之一：

```
src/memdir/memoryTypes.ts
```

```typescript
export const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
```

| 类型 | 作用域 | 描述 |
|------|--------|------|
| **user** | 始终私有 | 用户角色、目标、职责、知识水平 |
| **feedback** | 默认私有，可团队共享 | 用户对工作方式的纠正和确认 |
| **project** | 倾向团队共享 | 项目上下文：进行中的工作、目标、决策及原因 |
| **reference** | 通常团队共享 | 外部系统指针：Linear 项目、Grafana 面板、Slack 频道 |

**设计原则 — 明确排除的内容：**

- 代码模式、架构、文件路径 — 可通过 grep/git 派生
- Git 历史、变更记录 — `git log` / `git blame` 是权威来源
- 调试解决方案 — 修复已在代码中
- CLAUDE.md 已记录的内容
- 临时任务详情、当前对话上下文

即使用户明确要求保存以上内容，也会要求用户说明"什么是令人惊讶或非显而易见的"。

---

### 1.3 Memdir 文件系统结构

#### 目录路径解析

```
src/memdir/paths.ts → getAutoMemPath()
```

路径解析优先级链：

```
1. CLAUDE_COWORK_MEMORY_PATH_OVERRIDE 环境变量 (Cowork 全路径覆盖)
2. settings.json 中的 autoMemoryDirectory (仅受信源：policy/local/user)
3. {memoryBase}/projects/{sanitized-git-root}/memory/
   其中 memoryBase = CLAUDE_CODE_REMOTE_MEMORY_DIR || ~/.claude
```

**安全设计要点：** `projectSettings`（仓库中的 `.claude/settings.json`）被故意排除在路径覆盖之外 — 恶意仓库可能设置 `autoMemoryDirectory: "~/.ssh"` 来获取对敏感目录的写入权限。

#### 典型磁盘布局

```
~/.claude/
├── projects/
│   └── {sanitized-project-root}/
│       └── memory/                    ← Auto Memory 根目录
│           ├── MEMORY.md              ← 索引入口 (200行/25KB上限)
│           ├── user_role.md           ← 独立记忆文件
│           ├── feedback_testing.md
│           ├── project_auth_rewrite.md
│           ├── reference_linear.md
│           └── team/                  ← Team Memory 子目录
│               ├── MEMORY.md          ← 团队索引入口
│               ├── project_freeze.md
│               └── reference_grafana.md
```

#### MEMORY.md 索引文件

`MEMORY.md` 是**索引文件而非记忆本身**。每个条目应为一行、约150字符的链接：

```markdown
- [User Role](user_role.md) — senior data scientist, Go expert
- [Testing Policy](feedback_testing.md) — real DB, no mocks
```

**截断保护机制** (`src/memdir/memdir.ts`):
- 最大 200 行 (`MAX_ENTRYPOINT_LINES`)
- 最大 25KB (`MAX_ENTRYPOINT_BYTES`)
- 超限时先按行截断，再按字节截断，并附加警告信息

```typescript
export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  // 行截断 → 字节截断 → 附加警告
}
```

#### 单个记忆文件格式

每个记忆文件使用 YAML frontmatter：

```markdown
---
name: {{memory name}}
description: {{一行描述 — 用于决定未来对话中的相关性}}
type: {{user | feedback | project | reference}}
---

{{记忆内容 — feedback/project 类型使用: 规则, **Why:** 和 **How to apply:** 结构}}
```

---

### 1.4 Memory 完整生命周期

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        Memory 生命周期                                   │
│                                                                          │
│  创建                                                                    │
│  ├── 方式A: 用户明确要求 "记住这个"                                       │
│  │   → 主代理直接写入 memory/ 目录                                        │
│  │   → 更新 MEMORY.md 索引                                               │
│  │                                                                       │
│  ├── 方式B: 后台自动提取 (extractMemories)                                │
│  │   → 每轮查询结束时由 stopHooks 触发                                    │
│  │   → Forked Agent 分析最近 N 条消息                                     │
│  │   → 使用 Sonnet 模型判断是否有值得保存的信息                             │
│  │   → 写入 topic file + 更新 MEMORY.md                                   │
│  │                                                                       │
│  ├── 方式C: 团队同步拉取 (Team Memory Pull)                               │
│  │   → 会话启动时从服务器拉取                                              │
│  │   → 服务器内容写入 memory/team/ 目录                                    │
│  │                                                                       │
│  存储                                                                    │
│  ├── 文件格式: Markdown + YAML frontmatter                                │
│  ├── 索引: MEMORY.md (加载到 system prompt 上下文)                         │
│  ├── 目录确保: ensureMemoryDirExists() 在 prompt 构建时调用                │
│  │                                                                       │
│  检索                                                                    │
│  ├── 被动: MEMORY.md 内容始终在 system prompt 中                           │
│  ├── 主动: findRelevantMemories() — Sonnet 驱动的相关性查询                │
│  │   → scanMemoryFiles() 扫描所有 .md 文件的 frontmatter                  │
│  │   → 向 Sonnet 发送 query + 记忆清单                                    │
│  │   → Sonnet 返回最多 5 个相关文件名                                      │
│  │   → 文件内容注入对话上下文                                              │
│  │                                                                       │
│  更新                                                                    │
│  ├── 主代理或提取子代理均可使用 FileEdit/FileWrite 修改                     │
│  ├── "不要写重复记忆，先检查是否有可更新的现有记忆"                          │
│  ├── 相对日期转为绝对日期 (e.g., "周四" → "2026-03-05")                    │
│  │                                                                       │
│  过期与删除                                                               │
│  ├── memoryAge() 计算天数，提供人类可读的新鲜度                             │
│  ├── memoryFreshnessText() 为 >1 天的记忆附加陈旧警告                      │
│  ├── 用户要求 "忘记" → 找到并删除相关条目                                   │
│  ├── 系统提示强调: "记忆是时间点观察，不是实时状态"                          │
│  └── 推荐从记忆引用前验证: 文件路径→检查存在, 函数→grep                     │
└──────────────────────────────────────────────────────────────────────────┘
```

---

### 1.5 记忆检索机制

记忆检索使用**Sonnet 模型驱动的语义选择**，而非向量搜索或关键词匹配。

```
src/memdir/findRelevantMemories.ts
```

**检索流程：**

```
用户查询
    │
    ▼
scanMemoryFiles(memoryDir)
    │  递归扫描 memory/ 目录
    │  读取每个 .md 文件的前 30 行 frontmatter
    │  提取 filename, description, type, mtimeMs
    │  按 mtimeMs 降序排列
    │  上限 200 个文件
    │
    ▼
selectRelevantMemories()
    │  使用 sideQuery() 调用 Sonnet
    │  System prompt: "你正在选择对处理用户查询有用的记忆"
    │  User message: "Query: {query}\n\nAvailable memories:\n{manifest}"
    │  JSON Schema 输出: { selected_memories: string[] }
    │  最多选择 5 个
    │  已展示的记忆被 alreadySurfaced 过滤
    │
    ▼
返回 RelevantMemory[] (path + mtimeMs)
```

**关键设计决策：**
- 选择器会收到最近使用的工具列表 (`recentTools`)，避免为正在使用的工具重复选择参考文档
- 使用 `max_tokens: 256` 限制响应长度
- 查询源标记为 `memdir_relevance`

**新鲜度管理** (`src/memdir/memoryAge.ts`):
- 0天="today"，1天="yesterday"，N天="N days ago"
- 超过1天的记忆附加陈旧警告文本
- 警告强调"记忆是时间点观察"，引用代码行号可能已过时

---

### 1.6 记忆提取服务 (extractMemories)

```
src/services/extractMemories/extractMemories.ts
```

这是 Memory 系统最核心的后台组件 — 一个在每轮对话结束时自动运行的**子代理 (Forked Agent)**。

#### 触发时机

```
主查询循环结束 (模型产出无工具调用的最终响应)
    │
    ▼
handleStopHooks (stopHooks.ts)
    │
    ▼
executeExtractMemories()
    │  门控条件:
    │  ├── feature('EXTRACT_MEMORIES') 构建标记
    │  ├── tengu_passport_quail GrowthBook 特性标记
    │  ├── isAutoMemoryEnabled() = true
    │  ├── 非远程模式
    │  ├── 非子代理 (agentId 为空)
    │  ├── 无进行中的提取 (overlap guard)
    │  │
    │  频率控制:
    │  ├── turnsSinceLastExtraction >= tengu_bramble_lintel (默认 1)
    │  │
    │  互斥检查:
    │  └── hasMemoryWritesSince() — 主代理已写入记忆则跳过
```

#### 运行机制

```typescript
const result = await runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams,              // 共享父对话的 prompt cache
  canUseTool,                   // 受限工具集
  querySource: 'extract_memories',
  forkLabel: 'extract_memories',
  skipTranscript: true,         // 不记录到 transcript，避免竞态
  maxTurns: 5,                  // 硬性上限，防止验证兔子洞
})
```

**工具权限** (`createAutoMemCanUseTool`):
- **允许**: FileRead, Grep, Glob (只读，无限制)
- **允许**: Bash (仅只读命令: ls, find, grep, cat, stat, wc, head, tail)
- **允许**: FileEdit/FileWrite (仅 memory 目录内路径)
- **允许**: REPL (在 REPL 模式下替代原生工具)
- **拒绝**: 所有其他工具 (MCP, Agent, 写入 Bash 等)

**提取子代理的高效策略:**
- Turn 1: 并行 ReadFile 所有可能更新的文件
- Turn 2: 并行 WriteFile/EditFile 所有更改
- 不允许交叉读写

**合并与排队:** 如果提取正在进行时又触发了新提取，新上下文被暂存 (`pendingContext`)，当前提取完成后执行一次跟踪提取 (trailing extraction)。

#### 提取后通知

成功写入记忆文件后，通过 `appendSystemMessage(createMemorySavedMessage(memoryPaths))` 在对话中显示通知。UI 组件 `MemoryUpdateNotification` 显示 "Memory updated in ~/.claude/projects/.../memory/xxx.md · /memory to edit"。

---

### 1.7 Session Memory 会话记忆

```
src/services/SessionMemory/sessionMemory.ts
src/services/SessionMemory/prompts.ts
src/services/SessionMemory/sessionMemoryUtils.ts
```

Session Memory 与 Auto Memory 有本质区别：

| 维度 | Auto Memory | Session Memory |
|------|-------------|----------------|
| 持久性 | 跨会话持久 | 单会话生命周期 |
| 目的 | 长期用户/项目知识 | 当前工作状态追踪 |
| 存储位置 | `~/.claude/projects/{slug}/memory/` | `{projectDir}/{sessionId}/session-memory/summary.md` |
| 格式 | 独立 .md 文件 + MEMORY.md 索引 | 单一结构化 Markdown 文件 |
| 触发方式 | 每轮结束 | 基于 token 增长 + 工具调用阈值 |
| 用途 | 跨会话知识传递 | Context compaction 时注入 |

#### Session Memory 模板结构

```markdown
# Session Title
_短描述标题_

# Current State
_当前正在进行的工作_

# Task specification
_用户要求构建什么_

# Files and Functions
_重要文件及其作用_

# Workflow
_常用 bash 命令和顺序_

# Errors & Corrections
_遇到的错误和修复方式_

# Codebase and System Documentation
_系统组件文档_

# Learnings
_什么有效、什么无效_

# Key results
_用户要求的具体输出_

# Worklog
_逐步工作日志_
```

#### 触发条件

```typescript
export function shouldExtractMemory(messages: Message[]): boolean {
  // 初始化阈值: minimumMessageTokensToInit (默认 10,000 tokens)
  // 更新阈值: minimumTokensBetweenUpdate (默认 5,000 tokens 增长)
  // 工具调用: toolCallsBetweenUpdates (默认 3 次)
  // 触发条件: (token增长 AND 工具调用) OR (token增长 AND 最后轮无工具调用)
}
```

**配置可通过 GrowthBook 远程动态调整** (`tengu_sm_config`)。

#### 大小控制

- 每个 section 上限约 2,000 tokens (`MAX_SECTION_LENGTH`)
- 总文件上限约 12,000 tokens (`MAX_TOTAL_SESSION_MEMORY_TOKENS`)
- 超限时指令模型精简内容

Session Memory 的核心价值在于 **Auto Compact** — 当上下文窗口需要压缩时，Session Memory 内容被注入以保持工作连续性。

---

### 1.8 团队记忆同步 (Team Memory Sync)

```
src/services/teamMemorySync/index.ts
src/services/teamMemorySync/watcher.ts
src/services/teamMemorySync/types.ts
src/services/teamMemorySync/secretScanner.ts
src/services/teamMemorySync/teamMemSecretGuard.ts
```

团队记忆是 Auto Memory 的扩展，允许组织内所有用户共享项目上下文。

#### 同步架构

```
┌────────────────┐         ┌─────────────────────────┐
│  Local FS      │  ◄────  │  Anthropic Server API   │
│  memory/team/  │  Pull   │  /api/claude_code/       │
│                │  ────►  │  team_memory?repo={slug} │
│                │  Push   │                           │
└───────┬────────┘         └─────────────────────────┘
        │
        │  fs.watch({recursive: true})
        │  2s 防抖 (DEBOUNCE_MS)
        ▼
┌────────────────┐
│  File Watcher  │
│  检测变更 →     │
│  schedulePush  │
└────────────────┘
```

#### 同步语义

- **Pull**: 服务器内容覆盖本地 (server wins per-key)
- **Push**: 仅上传内容 hash 不同的键 (delta upload)
- **删除不传播**: 删除本地文件不会从服务器删除，下次 pull 会恢复
- **冲突处理**: 412 Precondition Failed → 重新获取 hash → 重试（最多 2 次）

#### 安全防护

**路径遍历防护** (`src/memdir/teamMemPaths.ts`):
- `sanitizePathKey()`: 拒绝 null 字节、URL 编码遍历、反斜杠、绝对路径、Unicode 规范化攻击
- `validateTeamMemWritePath()`: 双重检查 — `path.resolve()` 字符串级 + `realpath` 符号链接解析
- `realpathDeepestExisting()`: 沿目录树向上解析到最深存在的祖先
- 检测悬挂符号链接 (ENOENT + lstat 区分)
- 检测符号链接循环 (ELOOP)

**秘密扫描** (`src/services/teamMemorySync/secretScanner.ts`):
- 使用 gitleaks 规则子集的客户端扫描
- 涵盖 30+ 种 credential 模式 (AWS, GCP, GitHub PAT, Anthropic API Key, Slack tokens 等)
- `scanForSecrets()` 在上传前运行，匹配的文件被跳过
- `checkTeamMemSecrets()` 在 FileWrite/FileEdit 的 `validateInput` 中调用
- `redactSecrets()` 可在原地替换为 `[REDACTED]`

**永久失败抑制:** 对于不可恢复的推送失败 (no_oauth, 4xx except 409/429)，watcher 抑制后续重试直到检测到 unlink 事件（用户删除了文件）或会话重启。

---

### 1.9 Assistant 模式日志记忆 (KAIROS)

当 `feature('KAIROS')` 启用且处于 Assistant 长期会话模式时，记忆写入方式改变：

```
普通模式:  写入 topic file → 更新 MEMORY.md
KAIROS:    追加到 memory/logs/YYYY/MM/YYYY-MM-DD.md (append-only)
           → MEMORY.md 由独立的 /dream 技能夜间精炼
```

**设计原因:** Assistant 会话实际上是永久性的，agent 使用追加模式写入日期命名的日志文件，避免维护 MEMORY.md 作为实时索引的复杂性。MEMORY.md 仍然被加载到上下文中（作为精炼后的索引）。

---

### 1.10 Memory UI 与命令

**`/memory` 命令** (`src/commands/memory/memory.tsx`):
- 打开 `MemoryFileSelector` 组件让用户选择记忆文件
- 使用 `$VISUAL` 或 `$EDITOR` 环境变量打开外部编辑器
- 创建文件（如不存在），使用 `wx` 标志确保不覆盖

**`MemoryUpdateNotification` 组件** (`src/components/memory/MemoryUpdateNotification.tsx`):
- 在对话中显示 "Memory updated in {displayPath} · /memory to edit"
- 路径显示为相对路径（~/ 或 ./）中较短的一个

---

## Part 2: 数据持久化机制

### 2.1 ~/.claude 目录结构

```
~/.claude/                              ← getClaudeConfigHomeDir()
├── history.jsonl                       ← 全局命令历史
├── paste-cache/                        ← 大粘贴内容的 hash 存储
│   └── {sha256-16char}.txt
├── session-memory/
│   └── config/
│       ├── template.md                 ← 自定义 session memory 模板
│       └── prompt.md                   ← 自定义 session memory 提示
├── projects/                           ← 按项目组织的持久化数据
│   └── {sanitized-project-root}/
│       ├── {sessionId}.jsonl           ← 会话对话记录 (Transcript)
│       ├── {sessionId}/
│       │   ├── subagents/              ← 子代理对话记录
│       │   │   ├── agent-{agentId}.jsonl
│       │   │   └── agent-{agentId}.meta.json
│       │   ├── remote-agents/          ← 远程代理元数据
│       │   │   └── remote-agent-{taskId}.meta.json
│       │   ├── session-memory/
│       │   │   └── summary.md          ← Session Memory 文件
│       │   └── tool-results/           ← 持久化的工具输出
│       │       └── {uuid}.json
│       └── memory/                     ← Auto Memory 目录
│           ├── MEMORY.md
│           ├── *.md
│           └── team/                   ← Team Memory
│               ├── MEMORY.md
│               └── *.md
├── debug/                              ← 调试日志
│   └── latest
└── errors/                             ← 错误日志 (ant 用户)

~/.cache/claude-cli/                    ← envPaths('claude-cli').cache
└── {sanitized-cwd}/
    ├── errors/                         ← 错误日志
    ├── messages/                       ← 消息缓存
    └── mcp-logs-{server}/              ← MCP 服务器日志
```

---

### 2.2 对话历史 Transcript 持久化

```
src/utils/sessionStorage.ts
```

#### JSONL 格式

每个会话的对话历史存储为 JSONL (JSON Lines) 文件，路径格式为：

```
{projectDir}/{sessionId}.jsonl
```

**选择 JSONL 的原因：**
1. **追加写入**: 无需读取-修改-写入整个文件，减少竞态条件
2. **增量持久化**: 每条消息独立成行，崩溃恢复丢失极小
3. **流式读取**: 可从尾部读取最近消息（`readLinesReverse`），支持渐进加载
4. **大小安全**: 避免解析多 GB 的 JSON 文件导致 OOM

#### Transcript 条目类型

```typescript
type Transcript = (
  | UserMessage
  | AssistantMessage
  | AttachmentMessage
  | SystemMessage
)[]

// 不包含 Progress 消息 — 它们是临时 UI 状态
function isTranscriptMessage(entry: Entry): entry is TranscriptMessage {
  return entry.type === 'user' || entry.type === 'assistant'
    || entry.type === 'attachment' || entry.type === 'system'
}
```

#### 写入机制 — Project 类

```typescript
class Project {
  // 写入队列，按文件路径分组
  private writeQueues = new Map<string, Array<{ entry: Entry; resolve: () => void }>>()
  private FLUSH_INTERVAL_MS = 100        // 100ms 批次定时器
  private MAX_CHUNK_BYTES = 100 * 1024 * 1024  // 100MB 单次写入上限

  // 排队写入
  private enqueueWrite(filePath: string, entry: Entry): Promise<void>

  // 定时排空队列 — 批量追加以减少 I/O
  private async drainWriteQueue(): Promise<void> {
    for (const [filePath, queue] of this.writeQueues) {
      const batch = queue.splice(0)
      let content = ''
      for (const { entry, resolve } of batch) {
        const line = jsonStringify(entry) + '\n'
        if (content.length + line.length >= this.MAX_CHUNK_BYTES) {
          await this.appendToFile(filePath, content)  // 分块写入
        }
        content += line
      }
      await this.appendToFile(filePath, content)
    }
  }
}
```

**设计亮点：**
- `pendingEntries` 缓冲 — sessionFile 在第一条用户/助手消息时才物化，避免仅元数据的空文件
- 关闭时 `reAppendSessionMetadata()` — 确保 customTitle/tag 位于文件末尾 64KB 窗口内
- `readHeadAndTail()` 优化 — 只读取文件头和尾用于元数据提取

#### 大小保护

```typescript
export const MAX_TRANSCRIPT_READ_BYTES = 50 * 1024 * 1024  // 50MB OOM 保护
const MAX_TOMBSTONE_REWRITE_BYTES = 50 * 1024 * 1024       // 50MB 墓碑重写限制
```

---

### 2.3 命令历史 (Command History)

```
src/history.ts
```

用户命令历史存储在全局 JSONL 文件中：

```
~/.claude/history.jsonl
```

#### 存储格式

```typescript
type LogEntry = {
  display: string                               // 显示文本
  pastedContents: Record<number, StoredPastedContent>  // 粘贴内容
  timestamp: number                             // 时间戳
  project: string                               // 项目路径
  sessionId?: string                            // 会话 ID
}
```

#### 写入机制

- **异步缓冲**: `pendingEntries` 数组 + 100ms 防抖写入
- **文件锁**: 使用 `lock()` (lockfile) 保护并发写入，10s 超时
- **追加模式**: `appendFile()` 确保多进程安全
- **关闭时刷新**: 注册 cleanup handler 确保退出前写入

#### 大粘贴的分流存储

```typescript
const MAX_PASTED_CONTENT_LENGTH = 1024

if (content.content.length <= MAX_PASTED_CONTENT_LENGTH) {
  // 小内容: 内联存储在 history.jsonl 中
  storedPastedContents[id] = { content: content.content, ... }
} else {
  // 大内容: SHA-256 hash → 外部 paste-cache 文件
  const hash = hashPastedText(content.content)
  storedPastedContents[id] = { contentHash: hash, ... }
  void storePastedText(hash, content.content)  // fire-and-forget
}
```

#### 读取策略

- 从尾部倒序读取 (`readLinesReverse`)
- 当前会话条目优先展示
- 最多 100 条 (`MAX_HISTORY_ITEMS`)
- 支持 `removeLastFromHistory()` — 用于 ESC 键撤销自动恢复

---

### 2.4 大内容的 Hash 外部存储

```
src/utils/pasteStore.ts
```

```
~/.claude/paste-cache/
└── {sha256-first-16-chars}.txt
```

**内容寻址存储** — 相同 hash = 相同内容，覆盖安全。

```typescript
export function hashPastedText(content: string): string {
  return createHash('sha256').update(content).digest('hex').slice(0, 16)
}
```

**清理机制：**
```typescript
export async function cleanupOldPastes(cutoffDate: Date): Promise<void> {
  // 基于文件 mtime 的时间清理
  // 删除 cutoffDate 之前的所有 .txt 文件
}
```

---

### 2.5 工具结果持久化 (Tool Result Storage)

```
src/utils/toolResultStorage.ts
```

当工具输出过大时，不截断而是持久化到磁盘：

```
{projectDir}/{sessionId}/tool-results/{uuid}.json
```

**阈值决策链：**

```typescript
export function getPersistenceThreshold(toolName: string, declaredMax: number): number {
  // 1. Infinity = 硬性退出 (Read 工具自限于 maxTokens)
  // 2. GrowthBook 覆盖: tengu_satin_quoll (per-tool 阈值)
  // 3. 声明值与全局默认值 (50k chars) 取最小
}
```

**超过阈值时**: 内容写入 JSON 文件，原始位置替换为 `<persisted-output>` 标签 + 文件路径，模型可用 FileRead 读取完整内容。

---

### 2.6 文件索引系统 (FileIndex)

```
src/native-ts/file-index/index.ts
```

纯 TypeScript 实现的**模糊文件搜索引擎**，是 Rust NAPI 模块 (nucleo) 的移植版本。

#### 核心算法

```
FileIndex
├── loadFromFileList(fileList)      ← 从 ripgrep 文件列表加载
│   ├── 去重 (HashSet 语义)
│   ├── 预计算: 小写版本、a-z 位图、路径长度
│   └── 异步变体: loadFromFileListAsync (每 4ms 让出事件循环)
│
└── search(query, limit)            ← 模糊匹配搜索
    ├── 智能大小写: 全小写查询 → 不敏感; 含大写 → 敏感
    ├── O(1) 位图拒绝: 路径必须包含查询中所有字母
    ├── 融合 indexOf 扫描: 找位置 + 内联计算间隔/连续奖惩
    ├── 间隔惩罚剪枝: 最佳可能分数低于阈值则跳过
    ├── 边界/驼峰奖分: / \ - _ . 空格后的匹配得高分
    ├── Top-K 维护: 排序数组，避免全排序
    └── 测试文件惩罚: 包含 "test" 的路径 1.05× 惩罚
```

**评分常量** (模拟 nucleo/fzf-v2):
- `SCORE_MATCH = 16` (匹配)
- `BONUS_BOUNDARY = 8` (边界)
- `BONUS_CAMEL = 6` (驼峰)
- `BONUS_CONSECUTIVE = 4` (连续)
- `BONUS_FIRST_CHAR = 8` (首字符)
- `PENALTY_GAP_START = 3` (间隔开始)
- `PENALTY_GAP_EXTENSION = 1` (间隔延伸)

---

### 2.7 文件持久化 (File Persistence / BYOC)

```
src/utils/filePersistence/filePersistence.ts
src/utils/filePersistence/outputsScanner.ts
```

BYOC (Bring Your Own Cloud) 模式下的文件持久化 — 将修改的文件上传到 Files API：

```
环境变量:
  CLAUDE_CODE_ENVIRONMENT_KIND = 'byoc'
  CLAUDE_CODE_REMOTE_SESSION_ID = '{session-id}'

扫描目录:
  {cwd}/{sessionId}/outputs/

流程:
  1. findModifiedFiles() — 递归扫描 mtime >= turnStartTime 的文件
  2. 跳过符号链接 (安全)
  3. 检查文件数量上限 (FILE_COUNT_LIMIT)
  4. uploadSessionFiles() — 并行上传 (DEFAULT_UPLOAD_CONCURRENCY)
  5. 报告成功/失败计数
```

这是为远程 Claude Code 会话设计的，普通 CLI 用户不触发此路径。

---

### 2.8 数据清理与过期策略

Claude Code 的数据清理是**保守的被动策略**：

| 数据类型 | 清理机制 |
|---------|---------|
| 命令历史 | 最近 100 条 (`MAX_HISTORY_ITEMS`) |
| Paste cache | `cleanupOldPastes(cutoffDate)` — 基于 mtime 的时间清理 |
| Session transcript | 无自动清理; 50MB 读取保护避免 OOM |
| Auto Memory | 200 文件上限 (`MAX_MEMORY_FILES`); MEMORY.md 200行/25KB |
| Session Memory | 12K token 总量上限; 每 section 2K token 上限 |
| Tool results | 无自动清理; 随会话目录一起 |
| Error logs | 内存中保留最近 100 条 (`MAX_IN_MEMORY_ERRORS`) |
| Team Memory | 服务器端 max_entries 限制 (可按组织配置); 单文件 250KB |

**MEMORY.md 索引膨胀防护:**
```
> WARNING: MEMORY.md is {reason}. Only part of it was loaded. Keep index entries
> to one line under ~200 chars; move detail into topic files.
```

---

## 数据流全景图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        Claude Code 数据持久化全景                             │
│                                                                              │
│  用户输入                                                                    │
│  ├──► history.jsonl        (命令历史, JSONL, 锁保护)                         │
│  ├──► paste-cache/         (大内容 SHA-256 寻址, >1KB)                       │
│  │                                                                          │
│  对话消息                                                                    │
│  ├──► {sessionId}.jsonl    (Transcript, JSONL, 批量追加, 100ms 间隔)          │
│  ├──► subagents/agent-{id}.jsonl  (子代理对话)                               │
│  │                                                                          │
│  工具输出                                                                    │
│  ├──► tool-results/{uuid}.json    (大输出持久化, >50K chars)                  │
│  │                                                                          │
│  记忆系统                                                                    │
│  ├──► memory/MEMORY.md            (索引, 200行上限)                           │
│  ├──► memory/*.md                 (个人记忆文件, frontmatter)                 │
│  ├──► memory/team/MEMORY.md       (团队索引)                                 │
│  ├──► memory/team/*.md            (团队记忆, API 同步)                        │
│  ├──► session-memory/summary.md   (会话笔记, 结构化 Markdown)                │
│  │                                                                          │
│  分析遥测                                                                    │
│  └──► (内存中 → Datadog/firstParty, 不落盘持久化)                            │
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ 追加写入      │  │ 内容寻址     │  │ API 同步     │  │ 模糊索引     │     │
│  │ (JSONL)      │  │ (SHA-256)    │  │ (OAuth+PUT)  │  │ (位图+评分)  │     │
│  │              │  │              │  │              │  │              │     │
│  │ history      │  │ paste-cache  │  │ team memory  │  │ FileIndex    │     │
│  │ transcript   │  │ tool-results │  │ pull/push    │  │ search()     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘     │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

**文档版本:** 基于 Claude Code CLI 源码 commit `3da94d5` 分析
