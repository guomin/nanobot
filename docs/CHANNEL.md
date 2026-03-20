# Channel 系统详解

> nanobot 消息通道（Channel）与管理器

## 核心结构

```python
class BaseChannel(ABC):
    name: str              # 渠道名（telegram, dingtalk...）
    display_name: str      # 显示名
    config: Any            # 渠道配置
    bus: MessageBus        # 消息总线
    
    async def start()      # 启动监听
    async def stop()       # 停止
    async def send()       # 发送消息
```

## 支持的 Channel

| Channel | 文件 | 说明 |
|---------|------|------|
| Telegram | telegram.py | 使用 python-telegram-bot |
| DingTalk | dingtalk.py | 钉钉消息通道 |
| Discord | discord.py | Discord 机器人 |
| Slack | slack.py | Slack 集成 |
| Feishu | feishu.py | 飞书消息 |
| WhatsApp | whatsapp.py | WhatsApp Business |
| QQ | qq.py | QQ 机器人 |
| Matrix | matrix.py | Matrix 协议 |
| Email | email.py | 邮件收发 |
| MoChat | mochat.py | 企业微信 MoChat |
| WeCom | wecom.py | 企业微信 |

## Channel 生命周期

```
启动 → start() 监听消息
       ↓
    _on_message() 收到消息
       ↓
    _handle_message() 权限检查
       ↓
    bus.publish_inbound() 发送到总线
```

## ChannelManager 职责

| 功能 | 说明 |
|------|------|
| **初始化** | 扫描并创建已启用的 Channel |
| **启动** | 启动所有 Channel + 出站分发器 |
| **停止** | 停止所有 Channel + 分发器 |
| **分发出站** | 从 bus.consume_outbound() 消费消息，发送到对应 Channel |

### 启动流程

```python
async def start_all(self):
    # 启动出站分发器
    self._dispatch_task = asyncio.create_task(self._dispatch_outbound())
    
    # 启动所有 Channel
    tasks = []
    for name, channel in self.channels.items():
        tasks.append(asyncio.create_task(self._start_channel(name, channel)))
    
    await asyncio.gather(*tasks, return_exceptions=True)
```

### 出站分发逻辑

```python
async def _dispatch_outbound():
    while True:
        msg = await bus.consume_outbound()
        
        # 过滤进度消息（可选配置）
        if msg.metadata.get("_progress"):
            if tool_hint and not send_tool_hints: continue
            if not tool_hint and not send_progress: continue
        
        # 找到对应 Channel 并发送
        channel = self.channels.get(msg.channel)
        await channel.send(msg)
```

## Telegram Channel 实现细节

### 启动方式
- **Long Polling**（长轮询）而非 Webhook
- 连接池大小 16，避免长时运行超时

### 支持的消息类型
- TEXT（文本）
- PHOTO（图片）
- VOICE（语音）
- AUDIO（音频）
- DOCUMENT（文档）

### 命令处理
- /start - 欢迎消息
- /new - 新会话
- /stop - 停止任务
- /restart - 重启
- /help - 帮助

### Markdown 转 HTML
自动将 Markdown 转为 Telegram HTML 格式：
- 代码块、表格转换
- 加粗、斜体转换
- 超长消息分片（4000 字符限制）

## 权限控制

```python
def is_allowed(sender_id: str) -> bool:
    allow_list = config.allow_from
    if not allow_list: return False  # 空列表拒绝所有
    if "*" in allow_list: return True  # * 表示允许所有人
    return sender_id in allow_list
```

## 消息流全景

```
┌─────────────────────────────────────────────────────────────┐
│                      用户（各平台）                          │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    Channel（Telegram 等）                    │
│  start() 启动监听 → 收到消息 → 权限检查 → _handle_message()  │
└────────────────────────────┬────────────────────────────────┘
                             │ publish_inbound()
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      MessageBus                             │
│               (inbound Queue)                               │
└────────────────────────────┬────────────────────────────────┘
                             │ consume_inbound()
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    AgentLoop（处理）                         │
└────────────────────────────┬────────────────────────────────┘
                             │ publish_outbound()
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      MessageBus                             │
│              (outbound Queue)                               │
└────────────────────────────┬────────────────────────────────┘
                             │ consume_outbound()
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                ChannelManager._dispatch_outbound()          │
└────────────────────────────┬────────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         Telegram       DingTalk        Slack
              │              │              │
              ▼              ▼              ▼
         send()           send()         send()
              │              │              │
              └──────────────┴──────────────┘
                             │
                             ▼
                        用户（各平台）
```

## 关键文件

| 文件 | 说明 |
|------|------|
| `channels/base.py` | BaseChannel 基类 |
| `channels/manager.py` | ChannelManager 管理器 |
| `channels/telegram.py` | Telegram 实现 |
| `channels/registry.py` | Channel 注册发现 |

## 设计要点

1. **统一接口**：所有 Channel 继承 BaseChannel
2. **消息解耦**：Channel 不直接调用 Agent，通过 Bus 通信
3. **权限控制**：基于 allow_from 列表
4. **进度过滤**：可配置是否发送推理进度
5. **异步架构**：基于 asyncio，非阻塞