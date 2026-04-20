# Agent 路由引擎

## 一句话定位
根据消息内容和上下文，按优先级规则决定将请求分发给哪个 Agent。

## 核心设计决策 (ADR)

- **决策**: 五级优先级路由：显式指令 > Session 延续 > @mention > 渠道绑定 > 关键词 > 默认 Agent。
- **背景**: ToB 场景中用户习惯用 @mention，同时需要支持多轮对话的 Session 延续。
- **后果**: 规则清晰易维护，但多条规则同时命中时只取最高优先级，存在"规则遮蔽"问题。

## 路由优先级

```
P0  /agent <name> <text>    显式命令指定（最高优先级）
P1  Session.agent_id 有效   对话进行中，延续上一个 Agent
P2  @mention 匹配           消息中 @了某个 Agent 的别名
P3  渠道 + 群组绑定          特定群绑定特定 Agent
P4  关键词匹配               消息文本包含预配置关键词
P5  默认 Agent              兜底（可配置或返回错误）
```

## Agent 定义关键字段

```json
{
  "agent_id": "data-analyst",
  "routing_rules": [
    {"match": "mentions",  "pattern": ["@数据助手", "@data"]},
    {"match": "channel",   "pattern": {"channel": "feishu", "chat_id": "oc_xxx"}},
    {"match": "keyword",   "pattern": ["数据分析", "报表", "销售数据"]}
  ],
  "limits": {"max_concurrent": 20, "timeout_seconds": 300}
}
```

## 上下游依赖

- 上游: [[gateway.md]] MessagePipeline（调用 `resolve_agent()`）
- 下游: Worker 执行集群（拿到 agent_id 后入队）

## 关键注意点

- Session 延续依赖 Session 状态为 `active`，过期或 `/new` 后重新路由。
- `@mention` 在群聊场景下可能被误触，需配置精确的 mention 别名列表。
- 关键词匹配是全文包含，高频词（如"数据"）容易误命中，建议用短语。
