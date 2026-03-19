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

**核心循环逻辑**：

```python
while iteration < max_iterations:
    1. 构建消息 (system + history + current_message)
    2. 调用 LLM (chat_with_retry)
    3. 检查响应:
       - 有 tool_calls → 执行工具 → 把结果加回消息 → 继续循环
       - 无 tool_calls → 返回最终回复 → 结束
```

**关键特性**：
- **ReAct 模式**：思考 → 行动 → 观察，循环直到完成任务
- **进度流式**：通过 `on_progress` 回调实时显示 thinking 过程
- **并发处理**：每个 session 的消息作为独立 task 处理
- **命令支持**：`/stop` 停止任务、`/restart` 重启、`/new` 新会话

---

### 2. Context Builder（上下文构建）— `agent/context.py`

**构建发送给 LLM 的完整消息**：

```
System Prompt = 身份 + Bootstrap文件 + 记忆 + 技能 + 工具清单
                ↓
完整消息 = System + History + CurrentMessage + RuntimeContext
```

**Bootstrap 文件**（按优先级加载）：
- `AGENTS.md` — Agent 行为指南
- `SOUL.md` — 核心价值观
- `USER.md` — 用户信息
- `TOOLS.md` — 工具使用说明

**Runtime Context**（每次注入）：
- 当前时间、所在频道、Chat ID

---

### 3. Memory System（记忆系统）— `agent/memory.py`

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

**事件类型**：
```python
@dataclass
class InboundMessage:
    channel: str       # 来源渠道
    sender_id: str     # 发送者 ID
    chat_id: str       # 会话 ID
    content: str       # 消息内容
    media: list        # 附件列表
    metadata: dict     # 渠道特定数据

@dataclass
class OutboundMessage:
    channel: str
    chat_id: str
    content: str
    reply_to: str | None
```

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
