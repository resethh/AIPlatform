# DeerFlow Agent 执行架构分析

> 来源: [[raw/deerflow-agent执行.md]] | 分析日期: 2026-04-21

## 一句话定位

基于 LangChain `create_agent()` 工厂的多 Agent 系统：Lead Agent 处理用户请求，通过 `task` 工具委派给子 Agent 并发执行，13 层中间件链负责上下文、记忆、安全等横切面。

---

## 一、Agent 层次结构

```
┌─────────────────────────────────────────────────────┐
│                   Lead Agent                        │
│  make_lead_agent(config)                            │
│  主 agent，处理用户请求，可委派任务给子 agent             │
└──────────────┬──────────────────────────────────────┘
               │ 使用 task 工具
               ▼
        ┌──────────────────────────┐
        │     SubagentExecutor     │
        │  Scheduler Pool (3)      │
        │  Execution Pool (3)      │
        │  MAX_CONCURRENT = 3      │
        └──────┬──────────┬────────┘
               │          │
   ┌───────────▼──┐   ┌───▼──────────┐
   │  General-    │   │  Bash        │
   │  Purpose     │   │  Agent       │
   │  All tools   │   │  CLI tools   │
   │  Max 50 turns│   │  Max 30 turns│
   │  Timeout 900s│   │  Timeout 900s│
   └──────────────┘   └──────────────┘
```

**子 Agent 关键约束**：
- General-Purpose Agent 继承所有工具，但**禁止嵌套委派**（不能再调 `task`、`ask_clarification`、`present_files`）
- Bash Agent 仅限沙箱 CLI 工具
- 所有子 Agent 共享父 Agent 的 `sandbox_id`、`thread_id`、模型配置、`trace_id`

---

## 二、Agent 创建流程

```
make_lead_agent(config)
    ├── 解析运行时配置（model, thinking_enabled, subagent_enabled 等）
    ├── 组装工具（sandbox / MCP / builtin / task 工具）
    ├── 构建 13 层中间件链（见下节）
    ├── 生成系统提示（注入技能、记忆、子 agent 指令）
    └── langchain.agents.create_agent(
            model=chat_model,
            tools=tools,
            middleware=middlewares,
            system_prompt=prompt,
            state_schema=ThreadState,
        )
```

**运行时配置参数**（via `config.configurable`）：

| 参数 | 说明 |
|---|---|
| `thinking_enabled` | 启用模型扩展思考 |
| `model_name` | 选择特定 LLM |
| `reasoning_effort` | 指定推理强度 |
| `is_plan_mode` | 启用 TodoList 中间件 |
| `subagent_enabled` | 启用子 agent 委派 |
| `max_concurrent_subagents` | 最大并发子 agent 数（默认 3） |

---

## 三、13 层中间件链（执行顺序）

| # | 中间件 | 条件 | 作用 |
|---|---|---|---|
| 1 | ThreadDataMiddleware | 始终 | 创建线程隔离目录 |
| 2 | UploadsMiddleware | 始终 | 追踪并注入上传文件 |
| 3 | SandboxMiddleware | 始终 | 获取/管理沙箱生命周期 |
| 4 | DanglingToolCallMiddleware | 始终 | 修复缺失 ToolMessage（防 LLM 状态错乱） |
| 5 | GuardrailMiddleware | 可选 | 工具调用授权检查 |
| 6 | SummarizationMiddleware | 可选 | 接近 token 限制时压缩上下文 |
| 7 | TodoListMiddleware | `is_plan_mode=true` | 任务追踪 |
| 8 | TitleMiddleware | 始终 | 首次交互后自动生成标题 |
| 9 | MemoryMiddleware | 始终 | 队列异步内存更新（不阻塞关键路径） |
| 10 | ViewImageMiddleware | 按模型视觉支持 | 注入图像数据 |
| 11 | DeferredToolFilterMiddleware | 可选 | 隐藏延迟工具，节省 token |
| 12 | SubagentLimitMiddleware | `subagent_enabled=true` | 限制并发子 agent 数 |
| 13 | LoopDetectionMiddleware | 始终 | 检测并中断重复工具调用 |
| 14 | ClarificationMiddleware | 始终（最后） | 拦截澄清请求，中断执行返回用户 |

