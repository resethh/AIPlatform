# DeerFlow 上下文管理

DeerFlow 的上下文管理是一个多层联动系统，覆盖从单条消息到跨会话长期记忆的全生命周期。核心目标是：**在有限的 LLM 上下文窗口内，最大化信息密度和对话连贯性**。

---

## 一、设计思路

### 1.1 分层上下文模型

```
┌────────────────────────────────────────────────┐
│  系统提示（System Prompt）                        │
│  角色定义 + 记忆注入 + 技能目录 + 子 agent 规则   │
├────────────────────────────────────────────────┤
│  对话历史（Messages）                             │
│  当前会话内的全部轮次（可被 Summarization 压缩）    │
├────────────────────────────────────────────────┤
│  运行时注入（Runtime Injection）                   │
│  上传文件列表 + 图像 base64 + 工具调用修复          │
└────────────────────────────────────────────────┘
```

三层各司其职：系统提示提供长期个性化背景，对话历史承载当前会话推理链，运行时注入处理多模态和异常恢复。

### 1.2 核心设计原则

- **关键路径不阻塞**：记忆更新、token 统计等耗时操作全部异步，不影响用户响应延迟
- **渐进式加载**：技能、工具按需读取，初始系统提示只包含目录，避免一次性撑满上下文
- **原子一致性**：状态持久化（checkpointer）和记忆写入都使用原子操作，防止进程崩溃导致数据损坏
- **优雅降级**：任何单点失败（LLM 返回格式错误、文件读取失败）都返回空结构继续运行，而非崩溃

---

## 二、实现细节

### 2.1 系统提示的动态组装

**入口**：`lead_agent/prompt.py` → `apply_prompt_template()`

每次对话开始时，系统提示实时构建，不缓存，确保注入内容（记忆、技能）始终最新：

```
1. 角色定义      → "You are {agent_name}, an open-source super agent."
2. Agent Soul   → <soul> 可选个性化人格描述 </soul>
3. 记忆注入      → <memory> 用户画像摘要 + 高置信度事实 </memory>
4. 技能目录      → <skill_system> 可用技能列表 + 渐进加载说明 </skill_system>
5. 子 agent 规则 → <subagent_system> 并发上限 + 编排模式 </subagent_system>
6. 工作目录说明  → /mnt/user-data/{workspace,uploads,outputs} 路径映射
7. 响应风格规范  → 简洁、行动导向、语言与用户一致
8. 当前日期      → <current_date> 防止时间相关推理出错 </current_date>
```

**记忆注入细节**（`_get_memory_context()`）：

```python
memory_data = get_memory_data(agent_name)          # mtime 缓存读取
memory_content = format_memory_for_injection(
    memory_data,
    max_tokens=config.max_injection_tokens          # 默认 2000 token 上限
)
# 注入顺序：User Context → History → Facts（按置信度降序）
# 超出预算：逐条累加，超限立即截止，高置信度事实优先进入
```

**技能注入细节**（`get_skills_prompt_section()`）：

仅注入技能**目录**（名称 + 容器路径），不注入全文。LLM 在需要时通过 `read_file` 工具按需读取 `SKILL.md`，这是渐进式加载的核心机制。

---

### 2.2 对话历史的状态管理

**状态结构**：`thread_state.py` → `ThreadState`

```python
class ThreadState(AgentState):
    # 继承自 langchain AgentState，包含 messages 字段
    sandbox:        SandboxState | None         # 沙箱标识
    thread_data:    ThreadDataState | None       # 线程目录路径
    title:          str | None                   # 自动生成的会话标题
    artifacts:      list[str]                    # 产出文件 URI（去重）
    todos:          list | None                  # 任务清单（计划模式）
    uploaded_files: list[dict] | None            # 本轮上传文件元数据
    viewed_images:  dict[str, ViewedImageData]   # 已查看图像的 base64 缓存
```

**自定义 Reducer 策略**：

| Reducer | 逻辑 |
|---------|------|
| `merge_artifacts` | `dict.fromkeys()` 去重并保序 |
| `merge_viewed_images` | 新值为 `{}` 时清空（清理信号）；否则合并覆盖 |

**状态持久化**（LangGraph Checkpointer）：

```
请求到达 → checkpointer.get(thread_id) → 加载历史 ThreadState
             ↓
          agent 执行（追加新消息）
             ↓
          checkpointer.put(thread_id, updated_state) → 持久化
```

支持三种后端：

| 后端 | 适用场景 |
|------|---------|
| `InMemorySaver` | 开发/测试，进程重启后丢失 |
| `SqliteSaver` | 单机部署，文件持久化 |
| `PostgresSaver` | 生产环境，支持连接池 |

---

