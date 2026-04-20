# Memory 模块说明

`deerflow/agents/memory/` 负责 DeerFlow 的长期记忆机制：在每次对话结束后，用 LLM 提取关键信息写入 JSON 文件，并在下次对话开始时将记忆注入系统提示，实现跨会话的个性化。

---

## 文件结构

```
memory/
├── __init__.py   # 公开接口汇总
├── prompt.py     # 提示词模板 + 格式化工具函数
├── queue.py      # 防抖异步更新队列
└── updater.py    # 记忆读写 + LLM 更新逻辑
```

---

## `__init__.py` — 公开接口

统一导出模块内所有对外接口，外部代码只需从 `deerflow.agents.memory` 导入。

| 导出符号 | 来源 | 用途 |
|---------|------|------|
| `MEMORY_UPDATE_PROMPT` | prompt.py | 记忆更新主提示词 |
| `FACT_EXTRACTION_PROMPT` | prompt.py | 单条消息事实提取提示词 |
| `format_memory_for_injection` | prompt.py | 格式化记忆供注入系统提示 |
| `format_conversation_for_update` | prompt.py | 格式化对话供 LLM 分析 |
| `ConversationContext` | queue.py | 单条待处理对话的数据类 |
| `MemoryUpdateQueue` | queue.py | 防抖异步更新队列 |
| `get_memory_queue` | queue.py | 获取全局队列单例 |
| `reset_memory_queue` | queue.py | 重置队列（测试用） |
| `MemoryUpdater` | updater.py | LLM 驱动的记忆更新器 |
| `get_memory_data` | updater.py | 读取记忆（带 mtime 缓存） |
| `reload_memory_data` | updater.py | 强制从文件重新加载 |
| `update_memory_from_conversation` | updater.py | 一步完成记忆更新的便捷函数 |

---

## `prompt.py` — 提示词与格式化

### 提示词模板

**`MEMORY_UPDATE_PROMPT`**
- 输入占位符：`{current_memory}`（当前记忆 JSON）、`{conversation}`（对话文本）
- 输出：LLM 返回结构化 JSON，包含 `user`/`history` 更新、`newFacts`、`factsToRemove`
- 指导 LLM 按时间维度分层更新记忆，提取带置信度的事实

**`FACT_EXTRACTION_PROMPT`**
- 轻量级，仅从单条消息提取 `facts`，不更新 `user`/`history`

### 工具函数

**`_count_tokens(text)`**
- 优先使用 `tiktoken`（`cl100k_base` 编码）精确计数
- `tiktoken` 不可用时降级为 `len(text) // 4` 估算

**`_coerce_confidence(value)`**
- 将置信度值规范化到 `[0, 1]`
- NaN / ±inf 视为无效，回退到默认值 `0.0`，防止排序异常

**`format_memory_for_injection(memory_data, max_tokens=2000)`**
- 按 User Context → History → Facts 顺序组装文本
- Facts 按置信度降序排列，逐条累加 token，超出预算立即截止
- 最终兜底截断：若整体超出预算，按字符比例截断并追加 `...`

**`format_conversation_for_update(messages)`**
- 多模态消息（list 类型内容）拼接其中的文本块
- 剔除 `<uploaded_files>` 块（上传路径是会话临时数据）
- 纯上传消息（剔除后为空）整条跳过
- 超过 1000 字符的消息截断，避免提示词过长

---

## `queue.py` — 防抖异步更新队列

### `ConversationContext`（dataclass）

| 字段 | 类型 | 说明 |
|------|------|------|
| `thread_id` | `str` | 线程 ID，用于去重和记忆文件隔离 |
| `messages` | `list` | 过滤后的消息（用户输入 + 最终 AI 回答） |
| `timestamp` | `datetime` | 入队时间 |
| `agent_name` | `str\|None` | `None` = 全局记忆；非 `None` = per-agent 记忆 |

### `MemoryUpdateQueue` 核心机制

