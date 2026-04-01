# Claude Code CLI -- 架构全景总览

> 基于 Claude Code v2.1.88 源码分析
> 本文档是所有模块分析文档的顶层索引与架构概览，适合第一次接触代码库的开发者阅读。

---

## 目录

1. [项目概述](#1-项目概述)
2. [整体架构图](#2-整体架构图)
3. [核心数据流](#3-核心数据流)
4. [模块依赖关系](#4-模块依赖关系)
5. [开发者快速指南](#5-开发者快速指南)
6. [关键设计决策](#6-关键设计决策)
7. [文档索引](#7-文档索引)

---

## 1. 项目概述

### 1.1 Claude Code 是什么

Claude Code 是 Anthropic 官方推出的 CLI 工具，允许开发者在终端中与 Claude AI 进行交互式对话，实现代码编写、文件操作、Shell 命令执行、代码审查、Git 操作等开发任务。它不仅是一个聊天客户端，更是一个完整的 AI 驱动开发环境。

### 1.2 版本与技术栈

| 项目 | 说明 |
|------|------|
| **版本** | v2.1.88 |
| **运行时** | Bun (主要) / Node.js 18+ (兼容) |
| **语言** | TypeScript (TSX) |
| **UI 框架** | React 19 + 自定义 Ink 框架 (终端 UI) |
| **布局引擎** | Yoga (Flexbox, WASM / 纯 TS 回退) |
| **构建工具** | Bun bundler (主要) / esbuild (替代) |
| **API 客户端** | Anthropic SDK (支持直连 / Bedrock / Vertex / Foundry) |
| **协议** | MCP (Model Context Protocol), OAuth 2.0 + PKCE, LSP |
| **原生模块** | Rust NAPI (语法高亮、Diff、音频) + 纯 TS 回退 |
| **遥测** | OpenTelemetry (Metrics / Logs / Traces) |
| **包管理** | npm / Bun |

### 1.3 核心能力

- **交互式终端 REPL** -- 完整的终端 UI，支持虚拟滚动、Vim 模式、语法高亮
- **AI 对话引擎** -- 多轮对话循环，支持工具调用、上下文压缩、流式输出
- **40+ 内置工具** -- 文件读写编辑、Shell 执行、搜索、子代理、网络请求等
- **MCP 协议集成** -- 连接外部工具服务器，扩展 AI 能力边界
- **多模式权限** -- 6 种权限模式 + 规则系统 + Hook 拦截 + 分类器自动审批
- **多代理协作** -- Coordinator 模式、子代理、后台代理、Swarm 团队
- **远程执行** -- CCR 远程会话、IDE 桥接、Server 模式
- **持久化记忆** -- CLAUDE.md 项目记忆、会话记忆、记忆目录
- **90+ 斜杠命令** -- 丰富的命令系统，支持技能、插件、工作流扩展
- **SDK 接口** -- 编程式 API，支持 query/tool/session 等操作

---

## 2. 整体架构图

<!-- 整体架构全景图 -- 第 1 部分: 入口层 + 初始化链路 + 核心引擎层 -->

```mermaid
graph TD
    subgraph USER_INPUT["👤 用户入口"]
        direction LR
        TERM["用户终端<br/>stdin/stdout"]
        WEB["claude.ai Web"]
        IDE["IDE (VS Code等)"]
        SDKSCRIPT["SDK/脚本"]
    end

    subgraph ENTRY["🚪 入口层 (Entrypoints)"]
        direction LR
        CLI["CLI 入口<br/>cli.tsx"]
        BRIDGE["Bridge API<br/>bridgeMain"]
        MCPS["MCP Server<br/>mcp.ts"]
        AGENTSDK["Agent SDK<br/>agentSdk / Types.ts"]
    end

    TERM --> CLI
    WEB --> BRIDGE
    IDE --> MCPS
    SDKSCRIPT --> AGENTSDK

    CLI -- "快速路径分发<br/>--version / --daemon-worker<br/>remote-control / ..." --> INIT_CHAIN
    BRIDGE --> INIT_CHAIN
    MCPS --> INIT_CHAIN
    AGENTSDK --> INIT_CHAIN

    subgraph INIT_CHAIN["⚙️ 主初始化链路 (main.tsx)"]
        CMD["Commander 解析<br/>CLI 参数/子命令<br/>preAction 钩子"]
        INIT["init() 全局初始化<br/>配置/网络/遥测<br/>CA/代理/MDM"]
        SETUP["setup() 会话初始化<br/>cwd/hooks/worktree<br/>后台任务/预取"]
        CMD --> INIT --> SETUP
        subgraph MODES["启动模式"]
            direction LR
            INTERACTIVE["交互模式<br/>(REPL 启动)"]
            HEADLESS["非交互模式 (-p)<br/>runHeadless()"]
        end
        SETUP --> MODES
    end

    subgraph ENGINE["🧠 核心引擎层 (Core Engine)"]
        direction LR
        QE["QueryEngine<br/>submitMessage()<br/>会话生命周期管理<br/>token/cost 累计<br/>消息历史维护"]
        QUERY["query() 对话循环<br/>while (true):<br/>  snipCompact / microcompact<br/>  contextCollapse / autoCompact<br/>  callModel() / runTools()<br/>  stopHooks() / attachments"]
        CTXMGR["上下文管理<br/>System Prompt / CLAUDE.md<br/>Git 状态 / 环境信息"]
        COORD["Coordinator 模式<br/>Leader + Workers<br/>Scratchpad 共享"]

        QE -->|"submitMessage()"| QUERY
        CTXMGR -->|"注入上下文"| QUERY
    end

    MODES --> ENGINE
    ENGINE -->|"callModel()"| API_LAYER
    ENGINE -->|"runTools()"| TOOL_LAYER
```

<!-- 整体架构全景图 -- 第 2 部分: API 通信层 + 工具执行层 -->

```mermaid
graph TD
    subgraph API_LAYER["📡 API 通信层 (API Service)"]
        direction LR
        CLIENT["client.ts -- Anthropic SDK<br/>多提供商: 直连/Bedrock/Vertex/Foundry/Claude.ai"]
        CLAUDE["claude.ts -- 流式查询核心 (3400+ 行)<br/>prompt 缓存 / 努力度调节 / advisor 模式"]
        RETRY["withRetry.ts<br/>10次重试 / 529处理<br/>OAuth刷新 / 模型回退"]
        COST["成本追踪<br/>cost-tracker.ts<br/>按模型累计token"]
    end

    subgraph TOOL_EXEC["🔧 工具执行层 (Tool System)"]
        direction LR
        ORCH["toolOrchestration.ts<br/>partitionToolCalls()<br/>并发/串行批次"]
        EXEC["toolExecution.ts<br/>Zod验证 → PreToolUse Hook<br/>→ 权限检查 → tool.call()<br/>→ PostToolUse Hook → 结果"]
        STREAM_EXEC["StreamingToolExecutor<br/>流式响应中即时启动"]
        ORCH --> EXEC --> STREAM_EXEC
    end

    TOOL_EXEC --> TOOLS_IMPL

    subgraph TOOLS_IMPL["🧰 具体工具实现 (src/tools/)"]
        subgraph FILE_OPS["核心文件操作"]
            direction LR
            BASH["Bash -- Shell 命令"]
            READ["Read -- 文件/图片/PDF"]
            EDIT_T["Edit -- 精确替换编辑"]
            WRITE_T["Write -- 文件创建/覆写"]
            GLOBGREP["Glob/Grep -- 文件搜索"]
        end
        subgraph AGENT_OPS["代理/任务"]
            direction LR
            AGENT["Agent -- 子代理"]
            SENDMSG["SendMessage"]
            TASKMGMT["TaskCreate/Get/Update/Stop"]
        end
        subgraph NET_OPS["网络/外部"]
            direction LR
            WEBFETCH["WebFetch -- HTTP"]
            WEBSEARCH["WebSearch -- 搜索"]
            MCPTOOL["MCPTool -- MCP"]
            TOOLSEARCH["ToolSearch -- 延迟发现"]
            SKILL["Skill -- 技能执行"]
        end
    end
```

<!-- 整体架构全景图 -- 第 3 部分: 终端 UI 层 -->

```mermaid
graph TD
    subgraph UI_LAYER_A["🖥️ 终端 UI 层 -- 渲染引擎与组件树"]
        subgraph INK_ENGINE["自定义 Ink 渲染引擎"]
            direction LR
            REACT19["React 19"]
            RECONCILER["react-reconciler"]
            DOM_TREE["自定义 DOM 树"]
            YOGA["Yoga 布局 (Flexbox)"]
            SCREEN_BUF["Screen 缓冲区<br/>(CharPool+StylePool)"]
            LOGUPDATE["LogUpdate 差异更新<br/>(双缓冲 + 16ms 节流)"]
            ANSI_OUT["ANSI 终端输出"]

            REACT19 --> RECONCILER --> DOM_TREE --> YOGA
            SCREEN_BUF --> LOGUPDATE --> ANSI_OUT
        end

        YOGA --> SCREEN_BUF

        subgraph COMPONENT_TOP["组件树顶层"]
            direction LR
            APP["App"]
            APPSTATE_P["AppStateProvider"]
            KEYBINDING["KeybindingSetup"]
            REPL["REPL (5000+ 行, 核心屏幕)"]
            APP --> APPSTATE_P --> KEYBINDING --> REPL
        end

        REPL --> ALTSCREEN["AlternateScreen"] --> FULLSCREEN["FullscreenLayout"]
    end
```

以下继续展示 FullscreenLayout 之下的组件结构、设计系统与消息渲染管线：

```mermaid
graph TD
    FULLSCREEN["FullscreenLayout"]

    subgraph UI_LAYER_B["🖥️ 终端 UI 层 -- 消息区域与辅助系统"]
        subgraph MSG_AREA["消息展示区域"]
            SCROLLBOX["ScrollBox (虚拟滚动)"]
            VMSGLIST["VirtualMessageList"]
            subgraph MSG_TYPES["消息类型"]
                direction LR
                ASSTTEXT["AssistantTextMessage"]
                ASSTTOOL["AssistantToolUseMessage"]
                USERTEXT["UserTextMessage"]
                PERMREQ["PermissionRequest"]
            end
            SCROLLBOX --> VMSGLIST --> MSG_TYPES
        end

        subgraph INPUT_AREA["输入与通知"]
            direction LR
            PROMPTINPUT["PromptInput"]
            TEXTINPUT["TextInput / VimTextInput"]
            NOTIFICATIONS["Notifications"]
            MODAL["Modal (对话框/命令面板)"]
            PROMPTINPUT --> TEXTINPUT
        end

        subgraph DESIGN_SYS["设计系统"]
            direction LR
            THEMED["ThemedBox / ThemedText"]
            DIALOG["Dialog / Pane / Tabs"]
            FUZZY["FuzzyPicker / ProgressBar"]
        end

        subgraph MSG_PIPELINE["消息渲染管线"]
            direction LR
            NORMALIZE["messages → normalize → reorder"]
            COLLAPSE["→ collapse → group → VirtualMessageList"]
            HIGHLIGHT["Markdown → Token LRU → HighlightedCode"]
            NORMALIZE --> COLLAPSE --> HIGHLIGHT
        end
    end

    FULLSCREEN --> MSG_AREA
    FULLSCREEN --> INPUT_AREA
```

<!-- 整体架构全景图 -- 第 4 部分: 服务层 + 状态与安全层 + 特殊功能层 -->

```mermaid
graph TD
    subgraph SVC_LAYER["🔌 服务层 (Services Layer)"]
        subgraph SVC_ROW1["核心通信服务"]
            direction LR
            SVC_API["API 通信<br/>client.ts / claude.ts<br/>withRetry.ts"]
            SVC_MCP["MCP 协议<br/>client.ts / config.ts<br/>auth.ts / 7种传输"]
            SVC_OAUTH["OAuth 认证<br/>index.ts / client.ts<br/>crypto.ts / PKCE 流程"]
            SVC_ANALYTICS["分析遥测<br/>index.ts / growthbook<br/>datadog / sink.ts"]
            SVC_COMPACT["上下文压缩<br/>compact.ts / microCompact.ts<br/>autoCompact.ts / contextCollapse"]
        end
        subgraph SVC_ROW2["扩展服务"]
            direction LR
            SVC_MEM["记忆服务<br/>extract / sessionMem<br/>teamMem"]
            SVC_LSP["LSP 服务<br/>manager / server<br/>client"]
            SVC_PLUGIN["插件服务<br/>install / operations"]
            SVC_TIPS["提示服务<br/>tipReg / tipSched"]
            SVC_SYNC["设置同步<br/>settingsSync<br/>remoteManagedSet<br/>policyLimits"]
        end
    end

    SVC_LAYER --> STATE_LAYER
```

以下继续展示状态与安全层以及特殊功能层的详细结构：

```mermaid
graph TD
    subgraph STATE_LAYER["🔒 状态与安全层 (State & Security)"]
        subgraph STATE_ROW1["状态管理"]
            direction LR
            STORE["AppState Store<br/>自研响应式 Store<br/>不可变更新 / 发布-订阅<br/>React 集成 (useSyncExternal)"]
            GLOBAL_STATE["全局可变状态<br/>bootstrap/state.ts<br/>sessionId/cwd/<br/>cost/meter/hooks"]
            TASK["Task 系统<br/>7种任务类型:<br/>local_bash / local_agent /<br/>remote_agent / in_process_mate /<br/>local_workflow / monitor_mcp / dream"]
        end
        subgraph STATE_ROW2["安全与扩展"]
            direction LR
            PERM["权限系统<br/>6种模式 / 多源规则<br/>YOLO 分类器 / 拒绝追踪"]
            HOOKS["Hooks 系统<br/>27种事件 / 4种类型<br/>command/prompt/http/agent"]
            SANDBOX["沙箱安全<br/>文件系统限制<br/>网络域名限制<br/>settings.json 保护"]
        end
        subgraph STATE_ROW3["插件与记忆"]
            direction LR
            PLUGINS["插件系统<br/>builtin + marketplace<br/>MCP/LSP/Hooks 扩展"]
            SKILLS["Skills 系统<br/>bundled/disk/plugin/MCP<br/>Frontmatter 语法"]
            MEMDIR["记忆系统 (Memdir)<br/>MEMORY.md 索引<br/>四类记忆分类<br/>智能检索 (sideQuery)"]
        end
    end

    subgraph SPECIAL["⚡ 特殊功能层 (Special Features)"]
        subgraph SPEC_ROW1["交互与远程"]
            direction LR
            VOICE["语音交互<br/>CoreAudio / WebSocket<br/>三层门控"]
            REMOTE["远程Agent<br/>CCR 会话 / WebSocket<br/>HTTP POST"]
            IDE_BRIDGE["IDE 桥接<br/>Remote Control / REST API"]
            SERVER["Server<br/>Direct Connect / HTTP/WS"]
        end
        subgraph SPEC_ROW2["基础设施"]
            direction LR
            BUDDY["Buddy<br/>伴侣系统 / 确定性生成<br/>RPG 属性"]
            GIT["Git 集成<br/>直读.git / ref 解析<br/>watcher"]
            UPSTREAM["上游代理<br/>CONNECT over WS<br/>prctl 防调"]
            NATIVE_TS["原生 TS<br/>file-index / yoga-layout<br/>color-diff"]
            BASH_PARSE["Bash 解析<br/>递归下降 / 安全分析"]
            BUILD["构建系统<br/>Bun/esbuild / feature()"]
        end
    end

    STATE_LAYER --> SPECIAL
```

---

## 3. 核心数据流

### 3.1 从用户输入到 AI 响应 -- 完整链路

以下是一条用户消息从键入到收到 AI 响应的完整数据流，一笔到底：

```mermaid
graph TD
    START(["👤 用户在终端键入文本并按 Enter"])

    subgraph INPUT_PHASE["输入捕获与处理 (步骤 1-4)"]
        direction LR
        STEP1["[1] 终端输入捕获<br/>parseKeypress() 解析 ANSI 转义序列"]
        STEP2["[2] 事件分发<br/>Ink Dispatcher 分发 KeyboardEvent<br/>(捕获/冒泡)"]
        STEP3["[3] TextInput 处理<br/>TextInput/VimTextInput 组件接收按键<br/>(粘贴检测/图片/历史/补全)"]
        STEP4["[4] 输入提交<br/>PromptInput.onSubmit() 触发<br/>REPL 组件接收用户消息"]
        STEP1 --> STEP2 --> STEP3 --> STEP4
    end

    START --> INPUT_PHASE

    STEP5["[5] QueryEngine.submitMessage()<br/>discoveredSkillNames.clear() / setCwd(cwd)<br/>解析模型和思考配置"]
    STEP6["[6] 系统提示词构建<br/>getSystemPrompt() / getUserContext() / getSystemContext()<br/>coordinator + memory-mechanics 注入"]
    STEP7["[7] 用户输入处理 processUserInput()"]

    STEP4 --> STEP5 --> STEP6 --> STEP7

    STEP7_SLASH{"斜杠命令?"}
    STEP7_LOCAL["本地命令 → 直接执行, return"]
    STEP7_PROMPT["Prompt 命令 → 生成内容注入对话"]
    STEP7_MSG["构建 UserMessage (文本+图片+粘贴内容)"]

    STEP7 --> STEP7_SLASH
    STEP7_SLASH -->|"是"| STEP7_LOCAL
    STEP7_SLASH -->|"Prompt 类型"| STEP7_PROMPT
    STEP7_SLASH -->|"否"| STEP7_MSG
    STEP7_PROMPT --> STEP7_MSG

    STEP8["[8] 持久化 & 确认<br/>recordTranscript() / flushSessionStorage()<br/>yield buildSystemInitMessage()"]
    STEP7_MSG --> STEP8
    STEP8 --> STEP9["[9] 进入 query() 循环 -- queryLoop() while (true)"]
```

以下继续展示 query() 循环内部的处理流程直到 AI 响应返回给用户：

```mermaid
graph TD
    STEP9["[9] query() 循环入口"]

    subgraph LOOP["🔄 单次循环迭代"]
        subgraph PREP_AND_CALL["预处理与 API 调用"]
            direction LR
            A["A. 预处理<br/>snipCompact / microcompact<br/>contextCollapse / autoCompact"]
            B["B. API 调用<br/>queryModelWithStreaming()<br/>withRetry() 重试包装<br/>流式返回 StreamEvent"]
            A --> B
        end
        C["C. 响应处理<br/>assistant 消息 → yield 给调用方<br/>tool_use blocks → 收集<br/>StreamingToolExecutor 即时启动"]
        subgraph TOOL_RUN["工具执行"]
            direction LR
            D["D. 工具调度<br/>partitionToolCalls()<br/>并发安全 → 并行 (≤10)<br/>非并发 → 串行"]
            D_EACH["每个工具执行:<br/>Zod验证 → PreToolUse Hook<br/>→ 权限检查 → tool.call()<br/>→ PostToolUse Hook → 结果"]
            D --> D_EACH
        end
        E["E. 后处理<br/>handleStopHooks() / checkTokenBudget()<br/>getAttachmentMessages() / maxTurns"]
        F{"F. 继续决策"}

        PREP_AND_CALL --> C --> TOOL_RUN --> E --> F
    end

    STEP9 --> LOOP
    F -->|"有 tool_use → 继续循环"| PREP_AND_CALL
    F -->|"无 tool_use → return"| STEP10

    STEP10["[10] 结果收集<br/>addToTotalSessionCost() / recordTranscript()<br/>yield type:result, subtype:success"]
    STEP11["[11] UI 渲染<br/>normalizeMessages() → VirtualMessageList<br/>Markdown 渲染 → LogUpdate.diff() → ANSI 输出"]
    DONE(["👤 用户在终端看到 AI 响应"])

    STEP10 --> STEP11 --> DONE
```

### 3.2 工具权限决策链路

当 AI 模型决定调用某个工具时，权限检查经过以下层级：

```mermaid
graph TD
    TOOL_USE(["模型输出 tool_use"])

    L1["第1层: 工具级过滤 (getTools)<br/>filterToolsByDenyRules() -- 整工具禁用"]
    L2["第2层: 工具自身权限 (checkPermissions)<br/>每个工具各自的权限检查逻辑<br/>返回: allow / deny / ask / passthrough"]
    L3["第3层: Hook 权限干预 (PreToolUse)<br/>用户配置的 Hook 可以 allow / deny / ask"]
    L4["第4层: 统一权限决策<br/>(hasPermissionsToUseTool)"]

    TOOL_USE --> L1 --> L2 --> L3 --> L4

    L4 --> RULES["规则匹配<br/>alwaysAllow / alwaysDeny / alwaysAsk"]
    L4 --> YOLO["YOLO 分类器<br/>(auto 模式, LLM 判断)"]
    L4 --> INTERACTIVE["交互式询问<br/>(用户 UI 确认)"]
    L4 --> REMOTE_B["远程桥接<br/>(IDE/Web 确认)"]

    RULES --> RESOLVE
    YOLO --> RESOLVE
    INTERACTIVE --> RESOLVE
    REMOTE_B --> RESOLVE

    RESOLVE{"首个完成者获胜<br/>(ResolveOnce 模式)"}

    RESOLVE -->|"allow"| EXEC_TOOL["✅ 执行工具"]
    RESOLVE -->|"deny"| DENY_INFO["❌ 返回拒绝信息给模型"]
    RESOLVE -->|"ask"| ASK_UI["❓ 显示权限确认 UI"]
```

---

## 4. 模块依赖关系

### 4.1 模块层次图

```mermaid
graph TD
    subgraph L1["🚪 入口层 (Entrypoints)"]
        ENT["cli.tsx → main.tsx → init.ts → setup.ts → replLauncher.tsx<br/>mcp.ts / agentSdkTypes.ts / bridge / daemon / runner"]
    end

    subgraph L2["🧠 核心引擎层 (Core Engine)"]
        CORE["QueryEngine.ts ↔ query.ts ↔ context.ts<br/>cost-tracker.ts / history.ts / query/config.ts"]
    end

    subgraph L3["第3层: 通信 / 工具 / 命令"]
        direction LR
        subgraph L3A["📡 API 通信"]
            API_MOD["api/ / client.ts<br/>claude.ts / withRetry"]
        end
        subgraph L3B["🔧 工具系统"]
            TOOL_MOD["Tool.ts / tools.ts<br/>tools/ / toolOrch / toolExec"]
        end
        subgraph L3C["⌨️ 命令系统"]
            CMD_MOD["commands.ts<br/>commands/ / skills/"]
        end
    end

    subgraph L4["🔌 服务层 (Services)"]
        SVC_MOD["mcp/ | oauth/ | analytics/ | compact/ | extractMemories/<br/>SessionMemory/ | lsp/ | plugins/ | tips/ | settingsSync/ | policyLimits/ | voice.ts"]
    end

    subgraph BOTTOM["底层"]
        direction LR
        subgraph L5["🔒 状态与安全层"]
            STATE_MOD["state/store.ts | AppStateStore.ts | Task.ts<br/>hooks/ | permissions.ts | sandbox/"]
        end
        subgraph L6["🖥️ UI 层"]
            UI_MOD["ink.ts | ink/ (引擎)<br/>components/ | REPL.tsx<br/>vim/ | keybindings/"]
        end
        subgraph L7["🏗️ 基础设施层"]
            INFRA_MOD["bootstrap/ | utils/git/<br/>utils/bash/ | native-ts/<br/>remote/ | bridge/ | server/"]
        end
    end

    L1 --> L2
    L2 --> L3
    L3 --> L4
    L4 --> BOTTOM
```

### 4.2 关键依赖关系说明

| 依赖方向 | 说明 |
|---------|------|
| 入口层 --> 核心引擎 | `main.tsx` 创建 `QueryEngine`，启动 `query()` 循环 |
| 核心引擎 --> API 通信 | `query.ts` 调用 `queryModelWithStreaming()` 发起 API 请求 |
| 核心引擎 --> 工具系统 | `query.ts` 调用 `runTools()` 执行 AI 返回的工具调用 |
| 核心引擎 --> 命令系统 | `processUserInput()` 解析斜杠命令，分发到 `commands.ts` |
| 工具系统 --> 服务层 | MCPTool 调用 MCP 客户端；AgentTool 创建子会话 |
| 工具系统 --> 状态层 | 工具执行前检查权限；执行后通过 Hook 拦截 |
| UI 层 --> 核心引擎 | REPL 组件调用 `QueryEngine.submitMessage()` |
| UI 层 --> 状态层 | 组件通过 `useSyncExternalStore` 订阅 `AppState` |
| 服务层 --> 基础设施 | OAuth 使用 keychain；MCP 使用网络传输；压缩使用 API |

---

## 5. 开发者快速指南

### 5.1 建议阅读顺序

如果你是第一次接触这个代码库，建议按以下顺序阅读：

```
第 1 步: 理解全景
  --> 本文档 (00-overview.md)
  --> 快速浏览整体架构图和核心数据流

第 2 步: 理解启动流程
  --> 01-entrypoints-bootstrap.md
  --> 关注: cli.tsx -> main.tsx -> init.ts -> setup.ts -> REPL
  --> 关键文件: src/entrypoints/cli.tsx, src/main.tsx

第 3 步: 理解对话引擎
  --> 02-core-engine.md
  --> 关注: QueryEngine.submitMessage() -> query() -> queryLoop()
  --> 关键文件: src/QueryEngine.ts, src/query.ts

第 4 步: 理解工具系统
  --> 03-tool-system.md
  --> 关注: Tool 类型定义 -> buildTool() -> 工具执行流程
  --> 关键文件: src/Tool.ts, src/tools.ts, src/services/tools/toolExecution.ts

第 5 步: 理解 UI 渲染
  --> 04-ui-components.md
  --> 关注: 自定义 Ink -> REPL 组件 -> 消息渲染管线
  --> 关键文件: src/screens/REPL.tsx, src/ink/ink.tsx

第 6 步: 理解命令和服务
  --> 05-commands-services.md
  --> 关注: 命令注册 -> API 通信 -> MCP 连接
  --> 关键文件: src/commands.ts, src/services/api/claude.ts

第 7 步: 理解状态和权限
  --> 06-state-permissions.md
  --> 关注: Store 设计 -> 权限模式 -> Hooks 系统
  --> 关键文件: src/state/store.ts, src/types/permissions.ts

第 8 步: 按需深入特殊功能
  --> 07-special-features.md
  --> 按兴趣选择: 语音 / 远程 / 安全 / 构建 等
```

### 5.2 关键入口文件速查

| 场景 | 从哪个文件开始 |
|------|-------------|
| 理解启动过程 | `src/entrypoints/cli.tsx` |
| 理解全局初始化 | `src/entrypoints/init.ts` |
| 理解会话初始化 | `src/setup.ts` |
| 理解 CLI 参数 | `src/main.tsx` (第 902 行起) |
| 理解对话循环 | `src/query.ts` (queryLoop 函数) |
| 理解会话管理 | `src/QueryEngine.ts` |
| 理解工具定义 | `src/Tool.ts` (Tool 类型 + buildTool) |
| 理解工具注册 | `src/tools.ts` (getAllBaseTools) |
| 理解工具执行 | `src/services/tools/toolExecution.ts` |
| 理解 REPL UI | `src/screens/REPL.tsx` |
| 理解渲染引擎 | `src/ink/ink.tsx` |
| 理解命令系统 | `src/commands.ts` |
| 理解 API 调用 | `src/services/api/claude.ts` |
| 理解 MCP 客户端 | `src/services/mcp/client.ts` |
| 理解状态管理 | `src/state/store.ts` + `src/state/AppStateStore.ts` |
| 理解权限模型 | `src/types/permissions.ts` |
| 理解 Hook 系统 | `src/schemas/hooks.ts` + `src/types/hooks.ts` |
| 理解系统提示词 | `src/constants/prompts.ts` |
| 理解上下文管理 | `src/context.ts` |

### 5.3 代码量级参考

| 模块 | 核心文件 | 约代码行数 | 复杂度 |
|------|---------|----------|--------|
| API 查询 | `claude.ts` | 3400+ | 极高 |
| MCP 客户端 | `mcp/client.ts` | 3300+ | 极高 |
| REPL 界面 | `REPL.tsx` | 5000+ | 极高 |
| 对话循环 | `query.ts` | 1700+ | 高 |
| 命令注册 | `commands.ts` | 755+ | 中 |
| 工具执行 | `toolExecution.ts` | 500+ | 高 |
| 重试逻辑 | `withRetry.ts` | 500+ | 中 |
| 压缩服务 | `compact.ts` | 500+ | 中 |

---

## 6. 关键设计决策

### 6.1 对话循环: while(true) 而非递归

早期版本使用递归调用 `query()`，在长对话中导致栈溢出。当前版本重构为 `while (true)` + 显式 `State` 结构：
- 栈深度恒定，不随对话轮次增长
- 所有继续决策通过 `state.transition` 显式化
- 10+ 种恢复路径（reactive compact、max_output_tokens escalation 等）全部在循环内处理

### 6.2 工具系统: 对象字面量 + 工厂函数，而非类继承

所有工具通过 `buildTool()` 工厂函数创建，使用对象字面量而非抽象类。这个选择带来：
- `buildTool()` 自动注入 fail-closed 安全默认值（`isConcurrencySafe: false`，`isReadOnly: false`）
- MCP 工具可以通过对象展开（spread）从模板创建，避免类继承复杂性
- 每个工具都是独立的值对象，无隐式状态

### 6.3 UI: 自定义 Ink 框架

Claude Code 没有直接使用社区版 Ink，而是 fork 并大幅改造：
- **自定义 React reconciler** 适配 React 19
- **双缓冲帧渲染** + 16ms 节流，避免终端闪烁
- **Screen 缓冲区** 使用 CharPool/StylePool 字符串驻留，降低 GC 压力
- **硬件滚动提示** (DECSTBM + SU/SD)，利用终端原生滚动能力
- **仿浏览器事件系统**，支持捕获/冒泡两阶段分发

### 6.4 权限: 多源竞速 + ResolveOnce

权限系统支持多个并发审批源（本地 UI、远程桥接、Hook、YOLO 分类器）同时运行，首个完成者获胜：
- `ResolveOnce` 模式通过原子 `claim()` 确保单次解析
- 避免用户在本地确认后远程又弹出确认
- 分类器可以在后台静默审批，减少用户交互

### 6.5 启动优化: 极致的并行化和延迟加载

启动流程精心设计了多处并行化：
- 模块 import 期间并行启动 MDM 子进程 + Keychain 预读取（利用 ~135ms 的 import 窗口）
- `setup()` 与命令加载/Agent 定义加载并行执行
- MCP 配置解析与 trust dialog 重叠
- OpenTelemetry (~400KB) 延迟到遥测初始化时才加载
- React 组件（App、REPL）延迟到 `launchRepl()` 时才 import
- `--bare` 模式跳过 hooks/LSP/插件/记忆等全部非核心初始化

### 6.6 上下文管理: 多层渐进式压缩

为了在有限的上下文窗口内维持长对话，系统采用 5 层压缩策略：

| 层级 | 策略 | 触发时机 | 特性门控 |
|------|------|---------|---------|
| 1 | Snip Compact | 每轮循环开始 | HISTORY_SNIP |
| 2 | Microcompact | 每轮循环开始 | 默认启用 |
| 3 | Context Collapse | 每轮循环开始 | CONTEXT_COLLAPSE |
| 4 | Auto Compact | token 超阈值时 | 默认启用 |
| 5 | Reactive Compact | API 返回 prompt_too_long 时 | REACTIVE_COMPACT |

### 6.7 编译时特性门控

通过 Bun 的 `feature()` 内联函数实现零开销功能开关：
- 外部构建将 `feature('X')` 替换为 `false`，esbuild 进行死代码消除
- 内部功能（语音、Coordinator、YOLO 分类器等）不会出现在外部构建产物中
- 条件导入的模块在编译时被彻底移除

### 6.8 安全纵深防御

安全机制贯穿多个层面：
- **沙箱**: 基于 `@anthropic-ai/sandbox-runtime`，限制文件系统和网络访问
- **权限**: 6 种模式 + 多源规则 + 危险模式自动剥离 + 拒绝追踪回退
- **Git 安全**: ref 名称验证（防路径遍历）、Bridge ID 校验
- **上游代理安全**: `prctl(PR_SET_DUMPABLE, 0)` 防 ptrace，token 文件删除
- **Bash 解析**: 50ms 超时 + 50000 节点上限防 DoS
- **文件操作**: 路径规范化、O_NOFOLLOW、O_EXCL
- **Settings 保护**: 沙箱始终拒绝写入 `settings.json`

### 6.9 运行时兼容

关键模块同时支持 Bun 和 Node.js：
- WebSocket: Bun 原生 WebSocket vs Node.js ws 包
- TCP 服务器: `Bun.listen` vs `net.createServer`
- 布局引擎: Yoga WASM vs 纯 TypeScript 移植
- 原生模块: Rust NAPI vs 纯 TypeScript 回退（file-index、color-diff、bash-parser）

---

## 7. 文档索引

| 文档 | 路径 | 内容 |
|------|------|------|
| 架构全景总览 | [`00-overview.md`](./00-overview.md) | 本文档 -- 项目概述、架构图、数据流、依赖关系、快速指南 |
| 文档目录索引 | [`00-index.md`](./00-index.md) | 所有文档的摘要与快速查找指南 |
| 入口与启动流程 | [`01-entrypoints-bootstrap.md`](./01-entrypoints-bootstrap.md) | CLI 入口、初始化链路、启动性能 |
| 核心引擎与对话循环 | [`02-core-engine.md`](./02-core-engine.md) | QueryEngine、对话循环、上下文管理、压缩策略 |
| 工具系统 | [`03-tool-system.md`](./03-tool-system.md) | 工具定义、注册、执行、权限、MCP 工具 |
| UI 渲染与组件系统 | [`04-ui-components.md`](./04-ui-components.md) | Ink 框架、组件树、消息渲染、Vim 模式 |
| 命令系统与服务层 | [`05-commands-services.md`](./05-commands-services.md) | 斜杠命令、API/MCP/OAuth 服务 |
| 状态管理、权限与 Hooks | [`06-state-permissions.md`](./06-state-permissions.md) | Store、Task、Hooks、权限、插件、Skills |
| 特殊功能模块 | [`07-special-features.md`](./07-special-features.md) | 语音、远程、安全、Git、遥测、构建 |
| System Prompt 与消息压缩 | [`08-system-prompt-and-compression.md`](./08-system-prompt-and-compression.md) | Prompt 三层注入、CLAUDE.md 四层加载、6层压缩策略 |
| Settings 配置与 Agent 系统 | [`09-settings-and-agent-system.md`](./09-settings-and-agent-system.md) | 六层配置优先级、Agent 进程内隔离、Coordinator、Swarm |
| Memory 与数据持久化 | [`10-memory-and-persistence.md`](./10-memory-and-persistence.md) | Memdir 记忆系统、Sonnet 语义检索、JSONL 持久化 |

---

> 本文档基于 Claude Code v2.1.88 反编译源码分析。所有文件路径均相对于项目根目录。
