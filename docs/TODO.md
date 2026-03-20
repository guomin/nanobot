# 未深入模块 Todo 列表

## 待深入模块

### 高优先级

| 模块 | 路径 | 说明 |
|------|------|------|
| Memory | `agent/memory.py` | 记忆系统（MEMORY.md / HISTORY.md 归档） |
| Skills | `agent/skills.py` | 技能加载（SKILL.md） |
| Subagent | `agent/subagent.py` | 子 Agent 管理 |

### 中优先级

| 模块 | 路径 | 说明 |
|------|------|------|
| Cron | `cron/` | 定时任务 |
| Heartbeat | `heartbeat/` | 心跳/健康检查 |
| Config | `config/` | 配置管理 |
| CLI | `cli/` | 命令行入口 |

---

## 已详细梳理的模块

- ✅ Agent Loop（双层循环）
- ✅ Message Bus（消息总线）
- ✅ Session（会话管理）
- ✅ Tools（工具系统）
- ✅ LLM Provider（LiteLLM）
- ✅ Channel/Manager（通道管理）
- ✅ 消息类型与传递
- ✅ 工具传递机制
- ✅ 内外层协同