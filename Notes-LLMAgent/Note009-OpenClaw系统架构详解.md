# OpenClaw 系统架构详解

> 原文：[OpenClaw Architecture, Explained: How It Works](https://ppaolo.substack.com/p/openclaw-system-architecture-overview)  
> 作者：Paolo Perazzo | 发布日期：2026-02-11

---

## 1. Introduction

OpenClaw 是一个运行在**你自己基础设施**上的个人 AI 助手平台（笔记本、VPS、Mac Mini 或云容器均可）。它将 AI 模型与工具连接到你日常使用的即时通讯应用（WhatsApp、Telegram、Discord、Slack、Signal、iMessage、Microsoft Teams 等）。

**核心理念**：  
> LLM 提供智能，OpenClaw 提供操作系统。

OpenClaw 不是对 AI API 的简单封装，而是一个**为 AI Agent 提供运行环境的操作系统**，内置了：会话管理、记忆系统、工具沙箱、访问控制和编排逻辑。

---

## 2. High-Level Architecture: Hub-and-Spoke Model

<div align="center">
<img src="https://substackcdn.com/image/fetch/$s_!e6qy!,w_1272,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ff5354755-5809-446a-844a-498454484cfd_1205x1094.png" alt="image-20260310110037542" style="zoom:80%;" />
</div>

OpenClaw 以单一 **Gateway（网关）** 为核心，采用 Hub-and-Spoke 架构：
- **Gateway**：WebSocket 服务器，连接所有消息平台和控制接口，将路由后的消息派发给 Agent Runtime
- **Agent Runtime**：端到端运行 AI 循环——汇聚上下文 → 调用模型 → 执行工具 → 持久化状态

**接口层与运行层的分离**，使得你通过任意消息应用都能访问同一个持久化 AI 助手。

---

## 3. Core Components

### 3.1 Channel Adapters

每个消息平台有专属适配器（`src/telegram/`、`src/discord/`、`src/slack/`、`src/imessage/` 等），实现统一接口，负责：

| 职责 | 说明 |
|---|---|
| **认证（Authentication）** | WhatsApp 用 QR 码配对；Telegram/Discord 用 Bot Token；iMessage 需要 macOS 原生集成 |
| **入站消息解析（Inbound Message Parsing）** | 提取文本、媒体附件、表情回复、线程上下文，并统一格式 |
| **访问控制（Access Control）** | 白名单（`allowFrom`）、DM 配对策略（pairing/open/disabled）、群组策略 |
| **出站消息格式化（Outbound Message Formatting）** | 处理各平台 Markdown 方言、消息大小限制、媒体上传、打字指示符 |




### 3.2 Control Interfaces

| 接口 | 说明 |
|---|---|
| **Web UI** | 基于 Lit web components，Gateway 内置服务，默认 `http://127.0.0.1:18789/` |
| **CLI** | 基于 Commander.js，支持 `openclaw gateway`、`openclaw agent`、`openclaw doctor` 等命令 |
| **macOS 菜单栏应用** | Swift 编写，管理 Gateway 生命周期，支持 Voice Wake、WebChat 嵌入视图 |
| **移动端（iOS/Android）** | 通过 WebSocket 作为节点连接，暴露设备能力（摄像头、屏幕录制、位置服务） |

### 3.3 Gateway Control Plane

- 位于 `src/gateway/server.ts`，基于 Node.js 22+，使用 `ws` WebSocket 库
- 默认绑定 `127.0.0.1:18789`（仅本地回环，安全隔离）
- 设计原则：
  - 每台主机**只有一个** Gateway（防止 WhatsApp 协议冲突）
  - 全部协议使用 TypeBox 定义的 JSON Schema 类型验证
  - **事件驱动**，客户端订阅事件而非轮询
  - 所有副作用操作需要**幂等性键**，保证重试安全

### 3.4 Agent Runtime

实现于 `src/agents/piembeddedrunner.ts`，每轮对话执行四步：

- **Step 1：Session Resolution（会话解析）**: 将消息映射到对应会话（也是安全边界）
  - 直接消息 → `main`（完整权限）
  - 频道私信 → `dm:<channel>:<id>`（沙箱隔离）
  - 群组聊天 → `group:<channel>:<id>`（沙箱隔离）

- **Step 2：Context Assembly（上下文装配）**
  - 加载持久化的会话历史
  - 读取工作区配置文件（`AGENTS.md`、`SOUL.md`、`TOOLS.md`）构建系统 Prompt
  - 通过语义搜索拉取相关历史记忆

- **Step 3：Execution Loop（执行循环）**
  - 流式调用模型（Claude / GPT / Gemini / 本地模型）
  - 拦截并执行工具调用（bash、文件读写、浏览器自动化等）
  - 工具结果回注模型，继续生成

- **Step 4：Persist State（持久化状态）**
  - 将完整对话（消息、工具调用与结果）写回磁盘的 Session 文件





### System Prompt Architecture

OpenClaw 通过**组合多个来源**来构建最终的系统提示，而非依赖单一的静态 Prompt。所有来源叠加后，连同会话历史和当前用户消息一起送入模型。

<div align="center">
<img src="https://substackcdn.com/image/fetch/$s_!9OqU!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F7a977528-1f60-44b6-963c-9248b7b9dd0a_1966x558.png"/>
</div>


> **Skill 发现 vs. Skill 注入**：OpenClaw 能在运行时发现所有 Skill，但**不会**盲目地将全部 Skill 注入每个 Prompt。Runtime 仅将**当前轮次相关**的 Skill 选择性注入，避免无关内容膨胀上下文、降低模型性能。

> **可组合设计的价值**：这种分层组合方式意味着——你可以通过编辑工作区文件来改变 Agent 的行为、风格和任务能力，**无需修改任何源码**；而执行权限与沙箱策略则由 Runtime 强制执行，不受 Prompt 文件影响。



#### 来源一：工作区配置文件（静态基线）

| 文件 | 是否必选 | 作用 |
|---|---|---|
| `AGENTS.md` | 必选 | 核心运行指令：Agent 被允许做什么、全局约束、跨所有 Session 的不可违反规则 |
| `SOUL.md` | 可选 | 个性与语气：Agent 如何表达、如何组织回答，但不涉及工具行为或安全边界 |
| `TOOLS.md` | 可选 | 用户对工具使用方式的个人约定（非工具注册表），OpenClaw 会自动向模型提供工具定义 |

#### 来源二：动态上下文（每轮组装）

| 来源 | 说明 |
|---|---|
| **会话历史** | 当前 Session 的近期消息记录，保持对话连贯性 |
| **Skill 文件** `skills/<skill>/SKILL.md` | 完成特定任务的操作手册（类似 SOP/Playbook）；Skill 存在的前提是该文件必须存在 |
| **记忆搜索** | 语义相似的历史对话片段，让 Agent 无需用户重复即可引用数周前的上下文 |

#### 来源三：工具定义（自动生成）

| 来源 | 说明 |
|---|---|
| 内置工具 | `src/agents/pi-tools.ts` / `src/agents/openclaw-tools.ts`：bash、browser、文件操作、Canvas 等核心能力 |
| 插件工具 | 通过 `api.registerTool(toolName, toolDefinition)` 由扩展系统注册的自定义工具 |

#### 来源四：基础系统（运行时底座）

- **Pi Agent Core**：Agent 运行时库内置的基础指令，构成整个 Prompt 栈的最底层。







## 4. Interaction and Coordination Mechanism

### 4.1 Canvas and Agent-to-UI (A2UI)

<div align="center">
<img src="https://substackcdn.com/image/fetch/$s_!uZOJ!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F99d80f71-7a18-4854-9617-58e142334f6f_980x915.png"/ width="700px">
</div>

Canvas 是由 Agent 驱动的**可视化交互工作区**，运行在独立进程（默认端口 18793）。

**A2UI 工作流**：
1. Agent 调用 canvas update 方法，生成带 `a2ui-*` 属性的 HTML
2. Canvas 服务解析属性，通过 WebSocket 推送给浏览器客户端
3. 用户交互（如点击按钮）触发 Action 事件，转发给 Agent 作为工具调用
4. Agent 处理后更新 Canvas，界面自动刷新

These attributes create interactive elements without requiring the agent to write JavaScript. For example:
```html
<div a2ui-component="task-list">
  <button a2ui-action="complete" a2ui-param-id="123">
    Mark Complete
  </button>
</div>
```



### 4.2 Voice Wake & Talk Mode

- 平台：macOS、iOS、Android
- 唤醒词："Hey OpenClaw"（可自定义），或使用 Push-to-Talk 快捷键
- 音频通过 ElevenLabs 进行语音识别和 TTS 合成
- Talk Mode 支持连续对话与打断检测





### 4.3 Multi-Agent Routing

<div align="center">
<img src="https://substackcdn.com/image/fetch/$s_!MK0a!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fff9c8af5-2816-479d-9bf7-b7004909bd99_1223x497.png"/ width="900px">
</div>

可将不同频道/群组路由到完全隔离的 Agent 实例，各自拥有独立工作区、模型和配置：

```json
{
  "agents": {
    "mapping": {
      "group:discord:123456": {
        "workspace": "~/.openclaw/workspaces/discord-bot",
        "model": "anthropic/claude-sonnet-4-5"
      },
      "dm:telegram:*": {
        "workspace": "~/.openclaw/workspaces/support-agent",
        "model": "openai/gpt-4o",
        "sandbox": { "mode": "always" }
      }
    }
  }
}
```





### 4.4 Session Tools (Agent-to-Agent Communication)

| 工具 | 作用 |
|---|---|
| `sessions_list` | 发现活跃会话 |
| `sessions_send` | 向另一会话发送消息 |
| `sessions_history` | 获取其他会话的对话记录 |
| `sessions_spawn` | 以编程方式创建新会话（委托任务） |





### 4.5 定时任务与 Webhook

- **Cron Jobs**：基于配置的定时触发，例如每天 9:00 发送日报摘要
- **Webhooks**：外部系统触发 Agent 动作，例如 Gmail 邮件到达时触发处理

---






## 5. End-to-End Message Flow (以 WhatsApp 为例)

<div align="center">
<img src="https://substackcdn.com/image/fetch/$s_!qxRY!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fdc2a349c-91c9-402f-8356-16dbb2583952_1724x1327.png"/>
</div>

```
Phase 1: Ingestion       → Baileys 接收 WebSocket 事件，适配器解析消息
Phase 2: Access Control  → 白名单检查 & 会话路由（main / dm / group）
Phase 3: Context Assembly → 加载会话历史 + 构建系统 Prompt + 记忆检索
Phase 4: Model Invocation → 流式调用 AI 模型
Phase 5: Tool Execution  → 拦截工具调用，执行（可选沙箱），结果回注模型
Phase 6: Response Delivery → 格式化输出，通过 Baileys 发送，持久化会话状态
```

**典型延迟预算**：

| 阶段 | 耗时 |
|---|---|
| 访问控制 | < 10ms |
| 加载会话 | < 50ms |
| 构建 Prompt | < 100ms |
| 首 Token（网络） | 200–500ms |
| bash 工具执行 | < 100ms |
| 浏览器自动化 | 1–3s |

---






## 6. Data Storage and State Management

<div align="center">
<img src="https://substackcdn.com/image/fetch/$s_!EfuY!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fd3387882-3f4d-4d40-9ad3-f09ae4802852_1798x690.png"/>
</div>

OpenClaw 将所有数据集中存放在用户 home 目录下，便于备份、迁移和故障排查：

| 数据类型 | 存储位置 | 说明 |
|---|---|---|
| 主配置文件（JSON5）| `~/.openclaw/openclaw.json` | 支持注释与尾逗号，环境变量 > 配置文件 > 内置默认值 |
| 会话状态 | `~/.openclaw/sessions/` | append-only 事件日志，支持分支，便于历史回溯 |
| 记忆向量库 | `~/.openclaw/memory/<agentId>.sqlite` | SQLite + 向量嵌入，支持混合检索 |
| 凭据（权限 0600）| `~/.openclaw/credentials/` | 频道认证 Token、OAuth 凭据，自动排除版本控制 |





### 6.1 配置文件

主配置文件使用 **JSON5** 格式（支持注释和尾逗号），位于 `~/.openclaw/openclaw.json`。配置采用**三层覆盖**机制：

```
环境变量  >  配置文件值  >  内置默认值
```

这使得敏感 Token 可以通过环境变量注入，而静态配置保留在文件中，两者互不干扰。




### 6.2 会话状态与压缩（Session State & Compaction）

每个会话以独立文件持久化到 `~/.openclaw/sessions/`，记录对话历史、元数据及工具执行状态。

**会话标识符**编码了所属关系与信任边界：

| 会话类型 | 标识符格式 | 权限级别 |
|---|---|---|
| 操作员主会话 | `agent:<agentId>:main` | 完整主机访问 |
| 频道 DM 会话 | `agent:<agentId>:<channel>:dm:<identifier>` | 默认沙箱隔离 |
| 群组会话 | `agent:<agentId>:<channel>:group:<identifier>` | 默认沙箱隔离 |

**存储结构**：append-only 事件日志 + 支持分支，使得状态恢复、历史检视和对话树定位都非常自然。

**会话压缩（Compaction）**：当对话超出模型上下文窗口时，系统自动执行以下流程：

```
1. 记忆刷新（Memory Flush）：将重要信息提升写入记忆文件（仅当会话 workspace 可写时执行）
        ↓
2. 摘要压缩：对较旧的对话轮次生成摘要，替换原始内容
        ↓
3. 会话继续：压缩后的上下文作为新的起点，对话无缝延续
```





### 6.3 记忆系统（Memory Search）

OpenClaw 维护一套可搜索的对话记忆，在每次交互时自动检索语义相似的历史内容注入上下文，无需用户重复描述背景。

#### 存储与索引

记忆数据存放在 `~/.openclaw/memory/<agentId>.sqlite`，采用 **SQLite + 向量嵌入** 存储。消息到达时自动索引，检索使用**混合检索**策略：

- **向量语义搜索**（Semantic）：捕捉语义相似性，找到"意思接近"的历史内容
- **BM25 关键词搜索**：精确匹配 Token，找到"词汇相同"的历史内容

两者结合，兼顾召回率与精确率。



#### 结构化记忆文件

除向量索引外，还支持在 workspace 中维护结构化记忆文件：

| 文件 | 用途 | 访问范围 |
|---|---|---|
| `MEMORY.md` | 长期稳定事实（手动维护的精华信息） | 仅限 main 私密会话，**不在群组中暴露** |
| `memory/YYYY-MM-DD.md` | 每日活动原始日志 | 当日上下文参考 |



#### 嵌入模型优先级

记忆系统需要嵌入模型将文本转化为可搜索向量，按以下优先级自动选择：

```
本地模型（local.modelPath）→ OpenAI → Gemini → 禁用记忆搜索
```



#### 索引管理

- **自动重索引**：文件监听器检测到 memory 文件变更后，1.5s 防抖触发重索引
- **嵌入模型变更检测**：更换 Provider 或模型时，自动全量重索引
- **实验性全文索引**：启用 `experimental.sessionMemory: true` 后，完整会话历史也纳入搜索范围
- **向量加速**：若环境中存在 sqlite-vec，自动启用以加速 SQLite 内的向量检索




### 6.4 凭据管理

敏感认证数据统一存放在 `~/.openclaw/credentials/`，包括：

- WhatsApp 的 Baileys 会话数据
- Discord、Telegram 等平台的 OAuth 凭据
- 其他频道访问所需的密钥

文件权限限制为 **0600**（仅所有者可读写），且该目录自动被排除在版本控制之外，防止意外泄露。

---






## 7. Security Architecture

OpenClaw 通过**多层纵深防御（Defense in Depth）**实现安全保障，各层独立提供不同类型的保护，共同构成完整的安全体系。




### 7.1 Network Security

Gateway 默认仅绑定 `127.0.0.1`（本地回环接口），永不暴露公网。远程访问需通过以下方式显式配置：

| 访问方式 | 适用场景 | 说明 |
|---|---|---|
| **SSH 隧道**（推荐） | VPS / 远程主机 | 将本地端口转发到远程回环地址，流量走加密 SSH 通道 |
| **Tailscale Serve** | tailnet 内访问 | 仅对同一 Tailscale 网络内的设备暴露，HTTPS 加密 |
| **Tailscale Funnel** | 公网暴露 | 通过 Tailscale 基础设施对公网开放，**必须**启用密码认证 |

```bash
# SSH 隧道示例（VPS 推荐方案）
ssh -N -L 18789:127.0.0.1:18789 user@vps-host

# Tailscale Serve（tailnet 内 HTTPS）
# config: gateway.tailscale.mode: "serve"

# Tailscale Funnel（公网 HTTPS，必须配合密码认证）
# config: gateway.tailscale.mode: "funnel"
#         gateway.auth.mode: "password"
```



### 7.2 Authentication & Device Pairing

**Token / 密码认证**：非回环绑定时，所有 WebSocket 客户端必须在 `connect.params.auth.token` 中携带令牌；或通过 `gateway.auth.mode: "password"` 启用密码认证（Tailscale Funnel 场景必选）。

**设备配对（Device Pairing）**：在 Token 认证之上再增一道防线。每个客户端在 `connect` 握手时携带设备身份（Device ID + 密钥对）：

- **本地连接**（回环或同一 Tailscale 网络）：可配置为自动审批，简化同主机工作流
- **远程连接**：握手阶段需对挑战随机数（nonce）进行加密签名，并经过人工明确审批

审批通过后，Gateway 向该设备颁发**设备令牌**，后续连接无需重新审批。即使攻击者获取了认证 Token，若无设备身份也无法接入。

> **注意**：Control UI 需要安全上下文（HTTPS 或 localhost）才能通过 `crypto.subtle` 生成设备身份。若开启 `gateway.controlUi.allowInsecureAuth`，UI 将降级为仅 Token 认证并跳过设备配对，**这是安全降级操作**，优先使用 HTTPS（Tailscale Serve）或直接访问 `127.0.0.1`。




### 7.3 Channel Access Control

**DM 配对（Human-in-the-loop 审批）**：默认 `dmPolicy="pairing"` 策略下，未知发件人的首条消息会触发配对流程——Gateway 回复唯一配对码，操作员执行 `openclaw pairing approve <channel> <code>` 审批后，该发件人才会被加入本地白名单。可选策略：
- `"open"`：接受所有 DM（不推荐）
- `"disabled"`：完全拒绝 DM

**白名单（Allowlists）**：精确指定允许交互的手机号或用户名：
```json
// WhatsApp 示例
"channels.whatsapp.allowFrom": ["+1234567890"]
// Telegram 示例（用户名或数字 ID）
"channels.telegram.allowFrom": ["username"]
```

**群组策略**：
- `requireMention`：仅在被 @提及时响应，避免在公共群组中"全量监听"
- 群组白名单：`channels.whatsapp.groups` 设置后即为群组白名单；包含 `"*"` 表示允许所有群组
- 自定义提及模式：`messages.groupChat.mentionPatterns: ["@openclaw"]`






### 7.4 Tool Sandboxing

OpenClaw 以**会话为单位**进行 Docker 沙箱隔离，信任边界与会话类型直接对应：

| 会话类型 | 默认沙箱 | 说明 |
|---|---|---|
| `main`（操作员直接会话） | 无 | 完整主机访问，无 Docker 开销 |
| DM 会话 | 启用 | 即使已审批联系人，默认仍沙箱化，防止误操作或注入 |
| 群组会话 | 启用 | 多参与方输入风险更高，默认沙箱化 |

**沙箱容器提供**：隔离文件系统、可配置网络访问（默认关闭）、CPU/内存资源限制，容器用完即销毁，"爆炸半径"限制在容器内。

**影响安全边界的关键配置项**：

| 配置维度 | 说明 |
|---|---|
| **容器粒度** | 按会话隔离（最强）、按 Agent 隔离、多会话共享（最高效） |
| **主机挂载（Bind Mount）** | 无挂载 / 只读视图 / 读写访问；读写挂载会重新引入风险 |
| **网络访问** | 默认关闭，仅在确有需要时显式开启 |
| **逃逸口（Escape Hatches）** | 任何绕过沙箱的"主机级"工具均视为高信任面，需严格管控 |

**工具策略优先级**（后者覆盖前者，权限只能收紧不能放宽）：

```
Tool Profile → Provider Profile → Global Policy → Provider Policy
    → Agent Policy → Group Policy → Sandbox Policy
```

### 7.5 Prompt Injection Defense

**上下文隔离**：
- 用户消息携带来源元数据，系统指令与用户内容严格分离
- 工具结果使用结构化格式包装，防止被误解为系统指令

**模型选择**：推荐使用最新旗舰模型（如 Claude Opus 4.5）以获得更强的指令遵循能力。若因成本/延迟使用小模型，须通过**缩小爆炸半径**来补偿：偏好只读工具、最小化文件系统暴露、强制沙箱化、收紧白名单。

**操作层面最佳实践**：
- 通过配对/白名单锁定入站 DM
- 群组中优先使用 `requireMention`，避免"全量监听"公共频道
- 对链接、附件、粘贴的指令默认视为敌意输入
- 不可信频道广泛启用沙箱，禁用网络类工具（`web_search` / `web_fetch` / `browser`）
- 将 Secrets 保持在 Agent 可访问文件系统之外

> 沙箱和系统提示护栏属于软防护，真正的执行强制来自：频道访问控制 + 工具策略限制 + 沙箱容器隔离 + 必要时的显式执行审批。

---







## 8. Deployment Architecture

OpenClaw 支持四种主要部署模式，架构保持一致，差异在于 Gateway 运行位置和客户端接入方式。

| 部署模式 | 典型场景 | 核心特点 |
|---|---|---|
| **本地开发（macOS/Linux）** | 开发调试 | `pnpm dev` 前台运行，热重载，Gateway 绑定 `127.0.0.1:18789`，loopback 可信无需认证，全量 debug 日志 |
| **生产 macOS（菜单栏应用）** | 个人日常使用 | LaunchAgent 后台服务 + 登录自启动，原生菜单栏管理 Gateway 生命周期，内嵌 WebChat、Voice Wake，支持 iMessage；可通过 SSH 隧道或 Tailscale 远程访问 |
| **Linux/VPS（远程网关）** | 7×24 可用 | systemd 服务常驻，Gateway 仍绑定 loopback 保证安全，通过以下两种方式接入本地客户端 |
| **Fly.io（容器部署）** | 云原生 | Docker 容器 + 持久化 Volume 存储状态（配置/Session/凭证），Fly.io 托管 HTTPS + TLS 终止，面向公网 **必须开启强认证** |




### 8.1 Linux/VPS 两种接入方案

**Option A：SSH 隧道（推荐默认）**
<div align="center">
<img src="https://substackcdn.com/image/fetch/$s_!gEaj!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ff6598bd3-6b15-4880-8688-4bf6dcc6227c_367x609.png" width="300px" />
</div>

将本地端口转发到远端 loopback，本地 CLI 和 Web UI 透明地通过加密隧道访问远端 Gateway：

```bash
ssh -N -L 18789:127.0.0.1:18789 user@vps
```

---


**Option B：Tailscale Serve（仅限 tailnet 的 HTTPS）**
<div align="center">
<img src="https://substackcdn.com/image/fetch/$s_!CaLD!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F33a4546b-a7b0-45df-8b0b-b8047e25deb2_507x447.png" width="400px" />
</div>

VPS 和客户端设备同加入同一 Tailscale 网络（tailnet），VPS 通过 `tailscale serve` 将 Gateway 以 HTTPS 暴露给 tailnet 内设备，无需维护 SSH 隧道进程：

```json
{ "gateway": { "tailscale": { "mode": "serve" } } }
```



### 8.2 各模式安全要点对比

| 部署模式 | 网络暴露面 | 认证要求 |
|---|---|---|
| 本地开发 | loopback only | 无需认证 |
| 生产 macOS | loopback + 可选远程 | 远程需 Token 或 Password |
| Linux/VPS | loopback（SSH/Tailscale 隧道穿透） | SSH Key 或 Tailscale 设备信任 |
| Fly.io | 公网 HTTPS | **强制开启认证**（Token / Password） |

---








## 9. Plugin Extension System

OpenClaw 的设计目标是**无需修改核心代码**即可扩展。插件系统支持四个扩展维度：

### 9.1 四类插件

| 插件类型 | 作用 | 典型示例 |
|---|---|---|
| **Channel 插件** | 新增消息平台适配器 | Microsoft Teams、Matrix、Mattermost 等 |
| **Memory 插件** | 替换记忆存储后端 | 向量数据库、知识图谱（替代默认 SQLite） |
| **Tool 插件** | 扩展 Agent 可调用的工具能力 | 超出内置 bash、浏览器、文件操作之外的自定义工具 |
| **Provider 插件** | 接入自定义 LLM 或本地模型 | 自托管模型、私有推理服务 |

### 9.2 插件发现与加载机制

插件系统的核心实现位于 `src/plugins/loader.ts`，采用**基于发现的模型（Discovery-based Model）**，整体流程如下：

```
1. 扫描工作区包（workspace packages）
        ↓
2. 检查各包的 package.json 中是否存在 openclaw.extensions 字段
        ↓
3. 对声明的插件进行 Schema 验证（TypeBox 定义生成）
        ↓
4. 配置存在时热加载（Hot-load）插件
```

所有插件代码统一存放在 `extensions/` 目录下，每个插件通过在自己的 `package.json` 中声明 `openclaw.extensions` 字段来注册自身。插件工具通过 `api.registerTool(toolName, toolDefinition)` 注册到 Agent Runtime，注册后会自动出现在工具定义列表中，无需修改任何核心源码。

### 9.3 插件与核心功能的边界

- 插件**只扩展**，不替换核心 Gateway 和 Agent Runtime 的行为
- Channel 插件实现与内置适配器相同的统一接口，对 Gateway 透明
- Memory 插件替换存储后端，上层记忆检索 API 保持不变
- Tool 插件注册后，Agent 在 Execution Loop 中可以像调用内置工具一样调用它们
- Provider 插件扩展模型提供商列表，Session 配置时按名称引用即可

---







## 10. Conclusion

OpenClaw 代表了一种**本地优先、自托管、完全可控**的个人 AI 基础设施范式：

- **统一入口**：通过 Hub-and-Spoke 架构，单一 Gateway 统管所有消息平台
- **真正的持久化 Agent**：不是对话型聊天机器人，而是有状态、有记忆、能执行工具的智能体
- **安全纵深防御**：网络隔离 + 设备配对 + 白名单访问控制 + Docker 沙箱 + Prompt 注入防御
- **高度可扩展**：插件系统支持新增频道、记忆后端、工具和模型提供商

> 在 AI 能力越来越被封闭在专有 API 和围墙花园中的时代，OpenClaw 提供了一种替代方案：在你自己的基础设施上运行助手，通过你已经在用的频道访问它，同时对其工作原理保持完全透明。