### 2.3 上下文压缩：SummarizationMiddleware

当对话历史过长时，自动压缩旧消息，防止超出模型上下文窗口。

**实现来源**：直接使用 LangChain 内置的 `langchain.agents.middleware.SummarizationMiddleware`，DeerFlow 在其之上封装了一层配置适配器（`_create_summarization_middleware()`），不覆盖核心压缩逻辑。

**配置字段**（`SummarizationConfig`）：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enabled` | bool | `false` | 总开关 |
| `model_name` | str \| null | `null` | 摘要模型，null = 继承全局默认模型 |
| `trigger` | ContextSize \| list | `null` | 触发阈值（OR 逻辑，满足任一即触发） |
| `keep` | ContextSize | `messages: 20` | 保留最近多少上下文 |
| `trim_tokens_to_summarize` | int \| null | `4000` | 送给摘要 LLM 前先截取到多少 token |
| `summary_prompt` | str \| null | `null` | 自定义摘要提示词，null = LangChain 默认 |

**ContextSize 三种类型**：

| type | value 含义 | 示例 |
|------|-----------|------|
| `tokens` | 绝对 token 数 | `{type: tokens, value: 15000}` |
| `messages` | 绝对消息条数 | `{type: messages, value: 50}` |
| `fraction` | 模型最大输入的百分比 | `{type: fraction, value: 0.8}` |

**触发条件**（满足任一即触发，OR 逻辑）：

```yaml
summarization:
  enabled: true
  trigger:
    - type: tokens
      value: 15000     # 超过 15000 token
    - type: messages
      value: 50        # 超过 50 条消息
    - type: fraction
      value: 0.8       # 超过模型最大输入的 80%
```

**配置适配器执行流程**（`_create_summarization_middleware()`）：

```
1. get_summarization_config()           → 加载配置
2. config.enabled == false → return None（不加入中间件链）
3. trigger → ContextSize.to_tuple() 转换
   单个：("tokens", 15000)
   多个：[("tokens", 15000), ("messages", 50)]
4. keep → ("messages", 20)
5. model_name 非空 → 直接传字符串给 LangChain
   model_name 为 null → create_chat_model(thinking_enabled=False) 创建实例
6. 按需追加 trim_tokens_to_summarize / summary_prompt
7. return SummarizationMiddleware(**kwargs)
```

**压缩执行流程**（LangChain 内部逻辑，每次 before_model 前触发）：

```
Step 1  统计当前消息列表的 token 数 / 条数
            ↓
Step 2  检查是否满足任一 trigger 阈值
            ↓ （未触发 → 跳过）
Step 3  按 keep 策略切分消息：
        [旧消息区] ←→ [保留区（最近 N 条/token）]
            ↓
Step 4  trim_tokens_to_summarize 截取旧消息区到上限
        （防止摘要请求本身超出模型窗口）
            ↓
Step 5  调用摘要 LLM，生成旧消息区的摘要文本
            ↓
Step 6  构造替换消息：
        HumanMessage("Here is a summary of the conversation to date:\n\n{summary}")
            ↓
Step 7  新对话历史 = [摘要消息] + [保留区消息原文]
```

**AI/Tool 消息对的完整性保护**：切分时确保 `AIMessage` 和其对应的 `ToolMessage` 始终在同一侧，不拆散工具调用对，防止 LLM 收到格式非法的消息链。

**压缩前后对比示意**：

```
压缩前（60 条消息，触发 messages≥50）：
  [Turn 1] User → AI → Tool → Tool Result
  [Turn 2] User → AI → Tool → Tool Result
  ...
  [Turn 50] User → AI （保留区开始）
  ...
  [Turn 60] User → AI（当前轮）

压缩后：
  [摘要] "User is working on a DeerFlow memory module.
          Explored summarization middleware, discussed..."
  [Turn 50] User → AI   ← 保留区原文，完整保留
  ...
  [Turn 60] User → AI（当前轮）
```

**中间件链中的位置**：排在 `DanglingToolCallMiddleware` 之后、`TodoListMiddleware` 之前，在沙箱就绪后尽早压缩，减少后续中间件处理的消息量。

---

### 2.4 运行时上下文注入（中间件链）

每次 LLM 调用前，中间件链按严格顺序执行，动态修改送入 LLM 的消息列表：

```
ThreadDataMiddleware     → 创建 per-thread 目录
UploadsMiddleware        → 在最后一条 HumanMessage 前插入 <uploaded_files> 块
SandboxMiddleware        → 获取沙箱，写入 state.sandbox
DanglingToolCallMiddleware → 修复无响应的 tool_call（注入 error ToolMessage）
GuardrailMiddleware      → 工具调用前鉴权（可选）
SummarizationMiddleware  → 上下文压缩（可选）
TodoListMiddleware       → 注入任务清单工具（计划模式，可选）
TitleMiddleware          → 首轮后自动生成会话标题
MemoryMiddleware         → 入队异步记忆更新（不修改 prompt）
ViewImageMiddleware      → 注入 base64 图像内容（视觉模型，可选）
SubagentLimitMiddleware  → 截断超额 task 工具调用（可选）
ClarificationMiddleware  → 拦截澄清请求，中断执行（必须最后）
```

**DanglingToolCallMiddleware 修复逻辑**（处理用户中断场景）：

```
扫描所有 AIMessage.tool_calls
    ↓
