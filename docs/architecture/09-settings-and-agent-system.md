# 09 - Settings 多层配置系统 与 Agent 子进程/Coordinator 系统

> 深入补充分析：Settings 多层级配置加载与合并机制、远程托管设置、策略限制，
> 以及 Agent 子进程 fork/spawn 机制、Coordinator 模式、Swarm 多 Agent 协作架构。

---

## 目录

- [Part 1: Settings 多层配置系统](#part-1-settings-多层配置系统)
  - [1.1 配置源层级与优先级](#11-配置源层级与优先级)
  - [1.2 配置文件路径映射](#12-配置文件路径映射)
  - [1.3 配置 Schema 完整结构](#13-配置-schema-完整结构)
  - [1.4 配置合并策略](#14-配置合并策略)
  - [1.5 配置缓存与热重载](#15-配置缓存与热重载)
  - [1.6 远程托管设置 (Remote Managed Settings)](#16-远程托管设置)
  - [1.7 MDM 企业管理 (Mobile Device Management)](#17-mdm-企业管理)
  - [1.8 策略限制 (Policy Limits)](#18-策略限制)
  - [1.9 设置同步 (Settings Sync)](#19-设置同步)
  - [1.10 Feature Flags / GrowthBook](#110-feature-flags--growthbook)
- [Part 2: Agent 子进程系统](#part-2-agent-子进程系统)
  - [2.1 Agent 系统总览](#21-agent-系统总览)
  - [2.2 Agent 创建与运行机制](#22-agent-创建与运行机制)
  - [2.3 Fork Subagent 机制](#23-fork-subagent-机制)
  - [2.4 Coordinator 模式](#24-coordinator-模式)
  - [2.5 SendMessage 工具与 Agent 间通信](#25-sendmessage-工具与-agent-间通信)
  - [2.6 Swarm / Team 多 Agent 协作](#26-swarm--team-多-agent-协作)
  - [2.7 Agent 的资源隔离与限制](#27-agent-的资源隔离与限制)
  - [2.8 后台 Agent 任务管理](#28-后台-agent-任务管理)

---

## Part 1: Settings 多层配置系统

### 1.1 配置源层级与优先级

Settings 系统采用多层覆盖（layered override）架构。配置从低优先级到高优先级依次合并，后面的源覆盖前面的源。

```
┌─────────────────────────────────────────────────────────────┐
│                    最终合并结果 (effective)                    │
├─────────────────────────────────────────────────────────────┤
│  5. policySettings     ← 最高优先级 (企业管理/远程)           │
│  4. flagSettings       ← CLI --settings 参数 + SDK inline    │
│  3. localSettings      ← .claude/settings.local.json (gitignored) │
│  2. projectSettings    ← .claude/settings.json (共享)         │
│  1. userSettings       ← ~/.claude/settings.json (全局)       │
│  0. pluginSettings     ← 插件提供的基础设置 (最低优先级)       │
└─────────────────────────────────────────────────────────────┘
```

核心定义位于 `src/utils/settings/constants.ts`:

```typescript
export const SETTING_SOURCES = [
  'userSettings',      // 用户级全局设置
  'projectSettings',   // 项目级共享设置 (提交到 git)
  'localSettings',     // 项目级本地设置 (自动 gitignore)
  'flagSettings',      // CLI --settings 参数
  'policySettings',    // 企业管理策略 (最高优先级)
] as const
```

**关键设计**:
- `policySettings` 和 `flagSettings` 始终启用，不可通过 `--setting-sources` 禁用
- `policySettings` 是只读的，不可通过 `updateSettingsForSource` 写入
- `localSettings` 写入时自动添加 `.gitignore` 规则
- `pluginSettings` 不在 SETTING_SOURCES 中，作为最底层基础在 `loadSettingsFromDisk()` 内部特殊处理

**policySettings 内部的 "首个匹配胜出" 优先级** (定义在 `src/utils/settings/settings.ts`):

```
remote (API 远程) > HKLM/plist (MDM 管理员) > managed-settings.json (文件) > HKCU (用户可写注册表)
```

### 1.2 配置文件路径映射

| Source | 文件路径 | 说明 |
|--------|----------|------|
| `userSettings` | `~/.claude/settings.json` | 用户全局，cowork 模式下为 `cowork_settings.json` |
| `projectSettings` | `$CWD/.claude/settings.json` | 项目级共享，提交到版本控制 |
| `localSettings` | `$CWD/.claude/settings.local.json` | 项目级本地，自动 gitignored |
| `flagSettings` | CLI `--settings <path>` 指定 | SDK 也可通过 inline JSON 设置 |
| `policySettings` (macOS) | `/Library/Application Support/ClaudeCode/managed-settings.json` | 需要管理员权限 |
| `policySettings` (Linux) | `/etc/claude-code/managed-settings.json` | 需要 root 权限 |
| `policySettings` (Windows) | `C:\Program Files\ClaudeCode\managed-settings.json` | 需要管理员权限 |
| `policySettings` drop-in | `managed-settings.d/*.json` | 字母排序合并，后文件覆盖前文件 |
| remote settings 缓存 | `~/.claude/remote-settings.json` | API 远程设置的本地缓存 |

路径映射逻辑: `src/utils/settings/settings.ts` → `getSettingsFilePathForSource()`
管理路径: `src/utils/settings/managedPath.ts` → `getManagedFilePath()`

### 1.3 配置 Schema 完整结构

Settings Schema 定义在 `src/utils/settings/types.ts`，使用 Zod v4 验证。以下是所有支持的配置项:

```
SettingsSchema
├── $schema                          // JSON Schema 引用
├── apiKeyHelper                     // 认证脚本路径
├── awsCredentialExport              // AWS 凭证导出脚本
├── awsAuthRefresh                   // AWS 认证刷新脚本
├── gcpAuthRefresh                   // GCP 认证刷新命令
├── xaaIdp                           // XAA IdP 配置 (条件: CLAUDE_CODE_ENABLE_XAA)
│   ├── issuer                       // OIDC issuer URL
│   ├── clientId                     // OAuth client_id
│   └── callbackPort                 // 回调端口 (可选)
├── fileSuggestion                   // @ 文件建议配置
│   ├── type: "command"
│   └── command
├── respectGitignore                 // 是否遵守 .gitignore
├── cleanupPeriodDays                // 聊天记录保留天数 (默认 30, 0=禁用)
├── env                              // 环境变量 (Record<string, string>)
├── attribution                      // 归属文本
│   ├── commit                       // git commit 归属
│   └── pr                           // PR 归属
├── includeCoAuthoredBy              // [已废弃] 使用 attribution 替代
├── includeGitInstructions           // 是否包含 git 工作流指令
├── permissions                      // 权限配置
│   ├── allow                        // 允许规则列表
│   ├── deny                         // 拒绝规则列表
│   ├── ask                          // 需确认规则列表
│   ├── defaultMode                  // 默认权限模式
│   ├── disableBypassPermissionsMode // 禁用绕过权限模式
│   ├── disableAutoMode              // 禁用自动模式 (条件: TRANSCRIPT_CLASSIFIER)
│   └── additionalDirectories        // 额外许可目录
├── model                            // 覆盖默认模型
├── availableModels                  // 可用模型白名单 (企业)
├── modelOverrides                   // 模型 ID 映射 (企业)
├── enableAllProjectMcpServers       // 自动批准所有项目 MCP 服务
├── enabledMcpjsonServers            // 已批准的 MCP 服务列表
├── disabledMcpjsonServers           // 已拒绝的 MCP 服务列表
├── allowedMcpServers                // 企业 MCP 白名单
├── deniedMcpServers                 // 企业 MCP 黑名单
├── hooks                            // 钩子配置 (PreToolUse, PostToolUse, ...)
├── worktree                         // Git worktree 配置
│   ├── symlinkDirectories           // 符号链接目录
│   └── sparsePaths                  // sparse-checkout 路径
├── disableAllHooks                  // 禁用所有钩子
├── defaultShell                     // 默认 shell (bash/powershell)
├── allowManagedHooksOnly            // 仅允许管理策略的钩子
├── allowedHttpHookUrls              // HTTP 钩子 URL 白名单
├── httpHookAllowedEnvVars           // HTTP 钩子允许的环境变量
├── allowManagedPermissionRulesOnly  // 仅允许管理策略的权限规则
├── allowManagedMcpServersOnly       // 仅从管理策略读取 MCP 白名单
├── strictPluginOnlyCustomization    // 锁定自定义到仅插件来源
├── statusLine                       // 自定义状态行
├── enabledPlugins                   // 启用的插件 (plugin-id@marketplace)
├── extraKnownMarketplaces           // 额外市场源
├── strictKnownMarketplaces          // 企业市场白名单
├── blockedMarketplaces              // 企业市场黑名单
├── forceLoginMethod                 // 强制登录方式 (claudeai/console)
├── forceLoginOrgUUID                // 强制组织 UUID
├── otelHeadersHelper                // OTEL 头部脚本路径
├── outputStyle                      // 输出样式
├── language                         // 首选语言
├── skipWebFetchPreflight            // 跳过 WebFetch 预检
├── sandbox                          // 沙盒配置
├── feedbackSurveyRate               // 反馈调查概率
├── spinnerTipsEnabled               // spinner 提示
├── spinnerVerbs                     // 自定义 spinner 动词
├── spinnerTipsOverride              // 自定义 spinner 提示
├── syntaxHighlightingDisabled       // 禁用语法高亮
├── alwaysThinkingEnabled            // 启用思考模式
├── effortLevel                      // 努力级别 (low/medium/high/max)
├── advisorModel                     // 顾问模型
├── fastMode                         // 快速模式
├── promptSuggestionEnabled          // 提示建议
├── agent                            // 主线程使用的 agent 名称
├── companyAnnouncements             // 公司公告
├── pluginConfigs                    // 插件配置
├── remote                           // 远程会话配置
├── autoUpdatesChannel               // 自动更新通道 (latest/stable)
├── minimumVersion                   // 最低版本
├── plansDirectory                   // 计划文件目录
├── sshConfigs                       // SSH 连接配置
├── claudeMdExcludes                 // 排除的 CLAUDE.md 文件
├── pluginTrustMessage               // 插件信任自定义消息
├── channelsEnabled                  // 频道通知开关
├── allowedChannelPlugins            // 频道插件白名单
├── autoMemoryEnabled                // 自动记忆开关
├── autoMemoryDirectory              // 自动记忆目录
├── skipDangerousModePermissionPrompt // 跳过危险模式确认
└── ... (更多条件性字段)
```

### 1.4 配置合并策略

合并逻辑在 `src/utils/settings/settings.ts` → `loadSettingsFromDisk()` 中实现。

**多源合并规则** (使用 `lodash-es/mergeWith`):

| 值类型 | 合并策略 | 说明 |
|--------|----------|------|
| **数组** | **拼接去重** | `mergeArrays()`: `uniq([...target, ...source])` |
| **对象** | **深度递归合并** | lodash 默认行为，嵌套对象逐层合并 |
| **标量值** | **后者覆盖** | 高优先级源的值覆盖低优先级 |

**写入时的合并规则** (`updateSettingsForSource()`):

| 值类型 | 写入策略 | 说明 |
|--------|----------|------|
| **数组** | **整体替换** | 调用方负责计算最终数组状态 |
| **undefined** | **删除该键** | `srcValue === undefined` 触发 `delete object[key]` |
| **其他** | **lodash 默认合并** | 与读取时行为一致 |

关键实现: `settingsMergeCustomizer()` 函数:

```typescript
// src/utils/settings/settings.ts
export function settingsMergeCustomizer(objValue: unknown, srcValue: unknown): unknown {
  if (Array.isArray(objValue) && Array.isArray(srcValue)) {
    return mergeArrays(objValue, srcValue)  // 拼接去重
  }
  return undefined  // lodash 默认深合并
}
```

**安全性设计**:
- `skipDangerousModePermissionPrompt` 不从 `projectSettings` 读取 (防止恶意项目绕过安全对话框)
- `autoMemoryDirectory` 从 `projectSettings` 设置时被忽略
- `autoMode` 配置排除 `projectSettings` (防止注入分类器规则)

### 1.5 配置缓存与热重载

```
┌──────────────────────────────────────────────────┐
│                 三级缓存架构                       │
├──────────────────────────────────────────────────┤
│  parseFileCache     Map<path, ParsedSettings>    │  ← 文件级缓存
│  perSourceCache     Map<SettingSource, Settings>  │  ← 源级缓存
│  sessionSettingsCache   SettingsWithErrors | null │  ← 会话级合并缓存
└──────────────────────────────────────────────────┘
         │
         ▼ resetSettingsCache() 一次性清除全部三级
```

定义在 `src/utils/settings/settingsCache.ts`。

**热重载流程** (`src/utils/settings/changeDetector.ts`):

```
文件变更 (chokidar 监听)
    │
    ├── awaitWriteFinish (1s 稳定)
    ├── consumeInternalWrite() → 内部写入? 跳过
    ├── executeConfigChangeHooks() → 钩子阻止? 跳过
    │
    ▼
fanOut(source)
    ├── resetSettingsCache()   ← 清除所有缓存
    └── settingsChanged.emit() → 通知订阅者
        ├── applySettingsChange() → 更新 AppState
        │   ├── 重新加载权限规则
        │   ├── 更新钩子快照
        │   └── 同步权限模式
        └── 其他订阅者...
```

**MDM 轮询**: 每 30 分钟轮询一次注册表/plist 变更
**Remote Settings 轮询**: 每 1 小时通过 API 轮询
**文件删除处理**: 1.7 秒宽限期，处理 "删除后重建" 模式

### 1.6 远程托管设置

文件: `src/services/remoteManagedSettings/`

```
┌─────────────────────────────────────────┐
│       Remote Managed Settings           │
├─────────────────────────────────────────┤
│  API: /api/claude_code/settings         │
│  轮询间隔: 每 1 小时                     │
│  超时: 10 秒                             │
│  重试: 最多 5 次 (指数退避)              │
│  缓存文件: ~/.claude/remote-settings.json│
│  ETag 缓存: 基于 SHA256 校验和           │
│  安全检查: 危险设置变更需用户确认         │
└─────────────────────────────────────────┘
```

**适用用户**:
- Console 用户 (API key): 全部适用
- OAuth 用户 (Claude.ai): 仅 Enterprise/C4E 和 Team 订阅
- 外部注入 token: 适用 (API 返回空则无影响)
- 第三方 provider / 自定义 base URL: 不适用
- Cowork (local-agent): 不适用

**失败策略**: Fail-open (失败时继续运行，不阻塞)
- 网络失败 → 使用过期文件缓存
- 无缓存无网络 → 不应用远程策略

**数据流** (定义在 `syncCacheState.ts` / `syncCache.ts` / `index.ts`):

```
启动                                     运行中
 │                                         │
 ├─ loadSettings()                         ├─ pollRemoteSettings() (每1h)
 │   ├─ getRemoteManagedSettingsSyncFromCache()   │
 │   │   └─ 从 ~/.claude/remote-settings.json 加载│
 │   └─ fetchAndLoadRemoteManagedSettings()       │
 │       ├─ fetchWithRetry()                      │
 │       │   └─ If-None-Match (ETag)              │
 │       ├─ checkManagedSettingsSecurity()         │
 │       ├─ setSessionCache()                      │
 │       └─ saveSettings()                         │
 │                                                 │
 └─ settingsChangeDetector.notifyChange()          └─ 变更时 notifyChange()
```

### 1.7 MDM 企业管理

文件: `src/utils/settings/mdm/`

```
┌─────────────────────────────────────────────────────────────┐
│                    MDM 设置加载架构                           │
├───────────┬─────────────────────────────────────────────────┤
│   macOS   │  plutil -convert json 读取 plist                │
│           │  优先级: per-user > device-level > user (ant only)│
│           │  路径: /Library/Managed Preferences/             │
│           │  域: com.anthropic.claudecode                    │
├───────────┼─────────────────────────────────────────────────┤
│  Windows  │  reg query 读取注册表                            │
│           │  HKLM: HKLM\SOFTWARE\Policies\ClaudeCode (管理员)│
│           │  HKCU: HKCU\SOFTWARE\Policies\ClaudeCode (用户)  │
│           │  值名: Settings (REG_SZ JSON)                    │
├───────────┼─────────────────────────────────────────────────┤
│   Linux   │  无 MDM (使用 managed-settings.json 替代)        │
└───────────┴─────────────────────────────────────────────────┘
```

**启动时序** (`rawRead.ts` → `settings.ts`):
1. `startMdmRawRead()` 在 `main.tsx` 模块求值时立即启动子进程
2. 子进程与模块加载并行执行 (节省 5-50ms)
3. `ensureMdmSettingsLoaded()` 在首次设置读取前 await 结果
4. 结果缓存，后续通过 30 分钟轮询刷新

**Drop-in 目录** (`managed-settings.d/*.json`):
- 按文件名字母排序合并
- 后文件覆盖前文件 (类似 systemd/sudoers)
- 允许独立团队发布策略片段 (如 `10-otel.json`, `20-security.json`)

### 1.8 策略限制

文件: `src/services/policyLimits/`

策略限制独立于 Settings 系统，通过专用 API 获取组织级功能限制:

```
API: /api/claude_code/policy_limits
缓存: ~/.claude/policy-limits.json
轮询: 每 1 小时
响应格式: { restrictions: { [policy]: { allowed: boolean } } }
```

使用方式:
```typescript
isPolicyAllowed('allow_product_feedback')  // → boolean
isPolicyAllowed('allow_remote_sessions')   // → boolean
```

**核心设计**:
- 缺失的策略 = 允许 (fail-open)
- essential-traffic-only 模式下 `allow_product_feedback` 缺失时 = 拒绝 (fail-closed, HIPAA)
- 同样支持 ETag 缓存和重试逻辑

### 1.9 设置同步

文件: `src/services/settingsSync/`

设置同步用于在不同 Claude Code 环境间同步用户设置和记忆文件:

```
┌────────────────┐                    ┌────────────────┐
│  Interactive CLI │  ──── upload ───→ │   Anthropic API │
│  (本地开发环境)  │                    │  user_settings  │
└────────────────┘                    └───────┬────────┘
                                              │
                                       download│
                                              ▼
                                    ┌────────────────┐
                                    │   CCR (远程)    │
                                    │  Cloud 环境      │
                                    └────────────────┘
```

同步的文件:
- `~/.claude/settings.json` (用户设置)
- `~/.claude/CLAUDE.md` (用户记忆)
- `.claude/settings.local.json` (项目本地设置, 按 git remote hash 分组)
- `CLAUDE.local.md` (项目本地记忆, 按 git remote hash 分组)

### 1.10 Feature Flags / GrowthBook

文件: `src/services/analytics/growthbook.ts`

Claude Code 使用 GrowthBook SDK 进行功能开关控制:

```
┌──────────────────────────────────────────────────┐
│                GrowthBook 系统                    │
├──────────────────────────────────────────────────┤
│  两种功能开关类型:                                 │
│                                                  │
│  1. 编译时 feature() (bun:bundle)                │
│     - 代码: feature('COORDINATOR_MODE')           │
│     - 在构建时静态替换为 true/false               │
│     - 支持 dead code elimination                  │
│     - 用于: PROACTIVE, KAIROS, VOICE_MODE 等      │
│                                                  │
│  2. 运行时 GrowthBook 远程评估                    │
│     - getFeatureValue_CACHED_MAY_BE_STALE()      │
│     - checkStatsigFeatureGate_CACHED_MAY_BE_STALE│
│     - 基于用户属性的 A/B 测试与渐进发布           │
│     - 属性: id, sessionId, platform, accountUUID │
│             subscriptionType, rateLimitTier ...   │
└──────────────────────────────────────────────────┘
```

**典型用法**:
```typescript
// 编译时开关 - 整个代码块可被编译器消除
if (feature('TRANSCRIPT_CLASSIFIER')) { ... }

// 运行时开关 - 远程可控，支持用户定向
getFeatureValue_CACHED_MAY_BE_STALE('tengu_agent_list_attach', false)
```

---

## Part 2: Agent 子进程系统

### 2.1 Agent 系统总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Agent 系统架构总览                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐     ┌──────────────┐     ┌──────────────────┐     │
│  │   AgentTool  │────→│   runAgent() │────→│    query() 循环   │     │
│  │ (入口/调度)  │     │ (初始化/运行) │     │ (API + 工具执行)  │     │
│  └──────┬──────┘     └──────────────┘     └──────────────────┘     │
│         │                                                           │
│         ├── 同步 Agent (前台运行, 阻塞父级)                          │
│         ├── 异步 Agent (后台运行, 通知完成)                          │
│         ├── Fork Subagent (继承父级上下文)                           │
│         ├── Teammate (Swarm 团队成员)                                │
│         └── Remote Agent (远程 CCR 环境)                             │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │               Coordinator 模式                               │   │
│  │  主 Agent 不执行工具, 只调度 Worker Agent                    │   │
│  │  Worker 通过 <task-notification> 报告结果                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │               Swarm / Team 模式                              │   │
│  │  多个 Teammate 并行工作                                      │   │
│  │  通信: 文件信箱 (mailbox) + SendMessage 工具                 │   │
│  │  后端: tmux / iTerm2 / 进程内 (AsyncLocalStorage)            │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

核心文件:
- `src/tools/AgentTool/AgentTool.tsx` — Agent 工具入口 (schema, call, UI)
- `src/tools/AgentTool/runAgent.ts` — Agent 运行循环
- `src/tools/AgentTool/forkSubagent.ts` — Fork 子代理机制
- `src/tools/AgentTool/prompt.ts` — Agent 工具提示词
- `src/coordinator/coordinatorMode.ts` — Coordinator 模式
- `src/tools/SendMessageTool/SendMessageTool.ts` — Agent 间消息
- `src/utils/swarm/` — Swarm 多 Agent 协作

### 2.2 Agent 创建与运行机制

**Agent 不是真正的子进程，而是进程内的独立会话循环。**

每个 Agent 是在同一 Node.js 进程内运行的一个独立 `query()` 循环，拥有:
- 独立的系统提示 (system prompt)
- 独立的消息历史 (messages)
- 独立的工具池 (可限制)
- 独立的 AbortController (异步 Agent)
- 独立的文件状态缓存 (readFileState)
- 独立的 agentId

**创建流程** (`AgentTool.tsx` → `runAgent.ts`):

```
AgentTool.call()
    │
    ├─ 解析 subagent_type → 查找 AgentDefinition
    ├─ 检查权限 (filterDeniedAgents)
    ├─ 确定运行模式 (同步/异步/team)
    │
    ├─ [Team 路径] → spawnTeammate() → 独立终端或进程内
    │
    ├─ [Remote 路径] → teleportToRemote() → CCR 远程环境
    │
    ├─ [Worktree 路径] → createAgentWorktree() → 隔离 git 副本
    │
    └─ [标准路径] → runAgent()
        ├─ 创建 agentId (UUID)
        ├─ 解析模型 (getAgentModel)
        ├─ 构建系统提示 (getAgentSystemPrompt)
        ├─ 初始化 Agent MCP 服务 (initializeAgentMcpServers)
        ├─ 注册 frontmatter 钩子 (registerFrontmatterHooks)
        ├─ 预加载技能 (skills from frontmatter)
        ├─ 创建隔离上下文 (createSubagentContext)
        │   ├── 同步: 共享 setAppState, abortController
        │   └── 异步: 独立 abortController, 独立 setAppState
        │
        └─ for await (message of query(...))
            ├── 记录到 sidechain transcript
            ├── yield message → 父级
            └── 转发 API metrics 到父级
```

**内置 Agent 类型**:
- `Explore` — 只读搜索/浏览代码 (one-shot, 省略 claudeMd 和 gitStatus)
- `Plan` — 生成实施计划 (one-shot)
- `verification` — 验证代码变更
- `claude-code-guide` — Claude Code 使用指南
- `worker` — Coordinator 模式的通用工人

定义在 `src/tools/AgentTool/built-in/` 目录下。

### 2.3 Fork Subagent 机制

Fork 是一种特殊的 Agent 创建方式，子代理完整继承父级的对话上下文:

```
┌──────────────────────────────────────────────────────────┐
│                Fork Subagent 工作原理                      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  父级对话: [msg1, msg2, ..., assistant(tool_use blocks)] │
│                         │                                │
│                    buildForkedMessages()                  │
│                         │                                │
│  Fork 消息:                                              │
│    [完整 assistant msg,                                   │
│     user(                                                │
│       tool_result(placeholder) × N,  ← 字节相同          │
│       text(<fork-boilerplate> + 指令) ← 仅此不同         │
│     )]                                                   │
│                                                          │
│  设计目标: 多个 Fork 共享 prompt cache                    │
│  ├─ 所有 tool_result 使用相同占位文本                     │
│  ├─ 仅最后一个 text block 不同 (指令)                     │
│  └─ model 继承父级 (不设置 model 参数)                    │
│                                                          │
│  限制:                                                   │
│  ├─ 禁止递归 fork (isInForkChild 检测)                    │
│  ├─ 与 Coordinator 模式互斥                               │
│  ├─ 非交互式会话下不可用                                  │
│  └─ useExactTools=true (继承父级精确工具池)               │
└──────────────────────────────────────────────────────────┘
```

关键代码: `src/tools/AgentTool/forkSubagent.ts`

Fork 子代理的指令格式 (`buildChildMessage`):
```
<fork-boilerplate>
你是一个分叉的工作进程。不要对话或提问，直接使用工具执行。
保持在指令范围内。报告简洁。
输出格式: Scope / Result / Key files / Files changed / Issues
</fork-boilerplate>

<fork-directive>具体任务指令...</fork-directive>
```

### 2.4 Coordinator 模式

文件: `src/coordinator/coordinatorMode.ts`

Coordinator 模式将主 Agent 转变为纯调度者:

```
┌────────────────────────────────────────────────────────────┐
│                 Coordinator 模式架构                        │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────┐                                          │
│  │  Coordinator  │  仅有调度工具:                           │
│  │  (主 Agent)   │  Agent, SendMessage, TaskStop            │
│  └──────┬───────┘                                          │
│         │                                                  │
│    ┌────┴────┬────────┬────────┐                           │
│    ▼         ▼        ▼        ▼                           │
│  Worker1  Worker2  Worker3  Worker4                        │
│  (研究)   (实施)   (验证)   (研究)                          │
│                                                            │
│  通信方式:                                                  │
│  ├─ 启动: Agent({subagent_type: "worker", prompt: "..."}) │
│  ├─ 结果: <task-notification> (user 角色消息)              │
│  ├─ 继续: SendMessage({to: "agent-id", message: "..."})   │
│  └─ 停止: TaskStop({task_id: "agent-id"})                 │
│                                                            │
│  任务工作流:                                                │
│  1. Research (Workers 并行调研)                             │
│  2. Synthesis (Coordinator 综合分析)                        │
│  3. Implementation (Workers 实施)                           │
│  4. Verification (Workers 验证)                             │
└────────────────────────────────────────────────────────────┘
```

激活条件: `CLAUDE_CODE_COORDINATOR_MODE=1` 环境变量 + `feature('COORDINATOR_MODE')`

**Coordinator 的工具限制**:
- Worker 工具: Bash, Read, Edit, Write, Glob, Grep, Agent, SendMessage, Skill 等
- Coordinator 工具: Agent, SendMessage, TaskStop, subscribe_pr_activity 等
- Coordinator 不直接执行 Bash/Read/Write 等工具

### 2.5 SendMessage 工具与 Agent 间通信

文件: `src/tools/SendMessageTool/SendMessageTool.ts`

SendMessage 实现了多种通信模式:

```
┌─────────────────────────────────────────────────────────────┐
│                 SendMessage 路由逻辑                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  input.to                 路由目标                           │
│  ─────────────────────────────────────────────              │
│  "researcher"          → 按名称查找本地 Agent / Teammate     │
│  "*"                   → 广播到所有 Teammates                │
│  "uds:/path/to.sock"  → 本地 UDS 跨会话通信                 │
│  "bridge:session_..."  → Remote Control 跨机器通信          │
│  agent-id (UUID格式)   → 按 ID 查找本地 Agent               │
│                                                             │
│  本地 Agent 路由细节:                                        │
│  ├─ 运行中 → queuePendingMessage() 排队                     │
│  ├─ 已停止 → resumeAgentBackground() 自动恢复               │
│  └─ 已清理 → 从磁盘 transcript 恢复                         │
│                                                             │
│  结构化消息类型:                                             │
│  ├─ shutdown_request  → 请求 Teammate 关闭                   │
│  ├─ shutdown_response → 批准/拒绝关闭请求                    │
│  └─ plan_approval_response → 批准/拒绝计划                   │
└─────────────────────────────────────────────────────────────┘
```

**消息传递机制**:
- 本地 Agent: 直接通过 `queuePendingMessage()` 写入 AppState 中的待处理队列
- Teammates: 通过文件信箱 (`writeToMailbox()`) 写入磁盘 JSON 文件
- 跨会话: UDS socket 或 Bridge (Remote Control) 服务

### 2.6 Swarm / Team 多 Agent 协作

文件: `src/utils/swarm/`

Swarm 系统支持三种执行后端:

```
┌──────────────────────────────────────────────────────────────────┐
│                    Swarm 后端架构                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │   Tmux Backend    │  │  iTerm2 Backend  │  │  In-Process   │  │
│  │  (独立终端 pane)  │  │  (原生分屏)       │  │  Backend      │  │
│  ├──────────────────┤  ├──────────────────┤  ├───────────────┤  │
│  │ 真正的子进程      │  │ 真正的子进程      │  │ 同进程执行     │  │
│  │ 每个 pane 一个    │  │ it2 CLI 管理     │  │ AsyncLocal    │  │
│  │ claude 进程       │  │ 原生分屏面板      │  │ Storage 隔离  │  │
│  │ 通信: 文件信箱    │  │ 通信: 文件信箱    │  │ 通信: 文件信箱│  │
│  │ 终止: kill pane   │  │ 终止: kill pane   │  │ + AppState    │  │
│  └──────────────────┘  └──────────────────┘  │ 终止: abort   │  │
│                                               └───────────────┘  │
│                                                                  │
│  后端选择优先级 (registry.ts):                                    │
│  1. 在 tmux 内 → 使用 tmux                                       │
│  2. 在 iTerm2 内且 it2 可用 → 使用 iTerm2                        │
│  3. tmux 可用 → 使用 tmux (外部会话)                              │
│  4. 非交互式 / auto 且无 pane → 使用 In-Process                  │
│                                                                  │
│  统一接口: TeammateExecutor                                       │
│  ├── spawn(config)      → 启动 Teammate                          │
│  ├── sendMessage(id, msg)                                        │
│  ├── terminate(id)      → 优雅关闭                                │
│  ├── kill(id)           → 强制终止                                │
│  └── isActive(id)       → 检查活跃状态                            │
└──────────────────────────────────────────────────────────────────┘
```

**In-Process Teammate 详细机制** (`src/utils/swarm/spawnInProcess.ts` + `inProcessRunner.ts`):

```
spawnInProcessTeammate()
    │
    ├── 生成 agentId: "name@teamName"
    ├── 创建独立 AbortController
    ├── 创建 TeammateContext (AsyncLocalStorage 隔离)
    ├── 注册 InProcessTeammateTaskState 到 AppState
    │
    └── startInProcessTeammate() (异步, fire-and-forget)
        ├── runWithTeammateContext()
        │   └── runWithAgentContext()
        │       └── runAgent()
        │           └── query() 循环
        │
        ├── 进度追踪 (updateProgressFromMessage)
        ├── 消息历史 (appendTeammateMessage, 最多 50 条 UI 镜像)
        ├── 空闲检测 → 通知 leader
        ├── 计划审批流 (awaitingPlanApproval)
        └── 自动压缩 (autocompact)
```

**TeammateIdentity 结构**:
```typescript
type TeammateIdentity = {
  agentId: string      // "researcher@my-team"
  agentName: string    // "researcher"
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId: string
}
```

### 2.7 Agent 的资源隔离与限制

| 隔离维度 | 同步 Agent | 异步 Agent | Fork Agent | In-Process Teammate |
|----------|-----------|-----------|-----------|-------------------|
| 进程 | 同进程 | 同进程 | 同进程 | 同进程 |
| AbortController | 共享父级 | 独立 (unlinked) | 独立 | 独立 |
| setAppState | 共享 | no-op (隔离) | 共享 | no-op |
| 消息历史 | 独立 (空起始) | 独立 | 继承父级上下文 | 独立 |
| 文件状态缓存 | 新建 | 新建 | 克隆父级 | 克隆父级 |
| 系统提示 | Agent 定义 | Agent 定义 | 继承父级 | Agent 定义 |
| 思考模式 | 禁用 | 禁用 | 继承父级 | 禁用 |
| 工具池 | 按定义过滤 | 按定义过滤 | 精确继承父级 | 按定义过滤 |
| 权限模式 | 可覆盖 | shouldAvoidPrompts | bubble | 可独立切换 |
| 权限提示 | 显示 | 自动拒绝 | bubble 到父级 | 可显示 |
| MCP 服务 | 可添加 | 可添加 | 继承 | 继承 |
| 最大轮次 | AgentDef | AgentDef | 200 | AgentDef |

**工具限制机制** (`resolveAgentTools()`):
- `tools: ['Bash', 'Read']` — 白名单模式
- `disallowedTools: ['Write']` — 黑名单模式
- `tools: ['*']` — 所有工具 (Fork 专用)
- 异步 Agent 自动移除交互式工具 (用户输入等)

**资源清理** (在 `runAgent()` 的 finally 块中):
- 清理 Agent MCP 服务连接
- 清理 session hooks
- 清理 prompt cache 追踪
- 释放文件状态缓存
- 释放 Perfetto 注册
- 释放 transcript 子目录映射
- 释放 AppState 中的 todos 条目
- 终止该 Agent 启动的后台 shell 任务

### 2.8 后台 Agent 任务管理

文件: `src/tasks/LocalAgentTask/LocalAgentTask.tsx`

后台 Agent 使用 `LocalAgentTaskState` 在 AppState 中追踪:

```
┌──────────────────────────────────────────────────────────┐
│              后台 Agent 生命周期                           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  registerAsyncAgent()                                    │
│    → AppState.tasks[agentId] = { status: 'running', ... }│
│                                                          │
│  运行中:                                                  │
│  ├── updateAgentProgress() → 进度更新                     │
│  ├── queuePendingMessage() → 消息排队 (SendMessage)       │
│  ├── getProgressUpdate() → 获取进度快照                    │
│  └── killAsyncAgent() → 终止                              │
│                                                          │
│  完成:                                                    │
│  ├── completeAsyncAgent() → 标记完成                       │
│  └── enqueueAgentNotification()                           │
│      → <task-notification> XML 消息入队                    │
│      → 在下一个 tool round 注入到父级对话                  │
│                                                          │
│  输出:                                                    │
│  ├── 磁盘: ~/.claude/projects/.../subagents/{agentId}.jsonl│
│  ├── 摘要文件: symlink → transcript JSONL                  │
│  └── AgentSummary 服务: 异步生成进度摘要                    │
└──────────────────────────────────────────────────────────┘
```

**自动后台化**: 配置 `CLAUDE_AUTO_BACKGROUND_TASKS` 或 GrowthBook 门控 `tengu_auto_background_agents`，Agent 运行超过 120 秒自动转为后台。

**任务通知格式**:
```xml
<task-notification>
  <task-id>{agentId}</task-id>
  <status>completed|failed|killed</status>
  <summary>{人类可读的状态摘要}</summary>
  <result>{agent 的最终文本响应}</result>
  <usage>
    <total_tokens>N</total_tokens>
    <tool_uses>N</tool_uses>
    <duration_ms>N</duration_ms>
  </usage>
</task-notification>
```

---

## 附录: 关键文件索引

### Settings 系统

| 文件 | 用途 |
|------|------|
| `src/utils/settings/settings.ts` | 配置加载、合并、查询的核心实现 |
| `src/utils/settings/types.ts` | SettingsSchema (Zod) 完整定义 |
| `src/utils/settings/constants.ts` | SETTING_SOURCES 优先级定义 |
| `src/utils/settings/settingsCache.ts` | 三级缓存实现 |
| `src/utils/settings/changeDetector.ts` | chokidar 文件监听 + MDM 轮询 |
| `src/utils/settings/applySettingsChange.ts` | 设置变更应用到 AppState |
| `src/utils/settings/validation.ts` | Zod 错误格式化与 Schema 验证 |
| `src/utils/settings/managedPath.ts` | 管理设置目录路径 (跨平台) |
| `src/utils/settings/mdm/rawRead.ts` | MDM 子进程读取 (plist/reg) |
| `src/utils/settings/mdm/settings.ts` | MDM 设置解析与缓存 |
| `src/utils/settings/mdm/constants.ts` | MDM 常量 (域名、注册表路径) |
| `src/services/remoteManagedSettings/index.ts` | 远程托管设置获取与轮询 |
| `src/services/remoteManagedSettings/syncCacheState.ts` | 远程设置缓存状态 (无循环依赖) |
| `src/services/remoteManagedSettings/syncCache.ts` | 远程设置适用性检查 |
| `src/services/policyLimits/index.ts` | 策略限制 API 客户端 |
| `src/services/settingsSync/index.ts` | 设置同步 (上传/下载) |
| `src/services/analytics/growthbook.ts` | GrowthBook 功能开关 SDK |

### Agent 系统

| 文件 | 用途 |
|------|------|
| `src/tools/AgentTool/AgentTool.tsx` | Agent 工具入口 (schema, call, UI) |
| `src/tools/AgentTool/runAgent.ts` | Agent 运行循环 (query + 上下文) |
| `src/tools/AgentTool/forkSubagent.ts` | Fork 机制 (消息构建, 防递归) |
| `src/tools/AgentTool/prompt.ts` | Agent 工具提示词生成 |
| `src/tools/AgentTool/constants.ts` | Agent 常量 (工具名, one-shot 类型) |
| `src/tools/AgentTool/built-in/*.ts` | 内置 Agent 定义 |
| `src/tools/AgentTool/loadAgentsDir.ts` | 自定义 Agent 加载 |
| `src/tools/AgentTool/resumeAgent.ts` | Agent 恢复运行 |
| `src/coordinator/coordinatorMode.ts` | Coordinator 模式 (系统提示, 工具限制) |
| `src/tools/SendMessageTool/SendMessageTool.ts` | 消息路由 (Agent/Teammate/UDS/Bridge) |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | 后台 Agent 任务状态管理 |
| `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` | 进程内 Teammate 任务 |
| `src/tasks/InProcessTeammateTask/types.ts` | Teammate 任务状态类型 |
| `src/utils/swarm/spawnInProcess.ts` | 进程内 Teammate 创建 |
| `src/utils/swarm/inProcessRunner.ts` | 进程内 Teammate 运行循环 |
| `src/utils/swarm/backends/registry.ts` | 后端检测与选择 (tmux/iTerm2/in-process) |
| `src/utils/swarm/backends/InProcessBackend.ts` | 进程内后端实现 |
| `src/utils/swarm/backends/TmuxBackend.ts` | Tmux 后端实现 |
| `src/utils/swarm/backends/ITermBackend.ts` | iTerm2 后端实现 |
| `src/utils/swarm/backends/types.ts` | 后端类型定义 (PaneBackend, TeammateExecutor) |
| `src/utils/swarm/teamHelpers.ts` | Team 文件操作辅助 |
| `src/utils/swarm/teammateMailbox.ts` | 文件信箱通信 |
