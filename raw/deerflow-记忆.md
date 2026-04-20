# DeerFlow 记忆管理系统分析

---

## 数据结构

记忆以 JSON 文件存储，包含三大类型：

```json
{
  "version": "1.0",
  "lastUpdated": "2024-01-15T10:30:00Z",
  "user": {
    "workContext":    { "summary": "...", "updatedAt": "..." },
    "personalContext":{ "summary": "...", "updatedAt": "..." },
    "topOfMind":      { "summary": "...", "updatedAt": "..." }
  },
  "history": {
    "recentMonths":      { "summary": "...", "updatedAt": "..." },
    "earlierContext":    { "summary": "...", "updatedAt": "..." },
    "longTermBackground":{ "summary": "...", "updatedAt": "..." }
  },
  "facts": [
    {
      "id": "fact_abc123",
      "content": "User prefers TypeScript over JavaScript",
      "category": "preference",
      "confidence": 0.9,
      "createdAt": "2024-01-15T10:30:00Z",
      "source": "thread_xyz"
    }
  ]
}
```

### 字段说明

**user（用户上下文）**

| 字段 | 说明 |
|------|------|
| `workContext` | 工作信息（公司、项目、技术栈），2-3 句 |
| `personalContext` | 个人特征（语言、沟通偏好、兴趣），1-2 句 |
| `topOfMind` | 当前焦点（多个并发关注点），3-5 句，最常更新 |

**history（历史记录）**

| 字段 | 说明 |
|------|------|
| `recentMonths` | 最近 1-3 个月的活动详情，4-6 句 |
| `earlierContext` | 3-12 个月前的历史模式，3-5 句 |
| `longTermBackground` | 长期基础背景，2-4 句 |

**facts（事实库）**

| 字段 | 说明 |
|------|------|
| `id` | `fact_` + 8 位十六进制 |
| `category` | `preference` / `knowledge` / `context` / `behavior` / `goal` |
| `confidence` | 0-1，0.9-1.0 = 明确陈述，0.7-0.8 = 强烈暗示，0.5-0.6 = 推断 |
| `source` | 来源线程 ID |

---

## 存储位置

```
全局记忆:       ~/.deer-flow/memory.json
Agent 特定:     ~/.deer-flow/agents/{agent_name}/memory.json
```

`base_dir` 解析优先级（`config/paths.py`）：
1. 构造函数参数 `base_dir`
2. 环境变量 `DEER_FLOW_HOME`
3. 本地开发：`cwd/.deer-flow`（cwd 为 backend 目录时）
4. 默认：`$HOME/.deer-flow`

写入采用**原子操作**（`updater.py`）：先写临时文件 `.tmp`，再重命名到目标文件，避免文件损坏。

---

## MemoryMiddleware 的实现

**文件**：`agents/middlewares/memory_middleware.py`

触发点：`after_agent()` 钩子，位于中间件链第 10 位（agent 执行完成后）。

### 消息过滤

`_filter_messages_for_memory()` 决定哪些消息送入记忆更新：

| 类型 | 处理 |
|------|------|
| Human messages | 保留，但删除 `<uploaded_files>` 块 |
| AI messages（无 tool_calls） | 保留（最终回答） |
| AI messages（有 tool_calls） | 过滤掉（中间推理步骤） |
| Tool messages | 过滤掉（工具执行结果） |
| 仅含上传块的消息 | 整轮跳过（包括配对的 AI 回应） |

### 异步队列机制

`MemoryUpdateQueue` 管理后台更新：

```
after_agent() 被触发
    ↓
queue.add(thread_id, filtered_messages, agent_name)
    ├─ 同一线程去重：最新对话覆盖旧的
    └─ 重置防抖计时器（默认 30 秒）
    ↓
30 秒后 _process_queue() 执行
    ├─ 取出所有待处理对话
    ├─ 为每个创建 MemoryUpdater 实例
    ├─ 调用 updater.update_memory()
    └─ 批次间延迟 0.5 秒（避免速率限制）
```

全局单例（线程安全）：`get_memory_queue()` 通过 `threading.Lock()` 保证。

---

## 记忆注入到系统提示的流程

**文件**：`agents/lead_agent/prompt.py`

```
make_lead_agent(config)
    └─ apply_prompt_template(agent_name=X)
        └─ _get_memory_context(agent_name)
            ├─ 检查 config.enabled 和 config.injection_enabled
            ├─ get_memory_data(agent_name) → 读取 memory.json
            └─ format_memory_for_injection(data, max_tokens=2000)
                ├─ 按置信度降序排列 facts
                ├─ 逐条累加 token 数，超预算停止
                └─ 返回格式化字符串
        └─ 包裹为 <memory>...</memory> 注入系统提示
```

注入后的系统提示片段：

