# AI 中台记忆管理

## 一句话定位
企业级多 Agent 平台的记忆体系，覆盖短期（Session）到长期（跨 Session）的全生命周期管理，参考 OpenClaw/Hermes/DeerFlow 三个开源项目设计。

## 记忆层次模型

### OpenClaw 四层体系（参考基准）

| 层次 | 载体 | 生命周期 | 触发 |
|---|---|---|---|
| 短期记忆 | Session JSONL 逐行追加 | Session 存续期 | 实时写入 |
| 中期记忆 | `memory/YYYY-MM-DD.md` | 跨 Session，可检索 | `/new` 或 `/reset` |
| 长期记忆 | MEMORY.md / USER.md 等 workspace 文件 | 永久，每次 Session 注入 | Agent 自主 read/edit |
| 启动记忆 | BOOT.md / HEARTBEAT.md | 永久 | 网关启动 / 定时心跳 |

## 中台设计方案

### 存储架构

```
热存储:   Redis          → 当前 Session 上下文（最近 N 条消息）
温存储:   PostgreSQL     → 历史 Session 消息、用户事实、摘要
冷存储:   Vector DB      → 语义检索（长期记忆、知识库）
```

### 记忆更新机制

1. **防抖异步队列（参考 DeerFlow）**: 对话结束后延迟 N 秒批量写，避免每条消息都触发 LLM 提取。
2. **LLM 摘要提取**: 用专用 Prompt 从对话历史中抽取结构化事实（用户偏好、决策、错误上下文）。
3. **置信度标注**: 每条事实记录 `confidence` 分，低置信度事实需二次验证才写入长期记忆。

### 记忆衰减策略

- 时间衰减：`effective_weight = base_weight * exp(-λ * days_elapsed)`
- 访问强化：每次命中检索，权重 × 1.1（上限 cap）
- 睡眠巩固（v1.2 新增）：低峰期（凌晨 2-4 点）跨 Session 提炼模式，合并近似记忆、删除过期记忆、构建记忆关系图

### 记忆注入策略

- 系统提示注入：每次 Session 启动时，检索 Top-K 相关记忆写入 System Prompt。
- 渐进式加载：System Prompt 只含摘要，详情按需 `memory_tool.get()` 拉取。

## 关键数据结构

```json
{
  "id": "fact_abc123",
  "content": "用户偏好 TypeScript",
  "category": "preference",
  "confidence": 0.9,
  "createdAt": "2024-01-15T10:30:00Z",
  "source": "thread_xyz",
  "decay_weight": 0.95
}
```

## 上下游依赖

- 上游: [[architecture/gateway.md]]（Session 触发记忆写入）
- 下游: [[context/context-management.md]]（Memory Injection / Memory Flush 交汇点）
- 安全: [[security/agent-security.md]]（记忆访问控制、跨租户隔离）
- 参考实现: [[memory/deerflow-memory.md]]、[[memory/hermes-memory.md]]
