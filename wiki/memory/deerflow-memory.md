# DeerFlow 记忆系统

> 来源: [[raw/deerflow-记忆管理.md]] | 分析日期: 2026-04-21

## 一句话定位
DeerFlow 的长期记忆实现：对话结束后用 LLM 提取关键信息写入 JSON，下次对话开始时按置信度排序注入系统提示，实现跨 Session 个性化。

---

## 数据结构

```json
{
  "version": "1.0",
  "lastUpdated": "2024-01-15T10:30:00Z",
  "user": {
    "workContext":     {"summary": "...", "updatedAt": "..."},
    "personalContext": {"summary": "...", "updatedAt": "..."},
    "topOfMind":       {"summary": "...", "updatedAt": "..."}
  },
  "history": {
    "recentMonths":       {"summary": "...", "updatedAt": "..."},
    "earlierContext":     {"summary": "...", "updatedAt": "..."},
    "longTermBackground": {"summary": "...", "updatedAt": "..."}
  },
  "facts": [
    {
      "id": "fact_a1b2c3d4",
      "content": "User prefers TypeScript",
      "category": "preference",
      "confidence": 0.95,
      "createdAt": "2024-01-15T10:30:00Z",
      "source": "thread_xyz"
    }
  ]
}
```

三大类型：`user`（当前状态快照）、`history`（时间轴归档）、`facts`（结构化离散事实）。

---

## 记忆分类设计

### user / history 语义分层

| 区域 | 时间维度 | 放什么 |
|---|---|---|
| `user.workContext` | 当前 | 活跃项目、主要技术栈 |
| `user.personalContext` | 当前 | 语言偏好、沟通风格 |
| `user.topOfMind` | 当前 | 近期同时关注的 3-5 个焦点 |
| `history.recentMonths` | 近 1-3 个月 | 探索过的技术、解决过的问题 |
| `history.earlierContext` | 3-12 个月前 | 已完成项目、形成的工作模式 |
| `history.longTermBackground` | 长期不变 | 核心专长、基础风格 |

**设计要点**：`user` 频繁覆盖（旧内容直接替换），`history` 按时间沉淀归档，两者都是自然语言摘要，不可精确删除。

### facts 五大 category

| category | 含义 | 示例 |
|---|---|---|
| `preference` | 明确的喜好/厌恶 | "偏好 TypeScript，不喜欢 JavaScript" |
| `knowledge` | 已掌握的技术领域 | "熟悉 LangGraph 状态机" |
| `context` | 背景事实（职位、项目、地点） | "在某公司担任架构师" |
| `behavior` | 工作习惯、沟通方式 | "喜欢先看代码再提问" |
| `goal` | 明确表达的目标 | "计划今年掌握 Rust" |

> ⚠️ DeerFlow 只有 5 类 category，**无 `correction` 类**（中台设计补加了该类别，参考 [[memory/memory-management.md]]）。

---

## 置信度（Confidence）机制

### 赋值来源

| 置信度范围 | 含义 |
|---|---|
| 0.9 ~ 1.0 | 用户明确陈述（"记住我喜欢..."） |
| 0.7 ~ 0.8 | 对话中强烈暗示 |
| 0.5 ~ 0.6 | LLM 推断的模式（谨慎使用） |

置信度完全由 LLM 在返回 `newFacts` 时自行赋值，代码只做规范化（`_coerce_confidence()` 将 NaN/±inf 修正为 0.0）。

### 写入时过滤

```python
# 低于阈值的 fact 直接丢弃，不写入 facts.json
if fact.confidence < fact_confidence_threshold:  # 默认 0.7
    continue
```

低于 0.7 的事实被认为质量不可靠，静默丢弃。

### 注入时排序与截断

```python
# format_memory_for_injection()：按置信度降序，逐条累加 token
facts_sorted = sorted(facts, key=lambda f: f["confidence"], reverse=True)
for fact in facts_sorted:
    token_count += count_tokens(fact_text)
    if token_count > max_injection_tokens:  # 默认 2000
        break
    output.append(fact_text)
```

高置信度事实永远优先进入上下文，低置信度事实在预算不足时自然淘汰。

### max_facts 上限截取

```python
# 超出最大条数时，按置信度降序保留 top N
if len(facts) > max_facts:  # 默认 100
    facts = sorted(facts, key=lambda f: f.get("confidence", 0), reverse=True)[:max_facts]
```

低质量事实被永久淘汰，不参与后续注入。

---

## LLM 更新输出格式

每次记忆更新调用 LLM 后，期望返回如下 JSON diff：