---

## 四、状态定义（ThreadState）

`ThreadState` 继承 `AgentState`，额外字段：

```python
class ThreadState(AgentState):
    sandbox: NotRequired[SandboxState | None]           # Sandbox ID 和状态
    thread_data: NotRequired[ThreadDataState | None]    # 线程隔离目录
    title: NotRequired[str | None]                      # 自动生成标题
    artifacts: Annotated[list[str], merge_artifacts]    # 工件列表（去重）
    todos: NotRequired[list | None]                     # TodoList 项
    uploaded_files: NotRequired[list[dict] | None]      # 已上传文件
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
```

---

## 五、Skill 系统（DeerFlow 实现）

**核心结论**：Skill 以**系统提示**形式注入，而非工具。Agent 按需 `read_file` 读取 SKILL.md 全文，完全基于 LLM 决策触发，无代码层面自动触发。

### 加载链

```
make_lead_agent(config)
    └─ apply_prompt_template()
        └─ get_skills_prompt_section()
            └─ load_skills(enabled_only=True)
                ├─ 扫描 skills/public/ 和 skills/custom/
                ├─ parse_skill_file() → Skill 对象
                └─ 检查 extensions_config.json 启用状态
            └─ 生成 <skill_system> XML 块注入系统提示
```

### 注入格式

```xml
<skill_system>
<available_skills>
    <skill>
        <name>data-analysis</name>
        <description>...</description>
        <location>/mnt/skills/public/data-analysis/SKILL.md</location>
    </skill>
</available_skills>
</skill_system>
```

### 渐进式加载（LLM 驱动）

```
用户请求 → Agent 匹配 skill 名称 → read_file("/mnt/skills/.../SKILL.md")
    → 读取工作流说明 → 按指令调用 bash / read_file 执行步骤
```

### Skill 与 Hermes 的差异

| 特性 | DeerFlow | Hermes |
|---|---|---|
| 注入方式 | 系统提示 XML（含文件路径） | 系统提示文本（`<available_skills>`） |
| 触发方式 | LLM 决策调 `read_file` | LLM 调 `skill_view()` |
| 沙箱挂载 | `/mnt/skills/` 只读挂载 | `~/.hermes/skills/` 本地路径 |
| 缓存机制 | 无（每次 `make_lead_agent` 重新加载） | 三层缓存（LRU + 磁盘快照 + 冷扫描） |
| 安全扫描 | 安装时验证（路径/名称/字段） | 创建+安装+`skill_view` 时三次扫描 |

---

## 六、整体分层架构

```
Application Layer (port 8001)
  /api/models, /api/mcp, /api/skills, /api/memory, /api/threads/...
            ↓ HTTP
LangGraph Server (port 2024)
  make_lead_agent()
  ├─ Lead Agent (ThreadState + 13+ Middlewares + Tools)
  │   └─ task tool → SubagentExecutor
  └─ SubagentExecutor (background)
      ├─ General-Purpose Agent
      └─ Bash Agent
            ↓
Harness Package (deerflow.*)
  agents/ + subagents/ + tools/ + sandbox/ + mcp/
```

---

## 亮点与局限

| 维度 | 亮点 | 局限 |
|---|---|---|
| 中间件架构 | 横切面关注点分离清晰，13 层各司其职 | 中间件顺序固定，深度定制需修改工厂函数 |
| 子 Agent 并发 | 线程池并发最多 3，天然防过载 | 并发上限低，复杂任务可能成为瓶颈 |
| 状态持久化 | LangGraph Checkpointer 原生支持断点恢复 | 依赖 LangGraph 生态，迁移成本高 |
| Skill 系统 | 无代码触发，纯 LLM 驱动，灵活 | 无缓存，无条件激活，无安全扫描 |

---

## 对中台设计的启示

- 参考点 [[architecture/deerflow.md]]：整体框架概述
- 参考点 [[flows/multi-agent.md]]：多 Agent 协调时序
- 参考点 [[skills/skill-iteration.md]]：ToB Skill 设计参考了 DeerFlow 的渐进加载思路
