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

1. **防抖异步队列（参考 DeerFlow）**: 同一 Session 多轮更新覆盖合并，配置 `debounce_seconds` 延迟后批量触发 LLM 提取，避免每条消息都调一次 LLM。
2. **LLM 结构化提取**: 输出 `newFacts[]` + `factsToRemove[]` 的 JSON diff，而非原始消息存档。
3. **纠正/强化信号检测（参考 DeerFlow）**: 正则检测用户话语中的纠正信号（"你理解错了"、"重试"）和正向强化（"对，就是这样"），检测到纠正时重要度 ≥ 0.95 并记录 `correction_note`（错误做法）。
4. **写入安全扫描（参考 Hermes）**: 入库前检查 prompt injection 模式、凭证泄露尝试、不可见 Unicode 字符。

---

### 置信度体系（Confidence）

每条 fact 记录 `confidence`（0.0–1.0），贯穿写入、注入、淘汰三个环节：

**写入时赋值**

| 来源 | 初始 confidence |
|---|---|
| 用户明确声明（"记住我喜欢…"） | 0.9 |
| 纠正信号检测（"你做错了"） | ≥ 0.95 |
| LLM 推断（偏好/经验） | 0.5–0.8 |
| 事件记录（episodic） | 0.3–0.5 |

**Fact 六大分类**（参考 DeerFlow）：

| category | 含义 | 示例 |
|---|---|---|
| preference | 工具/风格/方法偏好 | "用户喜欢简洁回复" |
| knowledge | 专业知识/掌握技术 | "用户熟悉 TypeScript" |
| context | 背景事实（职位/项目） | "用户负责 CRM 系统" |
| behavior | 工作模式/沟通习惯 | "用户总是先问确认再执行" |
| goal | 目标/学习计划 | "用户计划学习 Rust" |
| correction | 明确纠正（含 correction_note 字段） | content=正确做法，note=之前的错误 |

**注入时过滤**：按 confidence 降序排列，逐条累加 token，超出 `max_injection_tokens` 预算截断。低于 `confidence_threshold`（默认 0.5）的 fact 不注入。

**淘汰时截断**：`len(facts) > max_facts` 时按 confidence 排序，保留 top N，低分 fact 直接丢弃。

---

### 衰减机制

三种衰减策略融合（参考三个开源项目取长补短）：

**1. 时间半衰期（参考 OpenClaw）**

```python
temporal_decay = 0.5 ** (age_days / HALF_LIFE_DAYS[memory_tier])
```

| 记忆层 | 半衰期 | 归档触发 |
|---|---|---|
| short_term | 7 天 | 超 7 天未访问 → 自动归档 |
| long_term | 90 天 | 超 90 天未访问 → importance × 0.5；< 0.1 → 归档 |
| permanent | 不衰减 | 永不归档 |

**2. 使用频率加成（参考 Hermes Holographic Provider）**

```python
access_boost = min(1.0 + log1p(access_count) * 0.1, 2.0)
```

被检索/引用越频繁，综合评分越高，对抗时间衰减。

**3. 综合评分（中台设计）**

```python
score = importance × temporal_decay × access_boost
```

`score < 0.1` 时归档，不再参与检索。

> **⚠️ DeerFlow 注意**：DeerFlow 目前只有 confidence 淘汰，**无时间衰减**（官方 MEMORY_IMPROVEMENTS.md 确认为 Planned 未实现）。中台设计在此基础上补加时间维度。

---

### 睡眠巩固（Memory Distiller）

定时任务，建议每天凌晨 2–4 点低峰期运行，流程：

```
收集 ≥24 小时的短期记忆（≥3 条才值得提炼）
    ↓
按主题聚类（向量相似度分组）
    ↓
单条记忆 → 直接升级为 long_term
多条同簇 → LLM 合并提炼为一条（输出 content + type + tags）
    ↓
提炼后的记忆 importance = max(簇内最高) × 1.1（略提升）
原始短期记忆标记为 superseded
    ↓
_decay_unused_memories()：短期 7 天归档 / 长期 90 天减半
```

**关键约束**：提炼时有矛盾 → LLM 以最新为准；提炼结果插入前再次去重（向量相似度 > 0.95 则跳过）。

---

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