找出没有对应 ToolMessage 的 tool_call_id
    ↓
在该 AIMessage 紧随其后插入合成 ToolMessage：
{
  content: "[Tool call was interrupted and did not return a result.]",
  status: "error"
}
    ↓
LLM 收到完整合法的消息链，可正常继续
```

**ViewImageMiddleware 注入逻辑**：

```
检测：最后一条 AIMessage 含 view_image 工具调用 且 所有调用已完成
    ↓
构建 HumanMessage：
  - "Here are the images you've viewed:"
  - 每张图：路径文本 + data:mime;base64,... 图像块
    ↓
注入到消息列表末尾，触发 LLM 读取图像内容
注入后：viewed_images 置为 {} 清空缓存，防止重复注入
```

---

### 2.5 Token 预算管理

上下文各组成部分均有独立的 token 限额，形成层次化预算：

| 组成部分 | 预算控制 | 默认值 |
|---------|---------|--------|
| 记忆注入 | `max_injection_tokens` | 2000 |
| 压缩前裁剪 | `trim_tokens_to_summarize` | 4000 |
| 保留消息 | `keep.tokens` / `keep.messages` | 20 条 |
| 事实条数 | `max_facts` | 100 条 |
| 子 agent 并发 | `MAX_CONCURRENT_SUBAGENTS` | 3 |

**记忆注入的 token 计数**采用 `tiktoken`（cl100k_base 编码）精确计算，不可用时降级为 `len(text) // 4` 估算。

---

## 三、亮点

### 3.1 渐进式上下文加载

技能全文（可达数 KB）不在启动时注入，仅在 LLM 识别到相关查询时通过 `read_file` 工具按需加载。对比直接注入所有技能全文，可节省 80%+ 的系统提示 token 占用。

### 3.2 异步写路径不阻塞响应

记忆更新是成本最高的操作（需调用 LLM），但完全在后台线程执行：

```
用户收到响应  ←── agent 执行路径（同步，快）
                     └─ 入队（< 1ms）
30 秒后触发   ←── 后台线程（异步，慢）
                     └─ LLM 提取 + 写文件
```

用户零感知延迟。防抖机制还确保高频对话只触发一次 LLM 调用。

### 3.3 双重原子性保障

- **文件写入**：`write(tmp) → rename()` 操作系统级原子替换，进程崩溃不损坏文件
- **缓存一致性**：写入后立即更新 mtime 缓存，防止"写后读"触发误判重载

### 3.4 中断恢复机制

用户打断工具执行时，`DanglingToolCallMiddleware` 自动修复消息链，使 LLM 能优雅地从中断点恢复，而非因格式错误崩溃。这对长耗时工具调用（bash 命令、文件操作）尤为重要。

### 3.5 多模态上下文无缝融合

图像通过 base64 注入机制自然融入对话历史，与文本消息格式统一，不需要特殊的多模态 API 路径。注入后自动清理，避免同一图像在后续轮次重复占用 token。

### 3.6 分层记忆结构

对比简单的"全文追加"记忆方案，三层分类（user 快照 / history 时间轴 / facts 离散事实）让注入内容结构清晰、可裁剪、可精确删除，同时支持置信度排序——最可靠的信息优先进入上下文窗口。

---

## 四、企业级部署优化点

### 4.1 持久化后端选型

| 场景 | 推荐方案 | 说明 |
|------|---------|------|
| 单机开发 | `SqliteSaver` | 零配置，文件持久化 |
| 多实例部署 | `PostgresSaver` | 连接池，多进程共享状态 |
| 无状态容器 | `PostgresSaver` + 外部 PG | 容器重启不丢 session |
| 超大并发 | `PostgresSaver` + 连接池调优 | psycopg pool_size 按并发量配置 |

当前代码已支持 PostgreSQL，生产部署直接修改 `config.yaml`：

```yaml
checkpointer:
  type: postgres
  connection_string: "postgresql://user:pass@host:5432/deerflow"
```

### 4.2 上下文窗口参数调优

