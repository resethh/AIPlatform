# Hermes 记忆系统

> 来源: [[raw/hermes-记忆.md]] | 分析日期: 2026-04-22

## 一句话定位
插件化多层记忆架构：BuiltinProvider 管理本地 MEMORY.md/USER.md（硬字符上限），Session FTS5 作为无限量历史档案层，可挂载 Honcho/Mem0 等 8 个外部 Provider；核心设计是"冻结快照"保持 LLM 前缀缓存稳定。

---

## 架构分层

```
MemoryManager（中央协调器）
    ├── BuiltinMemoryProvider  → MEMORY.md / USER.md（本地文件，字符硬限）
    ├── Session Search         → SQLite FTS5（历史对话全文索引，按需召回）
    └── ExternalMemoryProvider → 最多 1 个（Honcho / Mem0 / Holographic 等 8 个插件可选）
              ↑
    MemoryProvider (ABC)  统一插件接口，标准化生命周期
```

---

## 内置 Provider（BuiltinMemoryProvider）

### 数据结构

| 文件 | 内容 | 字符上限 |
|---|---|---|
| `~/.hermes/memories/<profile>/MEMORY.md` | Agent 笔记、事实、偏好 | 2200 字符（~800 tokens） |
| `~/.hermes/memories/<profile>/USER.md` | 用户画像：角色、习惯、偏好 | 1375 字符（~500 tokens） |

- 条目分隔符：`§`（段落符），支持多行
- 基于字符计数而非 Token 计数（与模型无关，跨 LLM 通用）

### 记忆操作

| 操作 | 行为 |
|---|---|
| `add` | 追加条目；超字符预算则拒绝；精确匹配去重 |
| `replace` | 子字符串定位旧条目，替换，重新计算预算 |
| `remove` | 子字符串定位，删除，重新计算预算 |
| `read` | 返回**冻结快照**，不返回实时状态 |

### 冻结快照机制

```
Session Start → 从 MEMORY.md/USER.md 加载 → 构建系统 Prompt → 冻结快照
整个 Session 期间：系统 Prompt 不变 → LLM 前缀缓存命中
Agent 中途写入：磁盘更新，但快照不刷新
Next Session Start：重新加载最新内容 → 新快照
```

**设计动机**：LLM Prefix Cache 要求前缀字节不变才能命中。如果每轮更新系统提示，缓存失效，每轮都要重新编码整个 Prompt，成本极高。冻结牺牲"本次 Session 立即生效"换取全 Session 的缓存命中。

**代价**：本次 Session 新写入的记忆要到下次 Session 才可见。

### 写入安全扫描

记忆写入前必须通过注入检测，拦截以下模式：

- 不可见 Unicode 字符（零宽字符等）
- Prompt 注入模式
- Shell 后门 / SSH 访问尝试
- 通过 curl/wget 的凭证窃取模式

### 原子写入与文件锁

```python
# 跨平台文件锁
fcntl.flock()     # Unix
msvcrt.locking()  # Windows

# 写临时文件 → os.replace() 原子覆盖目标
```

进程崩溃也不会产生部分写入的损坏文件；同时支持多进程并发安全写入。

---

## Session Search（FTS5 全文索引层）

### 设计定位

MEMORY.md 是"总是可见的提炼事实"（2200 字符硬限），Session Search 是"按需回忆的历史对话档案"（无限量）。两层职责分离，解决有界记忆与无限历史之间的矛盾。

```
MEMORY.md      → 语义压缩后的关键知识，每次 Session 全量注入（~1300 tokens 合计）
Session FTS5   → 所有历史对话的原始全文，平时不占 token，Agent 调工具时才触发
```

### 实现机制

- **存储**：SQLite 内置 FTS5 扩展，对所有历史 Session 对话建倒排索引
- **触发**：Agent 主动调用 `session_search_tool`，系统提示里包含使用场景说明
- **后处理**：搜索命中后用 Gemini Flash 对结果做摘要再注入，防止原始对话文本直接注入导致 token 爆炸

### 选用 FTS5 的原因

| 备选方案 | 为什么没选 |
|---|---|
| 全量注入历史 | 历史 Session 多后 token 爆炸 |
| 向量搜索 | 需要嵌入模型 + 向量 DB，违背本地零依赖原则 |
| SQL LIKE | Session 多了全表扫描太慢 |
| **FTS5** | SQLite 内置，Python 标准库直接用，倒排索引毫秒级，BM25 排序开箱即得，语义质量不足由 LLM 摘要补偿 |

### MEMORY.md 硬限的深层意图

2200 字符上限不是技术约束，而是**刻意的设计压力**：记忆满了 Agent 必须合并/替换旧条目，强制 LLM 做"什么值得保留"的价值判断，防止记忆无限堆积退化为噪音。

---

## 外部 Provider 体系

### 单外部 Provider 限制

