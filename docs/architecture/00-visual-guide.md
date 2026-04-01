# Claude Code CLI 可视化架构指南

> **给谁看：** 本文专为**非开发者**撰写——产品经理、设计师、管理者、市场人员，
> 以及一切好奇 "Claude Code 到底是怎么工作的" 但不想看代码的朋友。
>
> 全文零代码、零术语（必须出现的术语会用括号附上简单解释），
> 每一个概念都配有大幅 ASCII 示意图和日常生活类比。

---

## 目录

1. [Claude Code 是什么？（智能办公室类比）](#图1-claude-code-是什么智能办公室类比)
2. [一次对话的完整旅程](#图2-一次对话的完整旅程)
3. [工具箱全景图](#图3-工具箱全景图)
4. [大脑怎么记住之前说的话？](#图4-大脑怎么记住之前说的话)
5. [安全保障体系](#图5-安全保障体系)
6. [多人协作（多 Agent 系统）](#图6-多人协作多-agent-系统)
7. [启动过程（开机流程）](#图7-启动过程开机流程)
8. [配置系统（层层覆盖）](#图8-配置系统层层覆盖)
9. [插件和技能生态](#图9-插件和技能生态)
10. [数据的一生](#图10-数据的一生)

---

## 图1: Claude Code 是什么？（智能办公室类比）

想象你走进了一间 **AI 助手的办公室**。
你是客户，Claude 是坐在办公桌后面的智能助手，桌上摆满了各种办公工具。

```mermaid
graph TD
    subgraph office["🏢 Claude Code — AI 智能办公室"]
        User["👤 你（用户）<br>我想改一下这个文件..."]
        Brain["🧠 AI 大脑（Claude）<br>坐在办公桌后面<br>随时准备帮你干活"]
        User -->|走进办公室| Brain

        subgraph toolbox["🧰 办公桌上的工具箱"]
            subgraph row1[" "]
                T1["📁 文件柜<br>读写文件"]
                T2["💻 终端机<br>执行命令"]
                T3["🔍 搜索引擎<br>搜索代码"]
                T4["📝 笔记本<br>记录笔记"]
            end
            subgraph row2[" "]
                T5["🌐 浏览器<br>上网查资料"]
                T6["📻 对讲机<br>呼叫助手"]
                T7["📋 计划板<br>管理任务"]
                T8["🔌 外接设备<br>第三方插件"]
            end
        end

        Brain -->|桌上的工具| toolbox

        Reception["🛎️ 前台接待（命令系统）<br>/commit /help /compact"]
        Security["🔒 门卫/保安（权限系统）<br>这个操作需要你确认<br>这个操作被禁止"]
    end
```

**一句话总结：**
Claude Code 就像一间智能办公室——你说需求，AI 助手 Claude 用桌上的 40 多种工具帮你干活，
前台帮你对接服务，保安确保一切安全合规。

---

## 图2: 一次对话的完整旅程

当你在终端里打字，按下回车，到最终看到回复，
中间经历了一条完整的 "流水线"。就像寄快递一样，包裹经过层层处理才到你手上。

```mermaid
graph TD
    subgraph main["一次对话的完整旅程"]
        subgraph phase1["📥 接收阶段"]
            S1["1️⃣ 你打字<br>在终端输入问题或指令"]
            S2["2️⃣ 前台接收<br>检查是命令、问题还是闲聊"]
            S1 --> S2
        end

        subgraph phase2["🧠 处理阶段"]
            S3["3️⃣ 大脑思考<br>Claude 理解需求，想出方案"]
            S4["4️⃣ 挑选工具<br>从工具箱里挑出合适的工具"]
            S5["5️⃣ 保安审批<br>检查工具是否有权限"]
            S6["6️⃣ 执行任务<br>改文件、跑命令、搜索..."]
            S3 --> S4 --> S5 --> S6
        end

        subgraph phase3["📤 输出阶段"]
            S7["7️⃣ 整理结果<br>把原始结果加工成易懂的答案"]
            S8["8️⃣ 回复你<br>格式化后显示在终端屏幕上"]
            S7 --> S8
        end

        phase1 --> phase2 --> phase3
        S6 -->|还没完成，继续循环| S3
    end
```

**一句话总结：**
你的每条消息都经过"接收 -> 思考 -> 选工具 -> 审批 -> 执行 -> 整理 -> 回复"七站流水线，
如果任务没做完，Claude 会自动循环 "思考 -> 执行" 直到搞定。

---

## 图3: 工具箱全景图

Claude Code 内置了 **40 多种工具**，就像一个超级工具箱。
这些工具按用途分成七大类：

```mermaid
graph TD
    Title["🧰 Claude Code 的超级工具箱（40+ 种工具）"]

    subgraph file["📁 文件工具 — 操作你电脑上的文件"]
        F1["Read<br>读文件（文本/图片/PDF）"]
        F2["Edit<br>改文件（精确替换）"]
        F3["Write<br>写文件（创建新文件）"]
        F4["NotebookEdit<br>改笔记本"]
    end

    subgraph terminal["💻 终端工具 — 在命令行里执行操作"]
        B1["Bash 命令执行器<br>运行命令 / Git 操作<br>后台任务 / 安全沙箱"]
    end

    subgraph search["🔍 搜索工具 — 在代码库中找东西"]
        S1["Glob 按文件名搜索"]
        S2["Grep 按内容搜索"]
        S3["ToolSearch 搜索更多工具"]
    end

    Title --- file
    Title --- terminal
    Title --- search
```

```mermaid
graph TD
    Title2["🧰 工具箱（续）"]

    subgraph agent["🤖 AI 助手工具 — 派遣分身去干活"]
        A1["Agent 创建子助手"]
        A2["SendMessage 给子助手发消息"]
        A3["TeamCreate 创建助手团队"]
    end

    subgraph web["🌐 网络工具 — 上网找资料"]
        W1["WebSearch 网络搜索"]
        W2["WebFetch 抓取网页内容"]
    end

    subgraph task["📋 任务管理工具 — 管理待办事项"]
        K1["TaskCreate 建任务"]
        K2["TaskGet 查状态"]
        K3["TaskUpdate 更新进度"]
        K4["TaskStop 终止任务"]
    end

    subgraph mcp["🔌 外部插件工具 — 连接第三方服务"]
        M1["MCP 工具（模型上下文协议）<br>GitHub / 数据库 / Slack / 自定义服务<br>理论上数量无限"]
    end

    Title2 --- agent
    Title2 --- web
    Title2 --- task
    Title2 --- mcp
```

**一句话总结：**
Claude Code 有 40 多种内置工具，分为文件操作、命令执行、代码搜索、AI 分身、
网络查询、任务管理、外部插件七大类，覆盖了软件开发的方方面面。

---

## 图4: 大脑怎么记住之前说的话？

AI 的 "记忆" 其实是有限的。
想象 Claude 的大脑就是一张 **书桌**——桌面大小有限，不能把所有文件都摊开。

```mermaid
graph LR
    subgraph desk["🖥️ 书桌（上下文窗口 — AI 一次能看到的信息量有限）"]
        D1["💬 当前对话<br>你刚说的话、AI 的回复"]
        D2["📄 最近的工具结果<br>刚才读的文件、跑的命令"]
        D3["⚙️ 系统指令<br>AI 的行为规则"]
        D4["📊 项目情报<br>Git 状态、环境信息"]
    end
```

桌面快满了怎么办？Claude 有**四大记忆管理策略**：

```mermaid
graph LR
    subgraph t1["招数一：旧文件归档"]
        C1["聊天1 + 聊天2 + 聊天3<br>占很大空间"]
        C2["📋 摘要：之前讨论了 xxx<br>占很小空间"]
        C1 -->|像写会议纪要| C2
    end

    subgraph t2["招数二：便签 CLAUDE.md"]
        N1["📌 CLAUDE.md<br>这个项目用 TypeScript<br>提交代码前要跑测试<br>不要修改 config.json"]
    end

    subgraph t3["招数三：笔记本 Memory 系统"]
        M1["📒 MEMORY.md 索引"]
        M2["记忆1：架构偏好"]
        M3["记忆2：踩过的坑"]
        M4["记忆3：常用命令"]
        M1 --> M2
        M1 --> M3
        M1 --> M4
    end

    subgraph t4["招数四：渐进式清理（六层）"]
        P0["第0步：大文件存抽屉"]
        P1["第1步：旧聊天裁掉"]
        P2["第2步：旧结果只留标题"]
        P3["第3步：早期对话折叠"]
        P4["第4步：太长则写总结重开"]
        P5["第5步：紧急强制压缩"]
        P0 --> P1 --> P2
        P3 --> P4 --> P5
    end
```

**一句话总结：**
Claude 的 "书桌" 空间有限，它通过压缩旧对话、阅读项目便签（CLAUDE.md）、
维护长期记忆本（Memory），以及六层渐进式清理策略来管理有限的记忆空间。

---

## 图5: 安全保障体系

Claude Code 处理的是你的真实文件和代码，安全至关重要。
它的安全体系就像一个 **银行保险库**，有多层防护：

```mermaid
graph TD
    subgraph layer1["🚪 第1层：门卫（基础权限模式）"]
        P1["🔒 plan 模式<br>最严格：只能看、只能规划"]
        P2["🛡️ default 模式<br>普通：敏感操作需确认"]
        P3["⚡ auto 模式<br>自动：AI 自动判断安全性"]
        P4["🔓 bypassPermissions<br>最宽松：跳过大部分检查"]
        P1 -->|放宽| P2 -->|放宽| P3 -->|放宽| P4
    end

    subgraph layer2["🔎 第2层：审批流程（Hook 拦截）"]
        H1["Claude 要改文件"] --> H2["PreToolUse 拦截<br>放行 / 阻止 / 修改"]
        H2 --> H3["实际执行改文件"] --> H4["PostToolUse 拦截<br>检查结果 / 触发后续"]
    end

    layer1 --> layer2
```

```mermaid
graph TD
    subgraph layer3["📦 第3层：沙箱隔离（限制活动范围）"]
        S1["📂 文件系统限制<br>只能访问项目目录"]
        S2["🌐 网络限制<br>只能访问指定网址"]
        S3["🔧 配置保护<br>不能修改安全设置"]
    end

    subgraph layer4["📹 第4层：监控摄像（遥测和日志）"]
        L1["📝 操作日志"]
        L2["💰 花费追踪"]
        L3["❌ 错误记录"]
    end

    layer3 --> layer4
```

**一句话总结：**
Claude Code 有四层安全防护——权限模式控制大方向，Hook 拦截做精细审批，
沙箱隔离限制活动范围，遥测日志记录一切操作以备审计。

---

## 图6: 多人协作（多 Agent 系统）

当任务比较复杂时，Claude 不是一个人在战斗——它可以组建 **项目团队**，
把大任务拆分给多个 "分身" 同时处理。

```mermaid
graph LR
    subgraph team["👥 多 Agent 协作 — 项目团队"]
        Boss["👔 项目主管 Coordinator<br>分配任务、综合结果"]
        Explorer["🔍 调研员<br>搜索代码 / 读文档"]
        Worker["🔧 执行者<br>改代码 / 写文件"]
        Verifier["✅ 质检员<br>跑测试 / 验证结果"]
        Boss --> Explorer
        Boss --> Worker
        Boss --> Verifier
    end

    subgraph workflow["📋 典型工作流程"]
        W1["主管分配"] --> W2["调研员调查"] --> W3["主管汇总"]
        W3 --> W4["执行者实施"] --> W5["质检员验收"]
    end
```

```mermaid
graph LR
    subgraph comm["📡 沟通方式"]
        C1["📨 任务通知<br>干完活自动通知结果"]
        C2["📻 发消息 SendMessage<br>像用对讲机沟通"]
        C3["📄 共享文件 Scratchpad<br>像共享文档协作"]
    end

    subgraph isolation["🏗️ 隔离模式"]
        I1["📁 普通模式<br>同一目录（小任务）"]
        I2["🌿 Worktree 模式<br>独立代码副本（大任务）"]
        I3["☁️ 远程模式<br>云端独立环境（最高隔离）"]
    end
```

**一句话总结：**
Claude 可以组建项目团队——主管分配任务，调研员负责看，执行者负责做，
质检员负责查，多人协作互不干扰，大任务也能高效完成。

---

## 图7: 启动过程（开机流程）

每次你在终端输入 `claude` 命令启动 Claude Code，就像一家 **餐厅开门营业**：

```mermaid
graph TD
    subgraph boot["🍽️ 启动过程 — 餐厅开门营业"]
        S1["1️⃣ 程序启动 🏪<br>老板到店，输入 claude 命令"]
        S2["2️⃣ 初始化配置 💡<br>开灯开空调，读取各层配置"]
        S3["3️⃣ 加载工具 🥗<br>检查食材，准备 40+ 种工具"]
        S4["4️⃣ 准备界面 🍽️<br>准备餐具，渲染终端界面"]
        S1 --> S2 --> S3 --> S4
        S5["5️⃣ 信任确认 📋<br>挂营业牌：你信任这个项目吗？"]
        S6["6️⃣ 读取记忆 📖<br>新员工培训，读取 CLAUDE.md"]
        S7["7️⃣ 等待输入 🚪<br>等客人进门，一切就绪！"]
        S4 --> S5 --> S6 --> S7
    end
```

```mermaid
graph TD
    subgraph chain["⛓️ 完整流程链（源码文件）"]
        C1["入口分发 cli.tsx"] --> C2["参数解析 main.tsx"]
        C2 --> C3["全局初始化 init.ts"]
        C3 --> C4["会话初始化 setup.ts"]
        C4 --> C5["信任确认 对话框"]
        C5 --> C6["REPL 启动 交互循环"]
    end
```

**一句话总结：**
Claude Code 启动就像餐厅开门——先检查环境、加载配置、准备工具、确认信任，
最后一切就绪，等你 "下单"。

---

## 图8: 配置系统（层层覆盖）

Claude Code 的配置就像 **穿衣服**——一层套一层，外面的会遮住里面的：

```mermaid
graph BT
    subgraph config["👔 配置系统 — 穿衣服类比（越外层优先级越高）"]
        L1["🩲 第1层：内衣（用户全局）<br>~/.claude/settings.json<br>你个人的全局偏好"]
        L2["👕 第2层：衬衫（项目级 projectSettings）<br>.claude/settings.json<br>团队共享的项目配置"]
        L3["🧶 第3层：毛衣（项目本地 localSettings）<br>.claude/settings.local.json<br>你的私有偏好，不上传代码库"]
        L4["🧥 第4层：外套（命令行参数 flagSettings）<br>启动时临时指定 --model opus"]
        L5["🌧️ 第5层：雨衣（企业策略 policySettings）<br>公司 IT 统一下发，谁都盖不过"]

        L1 -->|被覆盖| L2
        L2 -->|被覆盖| L3
        L3 -->|被覆盖| L4
        L4 -->|被覆盖| L5
    end

    subgraph rule["📏 合并规则"]
        R1["同一配置项，谁在最外层就听谁的<br>用户设 model=sonnet + 项目本地设 model=opus → 生效 opus<br>企业策略说只能用 haiku → 最终就是 haiku"]
    end
```

**一句话总结：**
配置像穿衣服——用户全局设置是内衣，项目设置是衬衫，本地设置是毛衣，
命令行参数是外套，企业策略是雨衣；外层永远覆盖内层，企业策略说了算。

---

## 图9: 插件和技能生态

Claude Code 不只是一个封闭的工具，它像手机一样有自己的 "应用商店"：

```mermaid
graph LR
    subgraph builtin["📦 内置应用 — 出厂自带"]
        subgraph row1[" "]
            B1["Bash"]
            B2["Read"]
            B3["Edit"]
            B4["Write"]
            B5["Glob"]
        end
        subgraph row2[" "]
            B6["Grep"]
            B7["Agent"]
            B8["WebSearch"]
            B9["WebFetch"]
            B10["TaskCreate"]
        end
        B11["... 40+ 核心工具"]
    end

    subgraph skills["⚡ 技能包 — 可学习的新能力"]
        SK1["/commit 自动提交"]
        SK2["/review-pr 自动审查"]
        SK3["/simplify 简化代码"]
        SK4["/loop 定期执行"]
        SK5["/schedule 定时任务"]
        SK6["... 90+ 斜杠命令"]
    end

    builtin --> skills
```

```mermaid
graph LR
    subgraph plugins["🧩 插件 — 第三方扩展"]
        PL1["安装后获得新工具、新命令、新配置"]
        PL2["可带 MCP 服务 / Hook 规则 / 技能包"]
        PL3["企业可控制允许安装哪些"]
    end

    subgraph mcps["🔌 MCP 服务 — 外部工具连接器"]
        M1["GitHub 服务"]
        M2["数据库查询"]
        M3["Slack 发消息"]
        M4["你自己的服务"]
    end

    plugins --> mcps
```

**一句话总结：**
Claude Code 有四层扩展能力——内置工具开箱即用，技能包提供快捷操作，
插件市场可安装第三方扩展，MCP 协议可连接任意外部服务，生态无限扩展。

---

## 图10: 数据的一生

你和 Claude Code 对话产生的数据，从诞生到被再次使用，走过一条完整的生命旅程：

```mermaid
graph TD
    subgraph lifecycle["🔄 数据的一生 — 完整生命周期"]
        A1["1️⃣ 诞生 — 你输入一句话：帮我修复这个 bug"]
        A2["2️⃣ 加工处理 — 输入+系统指令+记忆+环境 → 发送给 Claude API → 最终回复"]
        A3["3️⃣ 实时呈现 — 打字机效果，流式输出到终端"]
        A4["4️⃣ 历史存档 — ~/.claude/projects/session_xxx.json，默认保留 30 天"]
        A5["5️⃣ 记忆提取 — 自动提取关键信息写入 MEMORY.md"]
        A6["6️⃣ 下次召回 — 读取 CLAUDE.md 和 MEMORY.md，智能检索相关记忆"]

        A1 --> A2 --> A3
        A3 --> A4 --> A5 --> A6
        A6 -->|新对话开始| A1
    end
```

**一句话总结：**
数据从你输入的那一刻诞生，经历加工处理、实时显示、本地存档、记忆提取，
最终在下次对话时被自动召回——形成一个不断学习和记忆的循环。

---

## 全文总结

> **一句话概括 Claude Code：**
>
> Claude Code 是一间 AI 智能办公室——你提需求，Claude 用 40 多种工具帮你干活，
> 有安保系统确保安全，有团队协作处理大任务，有记忆系统越用越懂你，
> 还有插件商店让能力无限扩展。
