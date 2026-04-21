# 消息全链路时序

> 涉及组件: [[architecture/gateway.md]] / [[architecture/agent-router.md]] / [[architecture/hermes.md]] / [[memory/memory-management.md]] / [[skills/skill-iteration.md]]
> 更新日期: 2026-04-21

## 概述

描述一条用户消息从进入系统到返回响应的完整路径，覆盖接入层、路由层、执行层三个阶段。是整个系统的骨架时序图，其他流程（Skill 加载、记忆读写、多 Agent 协调）都是从这条主链上分支出去的子流程。

## 时序图

```mermaid
sequenceDiagram
    autonumber
    actor User as 用户
    participant GW as Gateway<br/>(接入层)
    participant MQ as Redis Streams<br/>(消息队列)
    participant Router as Agent Router<br/>(路由引擎)
    participant Worker as AgentWorker<br/>(执行层)
    participant Agent as Agent Loop
    participant Memory as Memory Store<br/>(记忆层)
    participant Skills as Skill Manager
    participant LLM as LLM API

    User->>GW: 发送消息（IM Webhook / CLI）
    GW->>GW: 渠道适配（格式归一化）
    GW->>MQ: 入队（优先级: high / normal / low）
    GW-->>User: 同步回包 ≤40ms（"正在处理…"）

    MQ->>Router: Worker 拉取消息
    Router->>Router: 5 级优先级路由<br/>（显式绑定 > Session 锁 > @mention > 渠道默认 > 关键词）
    Router->>Worker: 分配给对应 AgentWorker

    Worker->>Memory: 读取记忆快照（冻结注入 System Prompt）
    Worker->>Skills: 构建 Level 0 Skill 索引（~1500 token）
    Worker->>Agent: 启动 Agent Loop（System Prompt + 对话历史）

    loop Agent 执行循环（最多 N 轮）
        Agent->>LLM: 发送请求（含工具定义）
        LLM-->>Agent: 返回（text / tool_call）
        alt tool_call = load_skill
            Agent->>Skills: load_skill(name) → Level 1 schema
            Skills-->>Agent: 返回 tool_schema，追加到 active_tools
        else tool_call = 业务工具
            Agent->>Agent: 执行工具（sql_query / web_search 等）
        else 纯文本回复
            Agent->>Agent: 退出循环
        end
    end

    Agent-->>Memory: 异步写入记忆更新（防抖队列，不阻塞）
    Agent->>Worker: 返回最终回复
    Worker->>GW: 推送结果
    GW->>User: 发送回复（IM 消息 / CLI 输出）
```

## 关键节点说明

| 节点 | 设计要点 | 参考 |
|---|---|---|
| Gateway 同步回包 | ≤40ms，满足 IM 平台要求；Agent 执行完全异步 | [[architecture/gateway.md]] |
| Redis Streams 入队 | 三级优先级；消费者组保证 at-least-once 投递 | [[architecture/gateway.md]] |
| 5 级路由 | 显式绑定 > Session 锁 > @mention > 渠道默认 > 关键词匹配 | [[architecture/agent-router.md]] |
| 记忆冻结快照 | Session 开始时固定，中途不更新，保证 Prompt Cache 命中率 | [[memory/memory-management.md]] |
| Level 0 Skill 索引 | 只注入名称+描述，~1500 token（全量注入约 9000 token） | [[skills/skill-iteration.md]] |
| 记忆异步写入 | 防抖队列批量 LLM 提取，不阻塞关键路径 | [[memory/memory-management.md]] |

## 异常路径

| 异常场景 | 处理方式 |
|---|---|
| LLM API 超时 | Worker 指数退避重试，超上限返回降级提示给用户 |
| 工具执行失败 | Agent 捕获错误，尝试替代路径或向用户说明原因 |
| Agent 超出最大轮次 | 强制退出循环，返回当前最佳结果并附说明 |
| 记忆写入失败 | 静默降级，不影响本次回复，下次 Session 记忆不更新 |
| 队列积压 | Worker 集群横向扩容（Redis Streams 消费者组支持） |

## 子流程索引

从本流程分支出去的详细时序图（待建）：

- `flows/skill-loading.md` — Skill 渐进加载（Level 0 → load_skill → Level 1 → Level 2）
- `flows/memory-lifecycle.md` — 记忆生命周期（Session 启动注入 → 对话中更新 → Session 结束巩固）
- `flows/multi-agent.md` — 多 Agent 协调（Lead Agent 委派 → 子 Agent 并发 → 结果汇总）
- `flows/agent-loop.md` — Agent 执行循环内部细节
