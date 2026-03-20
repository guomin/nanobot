# Session 管理详解

> nanobot 会话管理机制

## Session 数据结构

```python
@dataclass
class Session:
    key: str                    # channel:chat_id，会话唯一标识
    messages: list[dict]         # 消息列表
    created_at: datetime         # 创建时间
    updated_at: datetime        # 更新时间
    last_consolidated: int      # 已归档位置（标记）
```

## 存储方式

- **格式**：JSONL（每行一个 JSON）
- **路径**：`workspace/sessions/{key}.jsonl`
- **结构**：
  ```
  {"_type": "metadata", "key": "telegram:123", "created_at": "...", "last_consolidated": 0}
  {"role": "user", "content": "你好", "timestamp": "..."}
  {"role": "assistant", "content": "你好", "timestamp": "..."}
  ...
  ```

## 核心操作

### 获取/创建会话

```python
session = sessions.get_or_create(key)  # key = channel:chat_id
# 1. 先查缓存 _cache
# 2. 缓存未命中，从文件加载
# 3. 文件不存在，创建新 Session
```

### 获取历史消息

```python
history = session.get_history(max_messages=500)
```

**逻辑**：
1. 从 `last_consolidated` 位置开始（跳过已归档的）
2. 截取最近 N 条
3. 丢弃开头非 user 消息（避免半截对话）
4. 对齐 tool_call/tool_result 配对（防止 orphan tool result）

### 保存会话

```python
sessions.save(session)
# 写入 JSONL 文件，同时更新内存缓存
```

### 清空会话（/new 命令）

```python
session.clear()      # messages = [], last_consolidated = 0
sessions.save(session)
sessions.invalidate(key)  # 从缓存移除
```

## Session 生命周期

```
消息到达 → get_or_create(key) → 获取或创建 Session
                                    ↓
                            get_history() → 取历史消息
                                    ↓
                            _run_agent_loop() → LLM 循环
                                    ↓
                            _save_turn() → 追加新消息
                                    ↓
                            sessions.save() → 持久化
```

## _save_turn 细节

```python
def _save_turn(session, messages, skip):
    for m in messages[skip:]:
        # 跳过空 assistant 消息
        # 截断过长 tool result（>16000 字符）
        # 清理 Runtime Context 前缀
        session.messages.append(entry)
```

## 缓存机制

```python
class SessionManager:
    _cache: dict[str, Session] = {}  # 内存缓存
    
    def get_or_create(key):
        if key in self._cache:
            return self._cache[key]  # 直接返回
        session = self._load(key)     # 从文件加载
        self._cache[key] = session
        return session
```

## 会话隔离

- `key = channel:chat_id`
- 例如：`telegram:123`、`cli:direct`
- 每个会话独立存储，互不影响

## 关键文件

| 文件 | 说明 |
|------|------|
| `session/manager.py` | Session 和 SessionManager 类 |
| `session/__init__.py` | 模块导出 |

## 设计要点

1. **Append-only**：消息只追加，不修改历史
2. **last_consolidated**：标记已归档位置，支持增量归档
3. **内存缓存**：热点会话在内存，加速访问
4. **工具边界对齐**：确保 tool_call 和 tool_result 配对完整
5. **消息截断**：防止过长 tool result 撑爆上下文