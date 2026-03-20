# nanobot 工程精髓笔记

> 义父，这是我对 nanobot 的深度理解，帮你迅速抓住这个工程的精髓。

## 核心目标

**让 AI Agent 变得更轻量、更易用** — 99% 更少的代码实现同样的 Agent 能力。

---

## 架构概览：一句话总结

```
用户消息 → Channel(接收) → Bus(路由) → Agent(大脑) → Tool(执行) → LLM(思考) → Channel(回复)
```

nanobot 本质上是一个 **消息驱动的对话 Agent 框架**，核心流程是：
1. 接收用户消息
2. 构建上下文（历史 + 技能 + 记忆）
3. 调用 LLM + 工具执行循环
4. 返回结果

---

## 核心模块解析

### 1. Agent Loop（大脑）— `agent/loop.py`

#### 双层循环架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    外层循环: run() — 消费消息                          │
│                                                                          │
│  while self._running:                                                   │
│      msg = await bus.consume_inbound()  # 持续监听消息                  │
│          │                                                              │
│          └─> _dispatch(msg)                                            │
│                  │                                                      │
│                  └─> _process_message()                                │
│                          │                                              │
│                          └─> 内层循环: _run_agent_loop()                │
│                                  (LLM + 工具循环)                       │
└─────────────────────────────────────────────────────────────────────────┘
```

**外层循环**（256-278行）：事件驱动，持续消费消息
- 从消息总线获取消息
- 解析命令（/stop、/restart、/new、/help）
- 派发到 `_process_message()` 异步处理

**内层循环**（194行）：ReAct 循环，处理单次对话
- 最多迭代 40 次
- 调用 LLM → 执行工具 → 继续或退出

**全局唯一**：运行时只有一个 AgentLoop 实例，通过 `cli/commands.py` 的两个入口之一创建：
- `nanobot gateway` → gateway 命令，启动多渠道网关
- `nanobot agent` → agent 命令，启动 CLI 交互
两个入口不同时运行，共用同一个 MessageBus。

#### 设计洞见

**内外层对应的本质**：

| | 外层 | 内层 |
|---|---|---|
| **抽象** | 调度者（Dispatcher） | 工作者（Worker） |
| **维度** | 时间（持续运行） | 空间（单次任务） |
| **本质** | 存在（活着） | 思考（想清楚） |

**一句话**：
- 外层让系统**活着**（持续接收消息）
- 内层让 LLM**想清楚**（多轮工具调用）

**本质**：内层的循环是 LLM 本身决定的，外层的循环是系统架构选择的。

#### 关键设计实现

**外层设计**：
- 非阻塞等待（timeout=1s）
- 异步任务派发（create_task）
- 会话级任务管理（_active_tasks）
- 全局处理锁（_processing_lock）
- 后台任务（_background_tasks）
- 命令优先处理（/stop、/restart、/new）

**内层设计**：
- ReAct 循环（while < 40 次迭代）
- 消息累积（add_assistant_message + add_tool_result）
- 工具定义传递（get_definitions → tools 参数）
- 推理内容保留（reasoning_content / thinking_blocks）
- 进度回调（on_progress）
- 重试机制（chat_with_retry）

**协同方式**：
- 消息构建传递：外层 → 内层
- 进度回调：内层 → 外层 → 实时推送
- 状态持久化：内层 → 外层 → sessions.save()
- 控制命令：外层拦截 → 取消内层任务
- 工具跨层：MessageTool 直接调用 bus

#### 消息类型与传递

**消息总线级别**（bus/events.py）：

| 类型 | 方向 | 说明 |
|------|------|------|
| **InboundMessage** | Channel → Agent | 用户发给 bot |
| **OutboundMessage** | Agent → Channel | bot 回复用户 |

**LLM 消息级别**（消息历史）：

| role | 说明 |
|------|------|
| **system** | 系统提示（身份、Bootstrap、记忆、技能） |
| **user** | 用户消息 |
| **assistant** | LLM 回复（可能含 tool_calls、reasoning_content、thinking_blocks） |
| **tool** | 工具执行结果 |

**三类输入消息**（按性质）：

| 类型 | 性质 | 说明 |
|------|------|------|
| **System Prompt** | 状态变化反映 | 每次重新构建，包含背景和能力 |
| **Current Message** | 动作直接驱动 | 本轮输入 + Runtime Context |
| **History** | 相对被动 | 累积上下文，可能过长触发压缩 |

**消息分层**：
```
外部输入层：System + History + Current
    ↓