```
add(thread_id, messages, agent_name)
    ├─ 同线程去重：最新对话覆盖旧的
    └─ 重置防抖计时器（默认 30 秒）

30 秒后 _process_queue() 触发
    ├─ 加锁，防止并发重入（_processing 标志）
    ├─ 取出队列快照，立即清空（允许新对话继续入队）
    ├─ 逐条调用 MemoryUpdater.update_memory()
    ├─ 多条时批次间延迟 0.5 秒（缓解 API 速率限制）
    └─ finally 复位 _processing 标志
```

| 方法 | 说明 |
|------|------|
| `add()` | 入队，同线程去重，重置计时器 |
| `flush()` | 取消计时器，立即同步处理（测试/优雅关闭） |
| `clear()` | 清空队列不触发处理（测试重置） |
| `pending_count` | 当前待处理条目数 |
| `is_processing` | 是否正在处理 |

### 全局单例

```python
get_memory_queue()   # 线程安全的懒初始化单例
reset_memory_queue() # 销毁单例（测试用）
```

---

## `updater.py` — 记忆读写与 LLM 更新

### 存储路径

```
全局记忆:       ~/.deer-flow/memory.json
Per-agent:     ~/.deer-flow/agents/{agent_name}/memory.json
```

路径解析优先级：`config.storage_path`（绝对/相对）> `get_paths().memory_file`（默认）

### 内存缓存（`_memory_cache`）

```python
_memory_cache: dict[str | None, tuple[dict, float | None]]
# key:   agent_name（None = 全局）
# value: (数据字典, 文件 mtime)
```

- `get_memory_data()` 对比当前 mtime 与缓存 mtime，不一致时重新从磁盘加载
- 写入后立即更新缓存 mtime，避免误判为外部修改

### 原子写入（`_save_memory_to_file`）

```
1. 更新 lastUpdated 时间戳
2. 写入临时文件 memory.tmp
3. rename() 原子覆盖目标文件
4. 读取新 mtime，同步更新缓存
```

### `_extract_text(content)`

兼容 LLM 返回 `str` 和 `list[block]` 两种格式：
- 连续字符串块直接拼接（不加分隔符，保持 JSON 完整性）
- dict 块提取 `text` 字段，块间用换行连接

### `_strip_upload_mentions_from_memory(memory_data)`

用窄正则 `_UPLOAD_SENTENCE_RE` 清除记忆中描述上传事件的句子：
- 清洗 `user` / `history` 各字段的 `summary`
- 删除 `facts` 中匹配的条目
- 仅匹配上传动作，不误删 "User works with CSV files" 等合法描述

### `MemoryUpdater.update_memory()` 执行链

```
1. get_memory_data()                → 读取当前记忆（走缓存）
2. format_conversation_for_update() → 消息列表 → 纯文本对话
3. MEMORY_UPDATE_PROMPT.format()    → 构建完整提示词
4. model.invoke(prompt)             → 调用 LLM（thinking 关闭，节省成本）
5. 清理 markdown 代码块包裹         → 提取纯 JSON
6. json.loads()                     → 解析 LLM 返回
7. _apply_updates()                 → 应用更新（见下）
8. _strip_upload_mentions()         → 清除上传事件
9. _save_memory_to_file()           → 原子写入
```

### LLM 输出的 JSON 格式

`update_memory()` 步骤 4 收到 LLM 响应后，期望其返回如下结构（步骤 5 清除 markdown 代码块、步骤 6 `json.loads()` 解析）：