```json
{
  "user": {
    "workContext":     {"summary": "...", "shouldUpdate": true},
    "personalContext": {"summary": "...", "shouldUpdate": false},
    "topOfMind":       {"summary": "...", "shouldUpdate": true}
  },
  "history": {
    "recentMonths":       {"summary": "...", "shouldUpdate": true},
    "earlierContext":     {"summary": "...", "shouldUpdate": false},
    "longTermBackground": {"summary": "...", "shouldUpdate": false}
  },
  "newFacts": [
    {"content": "User prefers TypeScript", "category": "preference", "confidence": 0.95}
  ],
  "factsToRemove": ["fact_a1b2c3d4"]
}
```

`shouldUpdate=false` 或 `summary=""` 时跳过覆盖，防止 LLM 返回空摘要误清除已有记忆。`factsToRemove` 由 LLM 判断哪些事实已被推翻，按 ID 集合一次性过滤。

---

## 防抖异步队列（MemoryUpdateQueue）

```
MemoryMiddleware.after_agent()
    └─ _filter_messages_for_memory()   # 只保留用户输入 + 最终 AI 回答
    └─ queue.add(thread_id, messages)  # 非阻塞入队

30 秒防抖后 _process_queue() 触发：
    ├─ 同线程去重：最新对话覆盖旧的（只调一次 LLM）
    ├─ 逐条 MemoryUpdater.update_memory()
    └─ 批次间延迟 0.5s（缓解 API 速率限制）
```

核心决策：**记忆更新不在关键路径上**，用户看到响应后记忆才开始更新，30 秒防抖值可通过 `config.yaml` 的 `debounce_seconds` 调整。

---

## ⚠️ 无时间衰减

**DeerFlow 目前没有时间衰减机制。** 官方 `MEMORY_IMPROVEMENTS.md` 明确标注时间衰减为 `Planned / not yet merged`。

DeerFlow 的淘汰机制只有两种：
1. **置信度门槛**：写入时 `confidence < 0.7` 丢弃。
2. **容量压力**：`max_facts` 溢出时按置信度排序截取低分 fact。

中台设计在此基础上补加了时间维度，参考 [[memory/memory-management.md]] 衰减机制章节。

---

## 关键模块

| 文件 | 职责 |
|---|---|
| `memory/updater.py` | 记忆读写、mtime 缓存、LLM 更新逻辑、原子写入 |
| `memory/queue.py` | 防抖异步更新队列（`MemoryUpdateQueue`） |
| `memory/prompt.py` | 更新 Prompt 模板、`format_memory_for_injection()`、token 计数 |

---

## 配置项（config.yaml）

```yaml
memory:
  enabled: true
  injection_enabled: true
  max_injection_tokens: 2000      # 注入 token 上限
  debounce_seconds: 30            # 防抖等待时间
  max_facts: 100                  # 事实库最大条数
  fact_confidence_threshold: 0.7  # 写入最低置信度
  model_name: null                # null = 继承全局模型
  storage_path: null              # null = 默认路径 ~/.deer-flow/memory.json
```

---

## 局限性

| 问题 | 现状 |
|---|---|
| 无时间衰减 | 旧事实只要 confidence 高就永远留存，可能不反映用户现状变化 |
| 同 Session 内不生效 | 对话结束后异步更新，本次 Session 新增记忆要下次才能注入 |
| 全量覆写 JSON | 并发写入不安全（多进程场景） |
| 单用户本地设计 | `~/.deer-flow/memory.json` 无多租户隔离 |
| per-agent 记忆孤岛 | agent 间记忆路径隔离，信息不共享 |

---

## 对中台设计的启示

- 采用了 DeerFlow 的 **防抖异步队列**（`debounce_seconds` 可配置），参考点: [[memory/memory-management.md]] §记忆更新机制
- 采用了 DeerFlow 的 **置信度注入排序 + token 预算截断**，参考点: [[memory/memory-management.md]] §置信度体系
- **补加了时间衰减**（DeerFlow 缺失），参考 OpenClaw 的半衰期设计，参考点: [[memory/memory-management.md]] §衰减机制
- **补加了 `correction` category**（DeerFlow 只有 5 类），用于记录用户纠正事件

---

## 上下游依赖

- 上游: [[architecture/deerflow.md]]（MemoryMiddleware 在 Agent Loop 结束后触发）
- 下游: [[context/deerflow-context.md]]（Memory Injection 注入系统提示）
- 对比: [[memory/hermes-memory.md]]（冻结快照 vs DeerFlow 的动态更新）
- 中台设计: [[memory/memory-management.md]]
