# Agent 路由引擎

## 一句话定位
根据消息内容和上下文，按优先级规则决定将请求分发给哪个 Agent。

## 核心设计决策 (ADR)

- **决策**: 七级优先级路由（P0–P6）：显式指令 > Session 延续 > @mention > 渠道/角色/部门绑定 > LLM 意图分类 > 关键词 > 默认 Agent。
- **背景**: 静态规则（P0-P3）覆盖明确意图，P4 LLM 意图分类处理模糊请求，P5 关键词做规则兜底，P6 兜底不让请求丢失。
- **后果**: 命中即停，不会多余执行；P4 引入额外 LLM 调用延迟（通常 <200ms），置信度低时自动降到 P5；P4 是可选组件，`intent_classifier=None` 时直接跳过。

## 路由优先级

```
P0  /agent <name> <text>      显式命令指定（最高优先级）
P1  Session.agent_id 有效     对话进行中，延续上一个 Agent
P2  @mention 匹配             消息中 @了某个 Agent 的别名
P3  渠道 + 群 / 角色 / 部门绑定   支持时间窗口（工作时间/非工作时间路由不同 Agent）
P4  LLM 意图分类              轻量 LLM 推断消息意图标签，置信度低则降到 P5
P5  关键词匹配                 消息文本包含预配置关键词（any / all / regex 三种模式）
P6  默认 Agent                全局兜底，返回 general-assistant 或错误提示
```

## 路由规则数据模型

规则存储在 PostgreSQL `routing_rules` 表，支持细粒度匹配条件：

| 匹配维度 | 字段 | 示例 |
|---|---|---|
| 渠道 | `channel` | dingtalk / wecom / feishu / * |
| 对话类型 | `chat_type` | direct / group |
| 群 ID | `chat_id` / `chat_id_pattern` | 精确或正则匹配 |
| 用户 | `sender_id` / `sender_role` / `sender_dept` | 按人 / 角色 / 部门 |
| 意图标签 | `intent_labels` | ["data_analysis", "report"] |
| 关键词 | `keyword_pattern` + `keyword_mode` | any / all / regex |
| 时间条件 | `active_from` + `active_until` + `active_weekdays` | 仅工作时间生效 |
| 租户 | `tenant_id` | 多租户隔离 |

同一 Priority Level 内用 `priority_order` 区分顺序（数值越大越优先）。路由决策结果记录在 `routing_logs` 表（评估过的所有规则、命中层级、LLM 置信度），用于审计和调试。

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

## 参考来源

- 参考了 [[architecture/gateway.md]] 的 OpenClaw binding 机制（peer > channel > default 思路）

## 关键注意点

- Session 延续依赖 Session 状态为 `active`，过期或 `/new` 后重新路由。
- `@mention` 在群聊场景下可能被误触，需配置精确的 mention 别名列表。
- P4 LLM 意图分类是可选组件，未配置时自动跳过，不影响其他层。
- 关键词匹配是全文包含，高频词（如"数据"）容易误命中，建议用短语或 regex 模式。
- 时间条件绑定（P3）：同一渠道可设两条规则，工作时间路由高级客服，非工作时间路由基础客服，`priority_order` 区分优先。