内生反馈层：Tool Result + Reasoning
    ↓
输入 LLM → 产生新内生 → 循环
```

**内生消息**（LLM 多轮调用产生）：

| 类型 | 位置 | 说明 |
|------|------|------|
| **Tool Result** | role="tool" 消息 | 执行结果反馈给 LLM |
| **Reasoning** | assistant 消息中 | reasoning_content / thinking_blocks，保留思考过程 |

**工具传递**（洞见）：
- 独立于 System Prompt（不依赖 prompt 动态更新）
- 静态传递：工具定义只在循环开始获取一次
- 动态累积：工具调用结果加入消息历史
- 传递方式：通过 `tools` 参数传给 LLM（schema 格式）

LLM 返回 tool_calls → 框架执行 → 结果加到消息历史 → 继续循环

#### 内层循环：LLM + Tools

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        while iteration < 40                            │
│                              ↓                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 第1步: 获取工具定义 (197)                                        │   │
│  │   tool_defs = self.tools.get_definitions()                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 第2步: 调用 LLM (199-203)                                        │   │
│  │   response = await provider.chat_with_retry(                   │   │
│  │       messages=messages,                                        │   │
│  │       tools=tool_defs,                                          │   │
│  │       model=self.model,                                         │   │
│  │   )                                                             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 第3步: 判断返回值 (205)                                          │   │
│  │   if response.has_tool_calls:  → 有工具调用                     │   │
│  │   else: → 无工具调用(最终回答)，直接 break                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│           ↓                           ↓                               │
│      【有工具调用】                  【无工具调用】                    │
│           ↓                           ↓                               │
│  ┌──────────────┐              ┌──────────────────┐                  │
│  │ A. 发送进度   │              │ B. 清理思考内容  │                  │
│  │    (206-212) │              │    (233)        │                  │
│  └──────────────┘              └──────────────────┘                  │
│  ┌──────────────┐              ┌──────────────────┐                  │
│  │ C. 记录助手   │              │ C. 检查错误     │                  │
│  │    消息+工具  │              │    (236-239)   │                  │
│  │    (214-222) │              └──────────────────┘                  │
│  └──────────────┘                      ↓                              │
│  ┌──────────────┐              ┌──────────────────┐                  │
│  │ D. 逐个执行   │              │ D. 记录助手消息  │                  │
│  │    工具并记录 │              │    (240-243)   │                  │
│  │    (224-231) │              └──────────────────┘                  │
│  └──────────────┘                      ↓                              │
│       ↓                                 ↓                              │
│       └─────── 回到第1步，继续循环 ─────┘                              │
│                                 ↓                                       │
│                          ┌──────────────┐                              │
│                          │ E. 返回结果   │                              │
│                          │    (244)     │                              │
│                          └──────────────┘                              │
└─────────────────────────────────────────────────────────────────────────┘
```

**循环步骤详解**：

| 步骤 | 代码行 | 说明 |
|------|--------|------|
| **初始化** | 189-192 | `messages` 初始消息列表，`iteration=0` 计数器 |
| **循环条件** | 194 | 最多 40 次迭代 |
| **获取工具** | 197 | 从 `ToolRegistry` 获取工具定义（JSON 格式） |
| **调用 LLM** | 199-203 | 发送消息+工具给 LLM，返回 `response` |
| **有工具调用** | 205-231 | LLM 返回了工具调用，需执行后继续循环 |
| └ 发送进度 | 206-212 | 发送思考内容和工具提示给用户 |
| └ 记录消息 | 218-222 | 把助手回复+工具调用加入消息历史 |
| └ 执行工具 | 224-231 | 逐个执行工具，把结果加入消息历史 |
| **无工具调用** | 232-245 | LLM 返回最终回答，退出循环 |
| └ 清理思考 | 233 | 移除 `` 块 |
| └ 检查错误 | 236-239 | 如果 `finish_reason="error"`，返回错误信息 |
| └ 记录消息 | 240-243 | 把最终回答加入消息历史 |
| └ 设置结果 | 244 | `final_content = clean`，退出循环 |
| **超时处理** | 247-252 | 超过 40 次迭代，返回提示信息 |
| **返回** | 254 | `(final_content, tools_used, messages)` |

