# Hermes Agent 框架

## 一句话定位
Nous Research 开发的自我改进 Agent 框架，通过 Skills 程序性记忆实现闭环学习，内置 RL 训练支持。

## 核心设计决策 (ADR)

- **决策**: Skills 以 Markdown + YAML frontmatter 文件存储，而非数据库记录。
- **背景**: 文件更易 diff/版本控制，Agent 可直接 read/edit，无需 ORM。
- **后果**: 便于 Agent 自主管理 Skill，但多实例并发写入需要原子替换（tmpfile + os.replace）。

- **决策**: 系统提示采用"冻结快照"机制——Session 开始时固定，中途不随记忆写入而变化。
- **背景**: LLM 前缀缓存（Prompt Cache）依赖系统提示的稳定性，频繁变化会导致缓存失效、token 成本飙升。
- **后果**: 会话内记忆更新不会立即生效，需等到下次 Session 启动时才注入。

## 核心组件

| 组件 | 文件 | 职责 |
|---|---|---|
| Agent Loop | `run_agent.py` | 主循环，多轮工具调用 |
| Skill Manager | `skill_manager_tool.py` | Skill CRUD + 安全扫描 |
| Prompt Builder | `agent/prompt_builder.py` | 注入 Skills/记忆/上下文 |
| Memory Manager | `agent/memory_manager.py` | 记忆路由、聚合、预取 |
| Context Compressor | `agent/context_compressor.py` | 对话历史压缩 |
| RL Training | `tinker-atropos/` | GRPO 强化学习子模块 |

## Skills 系统

- **程序性记忆**：从经验提炼的可复用操作流程，存为 `SKILL.md`。
- **位置**: 内置 `skills/`（~25类）、用户自建 `~/.hermes/skills/`、Hub 扩展 `optional-skills/`。
- **渐进式加载**: `skills_list()` 只返回名称+描述（~3k token），按需调 `skill_view()` 获取完整内容。
- **条件激活**: 通过 frontmatter `platforms`、`fallback_for_tools` 等字段，按运行时环境动态激活。

## RL 训练集成

- 框架: Tinker-Atropos（git submodule），支持 GRPO 算法。
- 数据: Trajectory 文件记录完整操作轨迹，可批量生成用于训练。
- 用途: 对 Skill 创建质量、工具调用效率进行强化学习优化。

## 上下游依赖

- 下游参考: [[skills/hermes-skills.md]]（Skill 管理细节）、[[memory/hermes-memory.md]]（记忆系统细节）
- 对比参考: [[architecture/deerflow.md]]（LangGraph 风格 vs Hermes 自定义 Loop）