根据实际模型和业务场景调整压缩策略，避免过早压缩（丢失推理链）或过晚压缩（超出窗口报错）：

```yaml
summarization:
  enabled: true
  trigger:
    - type: fraction
      value: 0.75           # 调低到 0.75，给输出留更多空间
  keep:
    type: messages
    value: 30               # 保留更多近期消息，减少信息损失
  model_name: "gpt-4o-mini" # 用轻量模型做摘要，节省 70%+ 成本
  trim_tokens_to_summarize: 8000  # 大模型可适当调高
```

### 4.3 记忆系统的多租户隔离

当前 per-agent 记忆已提供路径隔离，企业多用户场景需进一步扩展：

- **用户级隔离**：将 `storage_path` 参数改为 `~/.deer-flow/{user_id}/memory.json`
- **组织级共享记忆**：多用户共享同一 `memory.json`，需在 `_save_memory_to_file` 加文件锁（`fcntl.flock`）
- **记忆访问审计**：在 `get_memory_data` / `update_memory` 写入操作日志，满足合规要求

### 4.4 上下文注入的安全边界

生产环境需防止恶意内容通过记忆注入影响 LLM 行为（Prompt Injection）：

- 对 `facts.content` 和摘要字段做长度上限（建议 512 字符/条）
- 在 `format_memory_for_injection` 输出前对特殊字符（`<`, `>`, `{`, `}`）做转义
- 审计记忆写入来源（`fact.source = thread_id`），异常 thread 触发的大量写入可告警

### 4.5 Summarization 模型成本优化

摘要模型和主对话模型分离，是降低企业运营成本的重要手段：

```yaml
summarization:
  model_name: "claude-haiku-4-5-20251001"  # 轻量模型做压缩
# 主对话仍使用 claude-opus-4-6 等高能力模型
```

记忆更新模型同理：

```yaml
memory:
  model_name: "claude-haiku-4-5-20251001"  # 记忆提取用轻量模型
```

两处优化叠加，可将辅助 LLM 调用成本降低 80-90%。

### 4.6 上下文监控与可观测性

生产部署建议在以下位置埋点：

| 监控点 | 位置 | 指标 |
|--------|------|------|
| Token 用量 | `TokenUsageMiddleware.after_model()` | input/output tokens，按 thread_id 分组 |
| 压缩触发频率 | `SummarizationMiddleware` 触发时 | 压缩前/后消息数，触发原因 |
| 记忆更新延迟 | `MemoryUpdater.update_memory()` 前后 | P99 延迟，成功/失败率 |
| 队列积压 | `MemoryUpdateQueue.pending_count` | 定期采样，防止记忆更新严重滞后 |
| 上下文注入大小 | `format_memory_for_injection()` 返回值 | token 数，判断记忆是否被频繁截断 |

### 4.7 冷启动与预热

大规模部署时，首次请求往往因冷初始化（checkpointer 连接、记忆文件加载）产生延迟峰值：

- **Checkpointer 预热**：服务启动时调用 `get_checkpointer()` 完成连接池初始化，而非等待首个请求
- **记忆预加载**：服务启动时调用 `get_memory_data()` 将常用 agent 的记忆装入缓存
- **MCP 工具懒初始化**：保持现有懒加载策略，避免启动时串行初始化所有 MCP 服务器拖慢就绪时间

---

## 五、上下文生命周期总览

```
新对话请求（thread_id）
    │
    ├─ checkpointer.get(thread_id)     # 加载历史 ThreadState
    │
    ├─ apply_prompt_template()         # 动态构建系统提示
    │      ├─ 记忆注入（mtime 缓存）
    │      ├─ 技能目录注入
    │      └─ 子 agent 规则注入
    │
    ├─ 中间件 before_model()
    │      ├─ UploadsMiddleware         → 注入上传文件列表
    │      ├─ DanglingToolCallMiddleware → 修复中断的工具调用
    │      ├─ SummarizationMiddleware   → 按需压缩历史消息
    │      └─ ViewImageMiddleware       → 注入图像 base64
    │
    ├─ LLM 调用（系统提示 + 历史消息 + 运行时注入）
    │
    ├─ 中间件 after_model() / after_agent()
    │      ├─ TokenUsageMiddleware      → 记录 token 用量
    │      ├─ TitleMiddleware           → 首轮生成标题
    │      └─ MemoryMiddleware          → 入队异步记忆更新
    │
    ├─ checkpointer.put(thread_id, state)  # 持久化更新后状态
    │
    └─ 返回响应给用户

30 秒后（后台线程）：
    └─ MemoryUpdateQueue._process_queue()
           └─ MemoryUpdater.update_memory()  # LLM 提取 → 原子写入
```