**消息流转示例**：

```
第1次迭代:
  输入: [用户消息]
  LLM返回: 工具调用 "read_file"
  → 执行工具，读文件结果
  → 消息变为: [用户消息, 助手(工具调用), 工具结果]

第2次迭代:
  输入: [用户消息, 助手(工具调用), 工具结果]
  LLM返回: "文件内容是..."
  → 无工具调用，退出
  → final_content = "文件内容是..."
```

#### 消息历史构建

在调用 LLM 前，通过 `ContextBuilder.build_messages()` 构建完整消息：

```
┌──────────────────────────────────────────────────────────────────┐
│                    build_messages()                              │
│                                                                  │
│  ┌────────────────┐                                             │
│  │ System Prompt  │ ← build_system_prompt()                     │
│  │ (role: system) │    - 身份定义                                │
│  └────────────────┘    - Bootstrap 文件 (AGENTS.md/SOUL.md等)  │
│         │              - 记忆 (MEMORY.md)                        │
│         │              - 技能 (Skills)                           │
│         ▼              - 工具清单                                 │
│  ┌────────────────┐                                             │
│  │   History     │ ← session.get_history()                      │
│  │ (role: user/  │    - 历史消息列表                             │
│  │  assistant)   │    - 已整合的消息                             │
│  └────────────────┘                                             │
│         │                                                        │
│         ▼                                                        │
│  ┌────────────────┐                                             │
│  │ Current Message│ ← 当前用户消息 + Runtime Context            │
│  │ (role: user)  │    - 当前时间                                 │
│  └────────────────┘    - 频道/Chat ID                          │
│                         - 用户输入 + 附件(图片)                   │
└──────────────────────────────────────────────────────────────────┘
```

**最终消息格式**：

```python
[
    {"role": "system", "content": "你是 nanobot..."},
    {"role": "user", "content": "上一条消息..."},
    {"role": "assistant", "content": "回复..."},
    {"role": "tool", "tool_call_id": "xxx", "name": "read_file", "content": "文件内容..."},
    {"role": "user", "content": "[Runtime Context]\n当前时间: ...\n\n用户的新消息"},
]
```

**关键特性**：
- **ReAct 模式**：思考 → 行动 → 观察，循环直到完成任务
- **进度流式**：通过 `on_progress` 回调实时显示 thinking 过程
- **并发处理**：每个 session 的消息作为独立 task 处理
- **命令支持**：`/stop` 停止任务、`/restart` 重启、`/new` 新会话

---

### 2. Memory System（记忆系统）— `agent/memory.py`

**两层记忆架构**：

```
┌─────────────────────────────────────────────────┐
│  会话消息 (JSONL)                                │
│  - session.messages[] → 发送给 LLM 的历史        │
│  - session.last_consolidated → 已归档的位置      │
└─────────────────────────────────────────────────┘
           ↓ 定期归档
┌─────────────────────────────────────────────────┐
│  长期记忆层                                      │
│  ├─ MEMORY.md  → LLM 总结的事实 (LLM可读)       │
│  └─ HISTORY.md → 可 grep 搜索的历史日志          │
└─────────────────────────────────────────────────┘
```

**智能整合策略**：
- Token 超过上下文窗口一半时自动触发
- 找到用户消息边界进行截断
- 调用 LLM 总结并保存到文件
- 失败3次后降级为原始归档

---

### 4. Tool System（工具系统）