```json
{
  "user": {
    "workContext":     { "summary": "...", "shouldUpdate": true },
    "personalContext": { "summary": "...", "shouldUpdate": false },
    "topOfMind":       { "summary": "...", "shouldUpdate": true }
  },
  "history": {
    "recentMonths":       { "summary": "...", "shouldUpdate": true },
    "earlierContext":     { "summary": "...", "shouldUpdate": false },
    "longTermBackground": { "summary": "...", "shouldUpdate": false }
  },
  "newFacts": [
    { "content": "User prefers TypeScript", "category": "preference", "confidence": 0.95 }
  ],
  "factsToRemove": ["fact_a1b2c3d4"]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `user.*.summary` | `string` | 新摘要文本（空字符串视为无更新） |
| `user.*.shouldUpdate` | `bool` | `true` = 覆盖该区域；`false` = 跳过 |
| `history.*.summary/shouldUpdate` | 同上 | 同上，对应历史时间分层 |
| `newFacts[].content` | `string` | 事实描述，去首尾空白后用于去重 |
| `newFacts[].category` | `string` | `preference/knowledge/context/behavior/goal` |
| `newFacts[].confidence` | `float` | `0.0~1.0`，低于阈值（默认 0.7）丢弃 |
| `factsToRemove` | `string[]` | 要删除的已有事实 ID 列表 |

### `_apply_updates()` 更新逻辑

**① user / history 区域更新**

```
for section in [workContext, personalContext, topOfMind]:
    if shouldUpdate == true AND summary != "":
        覆盖 current_memory["user"][section]
        写入当前 UTC 时间到 updatedAt
```

- `shouldUpdate=false` 或 `summary=""` 时跳过，不覆盖现有内容（防止 LLM 返回空摘要误清除记忆）
- `history` 的三个子区域（recentMonths / earlierContext / longTermBackground）逻辑相同

**② factsToRemove — 删除过时事实**

```
facts_to_remove = set(update_data["factsToRemove"])
current_memory["facts"] = [f for f in facts if f["id"] not in facts_to_remove]
```

- 按 ID 集合一次性过滤，O(n) 操作
- LLM 负责判断哪些事实被新信息推翻或过时

**③ newFacts — 添加新事实（四道过滤）**

```
1. 置信度过滤：confidence < fact_confidence_threshold(0.7) → 丢弃
2. 内容规范化：content.strip() → normalized_content
3. 去重检查：normalized_content in existing_fact_keys → 跳过（同批次内也去重）
4. 写入：生成 fact_{uuid8hex}，记录 createdAt + source(thread_id)
```

- `existing_fact_keys` 在函数入口一次性构建（现有事实的 strip 后内容集合），同批次内新增的 key 也实时追加进去，避免同一次 LLM 返回中出现重复事实
- `source` 字段写入 `thread_id`，便于追溯来源会话

**④ max_facts 上限截取**

```
if len(facts) > max_facts(100):
    facts = sorted(facts, key=confidence, reverse=True)[:100]
