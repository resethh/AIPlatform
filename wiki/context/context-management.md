# AI 中台上下文管理

## 一句话定位
单次 Session 内如何高效利用有限 Context Window：System Prompt 组装、对话历史压缩、Token 成本控制。与记忆管理的分界点在于"单次 Session 内 vs 跨 Session"。

## 上下文管理 vs 记忆管理

```
记忆管理  →  跨 Session 持久化  →  用户偏好/历史事实/组织知识  →  PG + Vector DB
上下文管理 →  单次 Session 内   →  Context Window 利用效率      →  Redis（热数据）

交汇点:
  Memory Flush:     压缩前把即将丢弃的对话提取到记忆
  Memory Injection: 记忆检索结果注入到 System Prompt
```

## 核心挑战

| 挑战 | 说明 |
|---|---|
| Context Window 有限 | 有效注意力远低于名义上限（GPT-4o 128K 的有效区间约前 32K） |
| Tool 输出膨胀 | 单次 SQL 返回可能 50K chars，文件读取可能 100K |
| 成本控制 | 每次请求塞满 128K 会烧预算，Token 用量需精细管理 |
| 压缩质量 | 不能丢关键信息（文件路径、决策、错误上下文） |
| 多租户隔离 | 不同租户的 Agent 配置和上下文策略需要隔离 |

## System Prompt 组装（ContextAssembler）

```
Agent 角色定义 / 系统指令
渠道格式适配提示（钉钉/企微/飞书回复风格不同）
Skill Prompt（Agent 绑定的 Skill 描述目录）
Tool Schemas（可用工具列表，渐进式加载）
Memory Injection（记忆检索 Top-K 结果）
Safety Rules
```

**渐进式加载原则**：System Prompt 只注入工具目录索引，详情在 Agent 主动查询时才拉取，避免一次性撑满上下文。

## 对话历史压缩策略（参考 Hermes/DeerFlow）

1. **滑动窗口**：保留最近 N 条消息，超出部分直接丢弃（简单，但丢信息）。
2. **LLM Summarization**：把旧消息压缩成摘要段落追加回上下文（保信息，但有 LLM 调用成本）。
3. **Memory Flush**：压缩前先把关键信息提取到记忆层，再丢弃原始消息（最完整，成本最高）。
4. **Tool 输出截断**：Tool 返回超过阈值（如 10K chars）时，只保留前 N 行 + 末尾 N 行。

## 关键接口

- `ContextAssembler.build(agent, session, memory_results)` → 返回组装好的 messages 列表
- `ContextCompressor.compress(messages, budget_tokens)` → 返回压缩后的 messages
- `MemoryFlusher.flush_before_compress(messages)` → 提取关键信息写入记忆层

## 上下游依赖

- 上游: [[architecture/gateway.md]] Worker 在构建 Prompt 时调用
- 下游: [[memory/memory-management.md]]（Memory Flush/Injection 交汇点）
- 参考实现: [[context/deerflow-context.md]]
