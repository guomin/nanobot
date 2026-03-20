# Tool 系统详解

> nanobot 工具（Tool）机制

## 内置工具

| 工具名 | 类 | 功能 |
|--------|-----|------|
| `read_file` | ReadFileTool | 读取文件，支持分页 |
| `write_file` | WriteFileTool | 写入文件 |
| `edit_file` | EditFileTool | 编辑文件（diff 模式） |
| `list_dir` | ListDirTool | 列出目录 |
| `exec` | ExecTool | 执行 Shell 命令 |
| `web_search` | WebSearchTool | 网页搜索（ddgs） |
| `web_fetch` | WebFetchTool | 获取网页内容 |
| `message` | MessageTool | 跨频道发消息 |
| `spawn` | SpawnTool | 启动子 Agent |
| `cron` | CronTool | 定时任务（可选） |

## Tool 基类

```python
class Tool(ABC):
    name: str          # 工具名（唯一标识）
    description: str   # 描述（给 LLM 看）
    parameters: dict   # JSON Schema 参数定义
    
    async def execute(**kwargs) -> str:
        # 执行逻辑，返回字符串结果
```

## Tool → Schema 转换

```python
def to_schema(self) -> dict:
    return {
        "type": "function",
        "function": {
            "name": self.name,
            "description": self.description,
            "parameters": self.parameters,
        },
    }
```

## 工具注册与调用

**注册**（AgentLoop._register_default_tools）：
```python
self.tools.register(ReadFileTool(...))
self.tools.register(ExecTool(...))
```

**获取定义**（传给 LLM）：
```python
tool_defs = self.tools.get_definitions()
# 返回 [tool.to_schema() for tool in self._tools.values()]
```

**执行**：
```python
result = await self.tools.execute(tool_name, arguments)
```

## 传给 LLM 的格式

```python
{
    "tools": [
        {
            "type": "function",
            "function": {
                "name": "read_file",
                "description": "Read the contents of a file...",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "path": {"type": "string", "description": "..."}
                    },
                    "required": ["path"]
                }
            }
        },
        ...
    ]
}
```

## 工具实现方式

| 工具类别 | 实现方式 |
|----------|----------|
| **文件系统** | `pathlib.Path` 读写，`_resolve_path()` 权限检查 |
| **Shell** | `asyncio.create_subprocess_shell()`，超时控制，危险命令黑名单 |
| **网页** | `httpx` 请求，`ddgs` 搜索，HTML 清洗 |
| **消息** | 调用 `bus.publish_outbound()` 直接发消息 |
| **子 Agent** | 创建新 `InboundMessage` 发到 bus |

## 工具执行流程

```
LLM 返回 tool_calls
    │
    ├─> name: 工具名
    ├─> arguments: 参数
    │
    ▼
tools.execute(name, arguments)
    │
    ├─> 参数校验（validate_params）
    ├─> 类型转换（cast_params）
    ├─> 执行（execute）
    └─> 返回结果字符串
    │
    ▼
结果加入消息历史，继续循环
```

## 安全机制

- **路径限制**：文件操作限制在 workspace 内
- **命令黑名单**：exec 禁止 rm -rf、shutdown 等危险命令
- **URL 验证**：web_fetch 防止 SSRF 攻击
- **参数校验**：JSON Schema 验证输入

## 关键文件

| 文件 | 说明 |
|------|------|
| `agent/tools/base.py` | Tool 基类 |
| `agent/tools/registry.py` | 工具注册表 |
| `agent/tools/filesystem.py` | 文件系统工具 |
| `agent/tools/shell.py` | Shell 执行工具 |
| `agent/tools/web.py` | 网页工具 |
| `agent/tools/message.py` | 消息工具 |
| `agent/tools/spawn.py` | 子 Agent 工具 |
| `agent/tools/cron.py` | 定时任务工具 |

## 设计要点

1. **Schema 驱动**：工具定义自动转为 OpenAI function 格式
2. **参数校验**：内置类型转换和 JSON Schema 验证
3. **安全沙箱**：路径限制、命令黑名单、URL 验证
4. **统一返回**：所有工具返回字符串，方便 LLM 处理
5. **可扩展**：通过 ToolRegistry 可动态注册新工具