MemoryManager 最多允许挂载 1 个外部 Provider，第二个注册时直接拒绝。

**原因**：防止工具 Schema 膨胀，避免多后端数据一致性问题，保持 LLM 工具调用列表简洁。

### 8 个可选 Provider

| Provider | 核心能力 | 更新机制 |
|---|---|---|
| **Honcho** | 辩证推理用户建模，跨 Session 上下文注入 | `dialecticCadence` 控制 LLM reasoning 刷新频率 |
| **OpenViking** (ByteDance) | 分层知识树，分级检索 L0→L1→L2 | Session commit 时自动提取 6 类记忆 |
| **Mem0** | 服务端 LLM 事实提取，语义搜索 + reranking | 自动去重，服务端管理 |
| **Hindsight** | 知识图谱，实体解析，多策略检索 | `hindsight_reflect` 做跨记忆合成 |
| **Holographic** | 本地 SQLite + FTS5，Trust Scoring，HRR 代数查询 | 非对称反馈（+0.05 / −0.10），矛盾自动检测 |
| **RetainDB** | 混合搜索（Vector+BM25+Reranking），7 种记忆类型 | Delta 压缩，importance 分级 |
| **ByteRover** | — | — |
| **Supermemory** | — | — |

### Honcho 三层记忆（最完整的外部 Provider）

```
Layer 1: Context（上下文表示）
├── 按 contextCadence 刷新（默认每轮）
└── 输出：用户表示 + 用户 Peer Card

Layer 2: Dialectic Supplement（辩证推理补充）
├── 按 dialecticCadence 刷新（默认每 3 轮）
├── 冷启动："这个人是谁？" → 自审空白 → 调和矛盾（1-3 Pass）
└── 提前退出：上一 Pass 有实质结论时停止，避免过度消耗

Layer 3: Session Messages（会话消息）
└── 异步同步，按 25k 字符分块上传
```

**三种召回模式**：`context`（仅自动注入）/ `tools`（仅工具调用）/ `hybrid`（两者均可）

---

## 系统 Prompt 组装顺序

```
1. Agent 身份（SOUL.md 或默认身份）
2. 记忆操作指南（memory 工具用法）
3. 跨会话搜索指南（session_search_tool 使用场景）
4. Skills 索引
5. 记忆块
   ├── MEMORY.md 内容（含字符预算进度）
   ├── USER.md 内容
   └── 外部 Provider 召回内容（如 Honcho）
6. 上下文文件（.hermes.md、AGENTS.md 等）
7. 平台提示（Telegram/Discord/SMS 特殊指令）
8. 环境提示（WSL/Termux/容器检测）
9. 工具使用强制指南（GPT/Gemini 适配）
```

记忆内容注入时用 `<memory-context>` XML 标签隔离，并附系统说明"这是背景信息而非用户新输入"，防止 LLM 混淆。

---

## 完整 Session 生命周期

```
初始化    → 注册 Provider → 构建系统 Prompt（一次性冻结）
每轮开始  → prefetch_all(query)：返回缓存结果，同时后台预取下一轮
每轮结束  → sync_all()：后台线程持久化，主线程不等待
触发压缩  → on_pre_compress()：压缩前提取关键信息保存到记忆
Session 结束 → on_session_end()：触发 extraction，flush 队列，关闭连接
```

---

## 局限性

| 问题 | 现状 |
|---|---|
| 冻结快照延迟 | 本次 Session 新增记忆下次才可见 |
| 字符上限粗糙 | 按字符计数，不按语义重要度，容量满时合并质量依赖 LLM |
| 单外部 Provider | 无法同时使用语义搜索（Mem0）+ 辩证建模（Honcho） |
| 本地单用户 | 无多租户隔离，不适合企业多用户部署 |
| Session Search 无语义 | FTS5 是关键词匹配，语义相近但措辞不同的历史无法召回 |

---

## 对中台设计的启示

- 采用了**冻结快照**设计保护 Prefix Cache，参考点: [[memory/memory-management.md]]
- **Session Search 分层思路**值得借鉴：热记忆（全量注入）+ 冷档案（按需 FTS 召回），中台可用 Redis + PostgreSQL 实现同等分层
- **字符硬限→有界记忆**的设计哲学：强迫质量判断，中台改用 confidence 分 + max_facts 实现更精细的有界控制
- **单外部 Provider 限制**在中台应改为多 Provider 组合路由，参考: [[memory/memory-management.md]]

---

## 上下游依赖

- 上游: [[architecture/hermes.md]]（MemoryManager 由 prompt_builder 在组装系统提示时调用）
- 下游: [[context/context-management.md]]（记忆注入 System Prompt 的交汇点）
- 对比: [[memory/deerflow-memory.md]]（冻结快照 vs DeerFlow 动态 JSON 的权衡）
- 中台设计: [[memory/memory-management.md]]