```
<memory>
User Context:
- Work: ...
- Personal: ...
- Current Focus: ...

History:
- Recent: ...
- Earlier: ...

Facts:
- [preference | 0.95] User prefers TypeScript
- [knowledge | 0.90] User knows LangGraph
</memory>
```

token 预算策略：
- 使用 `tiktoken`（`cl100k_base`）精确计数，无则用字符数 / 4 估算
- 高置信度 facts 优先进入预算
- 超出预算时末尾追加 `...`，保留 95% 余量

---

## 记忆更新：LLM 提取

**文件**：`agents/memory/updater.py`

`MemoryUpdater.update_memory()` 调用链：

```
1. get_memory_data(agent_name)          → 读取当前 memory.json
2. format_conversation_for_update()     → 将过滤后的消息格式化为文本
3. 构建 MEMORY_UPDATE_PROMPT            → 包含当前记忆 + 新对话
4. model.invoke(prompt)                 → 调用 LLM（thinking_enabled=False）
5. 解析 LLM 返回的 JSON
   ├─ 更新 user / history 各字段
   ├─ 新增 facts（内容去重）
   ├─ 删除 factsToRemove 中的 ID
   └─ 强制 max_facts 上限（按置信度保留 top N）
6. _strip_upload_mentions_from_memory() → 移除临时文件引用
7. _save_memory_to_file()               → 原子写入
```

LLM 返回的 JSON 格式：

```json
{
  "user": {
    "workContext": { "summary": "...", "shouldUpdate": true }
  },
  "history": { ... },
  "newFacts": [
    { "content": "...", "category": "preference", "confidence": 0.9 }
  ],
  "factsToRemove": ["fact_abc123"]
}
```

---

## Agent 记忆隔离

`agent_name` 贯穿整个调用链：

```
config["configurable"]["agent_name"]
    ├─ apply_prompt_template(agent_name=X)    → 注入对应记忆
    ├─ MemoryMiddleware(agent_name=X)         → 更新对应记忆
    └─ get_memory_data(agent_name=X)
        ├─ X = None    → ~/.deer-flow/memory.json              （全局）
        └─ X = "alice" → ~/.deer-flow/agents/alice/memory.json
```

默认 Lead Agent 使用全局记忆，自定义 agent 各自拥有独立的记忆文件。

---

## `/api/memory` REST API

**文件**：`app/gateway/routers/memory.py`

| 方法 | 端点 | 功能 |
|------|------|------|
| GET | `/api/memory` | 获取全局记忆数据 |
| POST | `/api/memory/reload` | 强制从文件重新加载（刷新缓存） |
| GET | `/api/memory/config` | 获取记忆系统配置 |
| GET | `/api/memory/status` | 获取配置 + 数据的组合响应 |

---

## config.yaml 配置项

```yaml
memory:
  enabled: true
  injection_enabled: true
  max_injection_tokens: 2000
  debounce_seconds: 30
  max_facts: 100
  fact_confidence_threshold: 0.5
  model_name: null          # null 表示继承全局 model_name
```

---

## 完整执行链

```
用户发送消息
    ↓
make_lead_agent() 初始化
    ├─ 读取 memory.json
    ├─ format_memory_for_injection()（token 预算，高置信度优先）
    └─ 注入 <memory> 到系统提示

Agent 推理（LLM 可见记忆上下文）
    ↓
after_agent() — MemoryMiddleware
    ├─ 过滤消息（保留用户输入 + 最终 AI 回答）
    └─ queue.add(thread_id, messages, agent_name)
    ↓
MemoryUpdateQueue（防抖 30s，同线程去重）
    ↓
MemoryUpdater.update_memory()
    ├─ 构建提示：当前记忆 + 新对话
    ├─ 调用 LLM 提取更新（thinking 关闭，节省成本）
    ├─ 应用 JSON 更新（增/删 facts，更新摘要）
    └─ 原子写入 memory.json
    ↓
下次对话时读取更新后的记忆
```

---

## 关键文件清单

| 文件 | 作用 |
|------|------|
| `agents/middlewares/memory_middleware.py` | 触发入口、消息过滤、队列调用 |
| `agents/memory/updater.py` | LLM 提取、更新应用、原子写入 |
| `agents/memory/queue.py` | 防抖队列、全局单例管理 |
| `agents/memory/loader.py` | 读取 memory.json、mtime 缓存 |
| `agents/memory/formatter.py` | 格式化注入内容、token 预算管理 |
| `agents/memory/prompt.py` | MEMORY_UPDATE_PROMPT 定义 |
| `agents/lead_agent/prompt.py` | `_get_memory_context()` 注入逻辑 |
| `config/memory_config.py` | 配置模型定义 |
| `config/paths.py` | base_dir 解析 |
| `app/gateway/routers/memory.py` | REST API |
