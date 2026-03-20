# Message Bus 消息总线详解

> nanobot 消息传递的核心基础设施

## 核心结构

```python
class MessageBus:
    inbound: asyncio.Queue[InboundMessage]   # 用户 → Agent
    outbound: asyncio.Queue[OutboundMessage] # Agent → 用户
```

## 消息类型

### InboundMessage

| 字段 | 类型 | 说明 |
|------|------|------|
| channel | str | 来源：telegram, cli, dingtalk, slack 等 |
| sender_id | str | 发送者 ID |
| chat_id | str | 会话 ID |
| content | str | 消息内容 |
| media | list[str] | 附件列表 |
| metadata | dict | 平台特定数据 |
| session_key | str | channel:chat_id，会话隔离 |

### OutboundMessage

| 字段 | 类型 | 说明 |
|------|------|------|
| channel | str | 目标渠道 |
| chat_id | str | 目标会话 |
| content | str | 回复内容 |
| reply_to | str | 回复某条消息 |
| media | list[str] | 附件 |
| metadata | dict | 含 _progress, _tool_hint 等控制标志 |

## 消息流

| 入口 | 发布 | 消费 |
|------|------|------|
| Channel（各平台） | publish_inbound | - |
| CLI | publish_inbound | - |
| Subagent | publish_inbound | - |
| AgentLoop.run() | - | consume_inbound |
| MessageTool（工具） | publish_outbound | - |
| AgentLoop | publish_outbound | - |
| Manager._dispatch_outbound() | - | consume_outbound |
| Channel.send() | - | 发送消息 |
发布端                          消费端
─────────────────────────────────────────────────
Channel (各平台)                AgentLoop.run()
    ↓ publish_inbound              ↓ consume_inbound

CLI                             Manager._dispatch_outbound
    ↓ publish_inbound              ↓ consume_outbound

Subagent                        Channel.send()
    ↓ publish_inbound              (各平台发送)

MessageTool (工具)
    ↓ publish_outbound
```

## 设计要点

### 1. 解耦
Channel 和 Agent 不直接依赖，通过 MessageBus 通信

### 2. 异步
生产/消费分离，消息队列缓冲峰值

### 3. 统一接口
多平台统一消息格式（Inbound/Outbound）

### 4. 会话隔离
session_key = channel:chat_id，不同渠道/用户消息隔离

### 5. 进度推送
metadata 含 `_progress=True` 标识实时进度

### 6. 工具提示
metadata 含 `_tool_hint=True` 标识工具调用提示

## 关键文件

| 文件 | 说明 |
|------|------|
| `bus/queue.py` | 消息队列实现 |
| `bus/events.py` | 消息类型定义 |

## 全局唯一

运行时只有一个 MessageBus 实例，被 AgentLoop、ChannelManager、CLI 共享。