**架构**：

```
ToolRegistry ←── 持有所有已注册工具
     │
     ├── ShellTool       → 执行命令
     ├── FileSystemTool  → 读写编辑文件
     ├── WebSearchTool   → 网页搜索
     ├── WebFetchTool    → 获取网页内容
     ├── CronTool        → 定时任务
     ├── MessageTool     → 跨频道发消息
     ├── SpawnTool       → 启动子 Agent
     ├── MCPTool         → MCP 协议工具
     └── ...更多
```

**Tool 基类**（`agent/tools/base.py`）：
```python
class Tool(ABC):
    name: str          # 工具名
    description: str   # 描述（给 LLM 看）
    parameters: dict  # JSON Schema 参数定义
    
    async def execute(**kwargs) -> str:
        # 执行逻辑
```

**参数验证**：内置 schema 校验 + 类型转换（字符串→整数等）

---

### 5. Channel System（渠道接入）

**设计模式**：每个平台是一个 Channel，遵循 `BaseChannel` 接口

```
BaseChannel (抽象基类)
     │
     ├── TelegramChannel
     ├── DiscordChannel
     ├── FeishuChannel
     ├── SlackChannel
     ├── DingTalkChannel
     ├── QQChannel
     ├── WhatsAppChannel
     ├── EmailChannel
     ├── MatrixChannel
     └── WecomChannel
```

**消息流向**：
```
Channel → _handle_message() → 权限检查 → InboundMessage → Bus.publish_inbound()
                                                              ↓
Bus.consume_inbound() → AgentLoop._dispatch() → Agent.process_message()
                                                              ↓
Agent 回复 → OutboundMessage → Bus.publish_outbound() → Channel.send()
```

---

### 6. Message Bus（消息总线）— `bus/`

详细内容见 [MESSAGE_BUS.md](./MESSAGE_BUS.md)

**Session Key**：由 `channel:chat_id` 构成，确保不同渠道的消息隔离

---

### 7. Session Manager（会话管理）— `session/manager.py`

**会话存储**：JSONL 格式
```
{metadata}
{message1}
{message2}
...
```

**关键字段**：
- `last_consolidated`：标记哪些消息已归档到文件
- `messages[]`：待发送给 LLM 的消息列表

**历史获取逻辑**：
1. 从 `last_consolidated` 位置开始取最新消息
2. 截取 max_messages 条
3. 丢弃开头非 user 消息（避免半截对话）
4. 对齐 tool_call/tool_result 配对

---

### 8. Provider System（模型接入）— `providers/`

**LiteLLM 统一封装**：

```python
# 通过 LiteLLM 支持 100+ 模型
# 只需配置 API key 和 model name
providers = {
    "openrouter": {...},   # OpenRouter 网关
    "anthropic": {...},    # Claude
    "openai": {...},       # GPT
    "deepseek": {...},     # DeepSeek
    "gemini": {...},       # Gemini
    "minimax": {...},     # MiniMax
    "ollama": {...},      # 本地模型
    "vllm": {...},        # 本地 vLLM
    # ...更多
}
```

**Provider Registry**：通过 `ProviderSpec` 配置新增 provider，无需改代码

---

### 9. Skills System（技能系统）— `agent/skills.py`

**技能本质**：Markdown 文件（SKILL.md），教导 Agent 如何使用工具

**技能来源**：
1. **内置技能**：`nanobot/skills/` 目录
2. **工作区技能**：`workspace/skills/` 目录
3. **ClawHub**：从网络下载社区技能

**渐进加载**：
- Always Skills → 每次都加载到 System Prompt
- 其他 Skills → 仅当 Agent 主动读取时加载

---

## 工作流程图

