# Claude Code CLI 架构文档索引

> 基于 Claude Code v2.1.88 源码分析

---

## 文档目录

| 编号 | 文档 | 内容摘要 |
|------|------|---------|
| 00 | [架构全景总览](./00-overview.md) | 项目概述、整体架构图、核心数据流、模块依赖关系、开发者快速指南、关键设计决策 |
| 01 | [入口与启动流程](./01-entrypoints-bootstrap.md) | CLI 入口分发、Bootstrap 阶段、Commander 命令解析、init/setup 初始化链路、交互式/非交互式分支、启动性能优化 |
| 02 | [核心引擎与对话循环](./02-core-engine.md) | QueryEngine 会话管理、query() 对话循环、上下文管理(System Prompt/CLAUDE.md/Git)、多层压缩策略、Coordinator 协调器、成本追踪、消息数据结构 |
| 03 | [工具系统](./03-tool-system.md) | Tool 类型定义与 buildTool 模式、工具注册/发现/加载、执行流程(编排/串并行/流式)、权限集成、BashTool/FileEditTool/AgentTool 等关键工具、MCP 工具、ToolSearch 延迟加载 |
| 04 | [UI 渲染与组件系统](./04-ui-components.md) | 自定义 Ink 框架(React+Yoga+双缓冲)、组件层次(App/REPL/FullscreenLayout)、消息渲染流水线、虚拟滚动、Markdown/Diff 渲染、用户输入处理、Vim 模式、快捷键系统、屏幕导航 |
| 05 | [命令系统与服务层](./05-commands-services.md) | 斜杠命令体系(90+ 命令)、命令注册/分发/生命周期、服务层架构(API/MCP/OAuth/Analytics/Compact/Memory/LSP)、API 通信与重试、MCP 协议7种传输、OAuth PKCE 流程 |
| 06 | [状态管理、权限与 Hooks](./06-state-permissions.md) | 自研 Store 响应式状态、AppState 类型、Task 系统(7种任务类型)、Hooks 系统(27种事件x4种Hook类型)、6种权限模式、权限规则/决策/分类器、插件系统、Skills 系统、数据迁移 |
| 07 | [特殊功能模块](./07-special-features.md) | 语音交互、远程 Agent(CCR)、IDE 桥接(Remote Control)、Server 模式、沙箱安全、Git/GitHub 集成、MCP 实现细节、遥测监控(OpenTelemetry)、Buddy 伴侣、记忆目录(Memdir)、上游代理、原生 TS 模块、构建系统、第三方依赖 |
| 08 | [System Prompt 构建与消息压缩](./08-system-prompt-and-compression.md) | System Prompt 三层注入架构、静态/动态分离与缓存优化、CLAUDE.md 四层加载(Managed/User/Project/Local)、@include 指令、6层消息压缩策略(Tool Result Budget → Snip → Microcompact → Context Collapse → Auto Compact → Reactive Compact)、Token 预算管理 |
| 09 | [Settings 配置与 Agent 子进程系统](./09-settings-and-agent-system.md) | 六层配置优先级、policySettings 首源胜出、三级缓存、安全设计(敏感配置排除项目源)、远程托管设置(MDM)、Agent 进程内隔离(非子进程)、Fork Prompt Cache 优化、Coordinator Worker 模式、SendMessage 多模式路由、Swarm 三种后端(tmux/iTerm2/In-Process) |
| 10 | [Memory 记忆系统与数据持久化](./10-memory-and-persistence.md) | 四类型记忆分类、Memdir 文件系统存储、Sonnet 驱动语义检索、记忆提取触发机制、Session Memory vs Auto Memory、团队记忆同步、JSONL 持久化、批量写入优化、大粘贴 Hash 分流、工具结果外部存储、FileIndex 模糊搜索引擎 |

---

## 按主题快速查找

### 想了解启动流程?
-> [01-entrypoints-bootstrap.md](./01-entrypoints-bootstrap.md) -- 从 `claude` 命令到 REPL 交互循环的完整链路

### 想了解 AI 对话是怎么工作的?
-> [02-core-engine.md](./02-core-engine.md) -- QueryEngine、query() 循环、上下文组装、压缩策略

### 想了解工具是怎么执行的?
-> [03-tool-system.md](./03-tool-system.md) -- 工具定义、注册、编排、执行、权限检查的完整流程

### 想了解终端 UI 怎么渲染的?
-> [04-ui-components.md](./04-ui-components.md) -- 自定义 Ink、React 组件树、虚拟滚动、Vim 模式

### 想了解斜杠命令和后端服务?
-> [05-commands-services.md](./05-commands-services.md) -- 命令系统、API/MCP/OAuth 服务层

### 想了解状态管理和权限控制?
-> [06-state-permissions.md](./06-state-permissions.md) -- Store、Task、Hooks、权限模型、插件/Skills

### 想了解特殊功能(语音/远程/安全等)?
-> [07-special-features.md](./07-special-features.md) -- 语音、远程执行、沙箱、Git 集成、遥测、构建系统

### 想了解 System Prompt 怎么构建的? 消息压缩怎么工作?
-> [08-system-prompt-and-compression.md](./08-system-prompt-and-compression.md) -- 三层 Prompt 注入、CLAUDE.md 加载、6层压缩策略

### 想了解配置系统和多 Agent 协作?
-> [09-settings-and-agent-system.md](./09-settings-and-agent-system.md) -- 六层配置优先级、Agent 进程内隔离、Coordinator、Swarm

### 想了解记忆系统和数据如何持久化?
-> [10-memory-and-persistence.md](./10-memory-and-persistence.md) -- Memdir 记忆存储、Sonnet 语义检索、JSONL 持久化、FileIndex
