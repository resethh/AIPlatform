# DeerFlow Agent 架构

## 一句话定位
基于 LangChain `create_agent()` 工厂函数构建的多 Agent 系统，Lead Agent 负责任务分解和子 Agent 协调。

## 核心设计决策 (ADR)

- **决策**: 使用高级 `create_agent()` 工厂而非直接操作 LangGraph `StateGraph` API。
- **背景**: 工厂函数封装了大量样板代码（工具绑定、消息处理、错误恢复），开发效率更高。
- **后果**: 灵活性略低，深度定制需绕过工厂函数；但适合快速迭代。

- **决策**: 记忆更新、Token 统计等耗时操作全部异步，不阻塞用户响应。
- **背景**: 记忆写入需要 LLM 调用（提取关键信息），可能耗时 1-3 秒。
- **后果**: 关键路径延迟低，但异步队列需处理失败重试和顺序保证。

## Agent 结构

```
make_lead_agent()           Lead Agent — 任务接收、分解、协调
    └── sub-agents          各专业子 Agent（由 Lead 动态调度）
         └── checkpointer   async_provider.py — 状态持久化
```

注册位置：`backend/langgraph.json`

## 上下文管理三层模型

```
┌────────────────────────────────────────────────┐
│  系统提示（System Prompt）                        │
│  角色定义 + 记忆注入 + 技能目录 + 子 Agent 规则   │
├────────────────────────────────────────────────┤
│  对话历史（Messages）                             │
│  当前会话内全部轮次（可被 Summarization 压缩）     │
├────────────────────────────────────────────────┤
│  运行时注入（Runtime Injection）                   │
│  上传文件 + 图像 base64 + 工具调用修复             │
└────────────────────────────────────────────────┘
```

## 核心设计原则

- **关键路径不阻塞**: 记忆更新等耗时操作全部异步
- **渐进式加载**: 技能/工具按需读取，系统提示只含目录索引
- **原子一致性**: checkpointer 和记忆写入都用原子操作

## 上下游依赖

- 下游参考: [[memory/deerflow-memory.md]]（记忆实现）、[[context/deerflow-context.md]]（上下文策略）
- 对比: [[architecture/hermes.md]]（自定义 Loop vs LangChain 工厂）