```
┌──────────────────────────────────────────────────────────────────┐
│                         nanobot Gateway                          │
│                                                                  │
│  ┌─────────┐    ┌──────┐    ┌───────────────────────────────┐  │
│  │Channel  │───▶│ Bus  │───▶│        Agent Loop              │  │
│  │(多渠道) │    │      │    │                                │  │
│  └─────────┘    └──────┘    │  ┌─────────────────────────┐   │  │
│                             │  │   Context Builder       │   │  │
│                             │  │   ├─ System Prompt     │   │  │
│                             │  │   ├─ Memory            │   │  │
│                             │  │   ├─ Skills            │   │  │
│                             │  │   └─ History           │   │  │
│                             │  └─────────────────────────┘   │  │
│                             │            │                    │  │
│                             │            ▼                    │  │
│                             │  ┌─────────────────────────┐   │  │
│                             │  │   LLM + Tools Loop      │   │  │
│                             │  │   (ReAct Cycle)         │   │  │
│                             │  │                         │   │  │
│                             │  │   think → tool → result │   │  │
│                             │  │   until done            │   │  │
│                             │  └─────────────────────────┘   │  │
│                             └───────────────────────────────┘  │
│                                          │                      │
│                                          ▼                      │
│                             ┌──────────────────────────────┐   │
│                             │         Memory System         │   │
│                             │  ├─ Session (JSONL)          │   │
│                             │  ├─ MEMORY.md (事实)        │   │
│                             │  └─ HISTORY.md (日志)        │   │
│                             └──────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 核心设计哲学

### 1. 极简主义
- 核心 Agent Loop 仅 ~500 行
- 每个模块职责单一
- 通过组合而非继承实现扩展

### 2. 可插拔架构
- Channel 通过插件机制接入
- Provider 通过 Registry 注册
- Tool 通过动态注册表管理

### 3. 容错设计
- 工具执行失败 → 返回错误提示 + 建议
- LLM 调用失败 → 重试 + 优雅降级
- 记忆整合失败 → 原始归档降级

### 4. 安全优先
- `allowFrom` 白名单机制
- `restrictToWorkspace` 沙箱限制
- Shell 命令路径限制
- 工具参数 schema 校验

---

## 关键文件速查

| 功能 | 文件路径 | 行数 | 说明 |
|------|----------|------|------|
| Agent 主循环 | `agent/loop.py` | ~510 | 核心 ReAct 逻辑 |
| 上下文构建 | `agent/context.py` | ~195 | System Prompt 构建 |
| 记忆系统 | `agent/memory.py` | ~357 | 两层记忆架构 |
| 会话管理 | `session/manager.py` | ~242 | 消息持久化 |
| 工具注册 | `agent/tools/registry.py` | ~70 | 工具动态管理 |
| 消息总线 | `bus/events.py` | ~38 | 事件类型定义 |
| 渠道基类 | `channels/base.py` | ~139 | Channel 接口 |
| LLM 提供者 | `providers/litellm_provider.py` | ~355 | LiteLLM 封装 |
| 技能加载 | `agent/skills.py` | ~228 | 技能管理 |
| CLI 入口 | `cli/commands.py` | — | 命令行接口 |

---

## 命令行使用

```bash
nanobot onboard           # 初始化配置和工作区
nanobot agent -m "你好"   # 单次对话
nanobot agent             # 交互式对话
nanobot gateway           # 启动网关（监听各渠道消息）
nanobot channels login    # WhatsApp 等需要扫码登录
nanobot status            # 查看状态
```

---

## 配置示例

```json
{
  "providers": {
    "openrouter": {
      "apiKey": "sk-or-xxx"
    }
  },
  "agents": {
    "defaults": {
      "model": "anthropic/claude-sonnet-4-5",
      "provider": "openrouter"
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "token": "xxx",
      "allowFrom": ["USER_ID"]
    }
  }
}
```

---

## 总结

**nanobot 的精髓**：

1. **Message Bus 架构**：解耦消息接收和处理
2. **ReAct Loop**：简单的"思考-行动-观察"循环
3. **Tool Registry**：工具的动态注册和执行
4. **两层记忆**：会话消息 + 持久化事实
5. **渐进上下文**：按需构建 System Prompt
6. **多渠道支持**：统一的消息格式，灵活的接入方式

这是一份**干净、可扩展、易理解**的 Agent 框架实现，适合学习和二次开发。
