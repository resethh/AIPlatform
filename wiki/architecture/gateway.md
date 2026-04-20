# 企业级多智能体网关

## 一句话定位
多渠道（钉钉/企微/飞书）接入的统一 AI 网关，负责消息标准化、Agent 路由、异步队列调度和回复投递。

## 核心设计决策 (ADR)

- **决策**: 接入层与执行层通过 Redis Streams 解耦，Webhook 同步响应 ≤40ms，Agent 执行异步进行。
- **背景**: 各 IM 平台 Webhook 有严格超时限制（飞书 3 秒），LLM 调用通常需要 2-10 秒。
- **后果**: 用户体验好（不超时），但引入了消息队列复杂性，需处理消费者组、ACK、死信。

- **决策**: Session 使用分布式锁（Redis String，TTL=30s），同一 Session 同一时刻只有一个 Worker 在执行。
- **背景**: 防止并发请求打乱对话上下文顺序。
- **后果**: 同一会话串行执行，高并发单用户场景有排队延迟。

## 架构分层

```
渠道适配层 → 消息处理管道 → Redis Streams 队列
                                    ↓
                             Worker 执行集群（Agent Loop）
                                    ↓
                             Reply Dispatcher → 各渠道回复 API
```

## 核心数据流

1. **UnifiedMessage**: 所有渠道消息统一格式，含 source / content / routing / metadata。
2. **消息管道 (MessagePipeline)**: 幂等去重 → 用户认证 → Session 解析 → Agent 路由 → 限流 → 入队。
3. **Worker AgentLoop**: 获取 Session 锁 → 加载配置 → 构建 Prompt → LLM 调用（支持多轮 Tool Calling）→ 保存 Session → 发布回复。
4. **ReplyDispatcher**: 监听 `reply:outbox` Stream → 调对应渠道 API 投递。

## 关键接口/契约

- 入队: `XADD task:normal {...}` — 字段包含 agent_id / session_id / text / reply_channel 等
- 回复: `XADD reply:outbox {...}` — 由 Worker 写入，Dispatcher 消费
- Session 锁: `SET session:{id}:lock worker_id EX 30 NX`

## Redis 数据结构

| Key 模式 | 类型 | 用途 |
|---|---|---|
| `session:{id}` | Hash | Session 元数据 |
| `session:{id}:context` | List | 最近 N 条消息 |
| `task:high/normal/low` | Stream | 三级优先级队列 |
| `reply:outbox` | Stream | 待投递回复 |
| `dedup:{channel}:{msg_id}` | String | 幂等去重，TTL=300s |
| `ratelimit:user:{id}` | String | 用户限流计数，TTL=60s |

## 上下游依赖

- 上游: 各渠道 Webhook（钉钉/企微/飞书）
- 下游: [[hermes.md]]、[[deerflow.md]]（Agent 执行模式参考）、[[agent-router.md]]（路由决策）
- 依赖存储: PostgreSQL（Agent 配置、会话历史、审计日志）、Redis（队列、热数据）

## 部署规格参考

| 规模 | 预估并发 | 日活 |
|---|---|---|
| 最小（单机 4核8G） | 50 | 500 |
| 标准（分层 ~8台） | 200 | 2000 |