```

- 超出上限时，按置信度降序保留质量最高的 N 条，低质量事实被淘汰
- 默认上限 100，可通过 `config.yaml` 的 `memory.max_facts` 调整

---

## 记忆数据结构（memory.json）

```json
{
  "version": "1.0",
  "lastUpdated": "2024-01-15T10:30:00Z",
  "user": {
    "workContext":     { "summary": "...", "updatedAt": "..." },
    "personalContext": { "summary": "...", "updatedAt": "..." },
    "topOfMind":       { "summary": "...", "updatedAt": "..." }
  },
  "history": {
    "recentMonths":       { "summary": "...", "updatedAt": "..." },
    "earlierContext":     { "summary": "...", "updatedAt": "..." },
    "longTermBackground": { "summary": "...", "updatedAt": "..." }
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

**facts.category 枚举**：`preference` / `knowledge` / `context` / `behavior` / `goal`

**facts.confidence 参考**：

| 范围 | 含义 |
|------|------|
| 0.9 ~ 1.0 | 用户明确陈述 |
| 0.7 ~ 0.8 | 对话中强烈暗示 |
| 0.5 ~ 0.6 | 推断模式（谨慎使用） |

---

## 整体调用链

```
MemoryMiddleware.after_agent()          # 中间件触发（agent 执行完毕后）
    └─ _filter_messages_for_memory()   # 过滤：保留用户输入 + 最终 AI 回答
    └─ get_memory_queue().add()        # 入队，防抖 30s

MemoryUpdateQueue._process_queue()     # 30s 后后台线程触发
    └─ MemoryUpdater.update_memory()   # LLM 提取 + 原子写入

下次对话：make_lead_agent()
    └─ _get_memory_context()           # 读取记忆（走缓存）
    └─ format_memory_for_injection()   # token 预算格式化
    └─ 注入 <memory> 到系统提示
```

---

## 配置项（config.yaml）

```yaml
memory:
  enabled: true                  # 总开关
  injection_enabled: true        # 是否注入系统提示
  max_injection_tokens: 2000     # 注入 token 上限
  debounce_seconds: 30           # 防抖等待时间（秒）
  max_facts: 100                 # 事实库最大条数
  fact_confidence_threshold: 0.7 # 写入事实的最低置信度
  model_name: null               # null = 继承全局模型
  storage_path: null             # null = 使用默认路径
```

---

## 架构师视角：设计总结

### 整体定位

**跨会话个性化层**。在无状态的 LLM 对话之上，叠加一个持久化的用户画像，让 agent 在每次新对话中"记得"用户是谁、在做什么、偏好什么。

---

### 核心架构决策

**1. 存储选型：JSON 文件而非数据库**

选择 `~/.deer-flow/memory.json` 平铺存储，而非 SQLite / 向量数据库。

- 优点：零依赖、可人工查看和编辑、部署简单
- 代价：并发写入安全依赖操作系统的 `rename()` 原子性，不支持多进程横向扩展
- 设计判断：单用户本地场景，并发压力不存在，此代价可接受

**2. 记忆结构：三层语义分层**

```
user (当前快照)      → 工作/个人/当下关注（频繁覆盖）
history (时间轴)     → 近月/较早/长期背景（渐少覆盖）
facts (离散事实库)   → 带置信度的具体知识点（增删管理）
```

这是一个**压缩-展开**设计：LLM 写入时将非结构化对话压缩为结构化摘要；注入时将结构化记忆展开为自然语言提示词。两端都由 LLM 驱动，人工不介入。

**3. 读写一致性：mtime 缓存 + 原子写入**

```
读：file.stat().mtime ≠ cache.mtime → 重新加载
写：write(tmp) → rename(tmp, target)   # OS 原子操作
写后：立即更新 cache.mtime             # 防止自己触发缓存失效
```

解决了"写后立刻读"的一致性问题，同时支持外部工具直接修改文件（如用户手动编辑记忆）后自动感知变更。

**4. 异步更新：防抖队列解耦写路径**

```
agent 执行路径（同步）
    └─ MemoryMiddleware.after_agent()
         └─ queue.add()  ← 立即返回，不阻塞用户响应

后台线程（异步，30s 防抖）
    └─ MemoryUpdater.update_memory()  ← LLM 调用（慢）
```

核心决策：**记忆更新不在关键路径上**。用户看到响应后，记忆才开始更新。防抖机制还处理了同一会话高频入队的场景（最新对话覆盖旧的，LLM 只调用一次）。

**5. 注入控制：token 预算 + 置信度优先级**

```
注入顺序：User Context → History → Facts（按置信度降序）
预算控制：tiktoken 精确计数，超出立即截止
兜底截断：整体超限时按字符比例截断 + "..."
```

高置信度事实永远优先进入上下文，低置信度事实在预算不足时被自然淘汰，不需要人工管理。

---

### 安全与健壮性

| 问题 | 解决方案 |
|------|---------|
| 上传文件路径污染长期记忆 | 写入前 regex 清洗 + LLM prompt 明确禁止 |
| LLM 返回格式错误 | `json.loads` 异常捕获，静默失败不崩溃 |
| 事实无限累积 | `max_facts=100`，按置信度降序截取 |
| 低质量事实写入 | `confidence_threshold=0.7` 过滤 |
| 内容重复 | 规范化（strip）后集合比对，跨批次去重 |
| 文件损坏 | 读取失败返回空结构，系统继续运行 |
| NaN/inf 置信度 | `_coerce_confidence()` 强制规范化，防止排序异常 |

---

### 架构层次图

```
┌─────────────────────────────────────────────┐
│  Lead Agent (系统提示注入层)                  │
│  format_memory_for_injection() → <memory>   │
└──────────────────┬──────────────────────────┘
                   │ 读（mtime 缓存）
┌──────────────────▼──────────────────────────┐
│  memory.json（持久化存储）                    │
│  user / history / facts                     │
└──────────────────▲──────────────────────────┘
                   │ 原子写入
┌──────────────────┴──────────────────────────┐
│  MemoryUpdater（LLM 驱动更新）               │
│  prompt → LLM → JSON → _apply_updates()     │
└──────────────────▲──────────────────────────┘
                   │ 30s 防抖
┌──────────────────┴──────────────────────────┐
│  MemoryUpdateQueue（异步解耦）               │
└──────────────────▲──────────────────────────┘
                   │ 非阻塞入队
┌──────────────────┴──────────────────────────┐
│  MemoryMiddleware（中间件触发）               │
│  _filter_messages_for_memory()              │
└─────────────────────────────────────────────┘
```

---

### 关键权衡与局限

| 权衡 | 当前选择 | 潜在问题 |
|------|---------|---------|
| 记忆更新时机 | 对话结束后异步 | 同一会话内无法读到本次更新 |
| 存储粒度 | 全量覆写 JSON | 并发写入（多线程/多进程）不安全 |
| 事实提取质量 | 完全依赖 LLM | 模型不稳定时提取质量波动 |
| per-agent 记忆 | 路径隔离 | agent 间记忆不共享，存在信息孤岛 |
| 记忆长度 | token 预算截断 | 长期用户的早期历史可能被挤出注入窗口 |

总体而言，这是一个**实用主义优先**的设计：在本地单用户场景下，用最小依赖和最低复杂度实现了可用的跨会话个性化，换取了横向扩展能力和实时一致性。

---

### 记忆分类机制：LLM 如何决定放哪里

分类完全由 **LLM 根据 `MEMORY_UPDATE_PROMPT` 的指令判断**，代码层不做任何分类逻辑。

**user —— 当前状态快照（易变，频繁覆盖）**

| 子区域 | 放什么 | 示例 |
|--------|--------|------|
| `workContext` | 现在的工作、活跃项目、主要技术栈 | "负责 DeerFlow 开源项目，主栈 Python+LangGraph" |
| `personalContext` | 语言、沟通偏好、工作之外的兴趣 | "中英双语，偏好简洁回答" |
| `topOfMind` | 近期同时在做的 3-5 个焦点 | "正在做记忆模块、调研 RAG 方案、跟踪 DeepSeek 动态" |

**history —— 时间轴（越久越稳定，按时间归档）**

| 子区域 | 时间范围 | 放什么 |
|--------|----------|--------|
| `recentMonths` | 近 1-3 个月 | 探索过的技术、解决过的问题、参与的项目 |
| `earlierContext` | 3-12 个月前 | 已完成的项目、形成的工作模式 |
| `longTermBackground` | 长期不变的背景 | 核心专长、长期兴趣、基础风格 |

**facts —— 离散知识点（精确可引用，可按 ID 删除）**

| category | 放什么 | 示例 |
|----------|--------|------|
| `preference` | 明确的喜好/厌恶 | "偏好 TypeScript" |
| `knowledge` | 已掌握的技术领域 | "熟悉 LangGraph" |
| `context` | 背景事实（职位、项目、地点） | "在某公司担任架构师" |
| `behavior` | 工作习惯、沟通方式 | "喜欢先看代码再提问" |
| `goal` | 明确表达的目标 | "计划今年掌握 Rust" |

**三类数据的本质区别：**

- **user** = "这个人现在是什么状态" → 会被频繁覆盖，旧内容直接替换
- **history** = "这个人做过什么" → 按时间归档，信息随时间沉淀到更深层
- **facts** = "关于这个人有哪些具体、可复用的知识点" → 独立条目，可精确增删

LLM 同时看到当前记忆 JSON 和新对话，对每个区域独立判断 `shouldUpdate=true/false`，只更新有新信息的部分，避免用空内容覆盖已有摘要。
