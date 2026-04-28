# DeerFlow LangGraph Agent 架构分析
## Agent 架构总览

DeerFlow 使用 **LangChain 的高级 `create_agent()` 工厂函数**（而非低级的 LangGraph `StateGraph` API）来创建 agent。

---

## 核心 Agent 定义位置

### 1. Lead Agent（主 Agent）

**文件**: `backend/packages/harness/deerflow/agents/lead_agent/agent.py`

工厂函数 `make_lead_agent(config: RunnableConfig)` 是核心入口，注册在 `backend/langgraph.json` 中：

```json
{
  "graphs": {
    "lead_agent": "deerflow.agents:make_lead_agent"
  },
  "checkpointer": {
    "path": "./packages/harness/deerflow/agents/checkpointer/async_provider.py:make_checkpointer"
  }
}
```

**运行时配置参数**（via `config.configurable`）：

| 参数 | 说明 |
|------|------|
| `thinking_enabled` | 启用模型扩展思考 |
| `model_name` | 选择特定 LLM |
| `reasoning_effort` | 指定推理强度 |
| `is_plan_mode` | 启用 TodoList 中间件 |
| `subagent_enabled` | 启用子 agent 委派 |
| `max_concurrent_subagents` | 最大并发子 agent 数（默认 3） |
| `is_bootstrap` | bootstrap 模式标志 |
| `agent_name` | 自定义 agent 名称 |

---

### 2. 子 Agent 系统

**目录**: `backend/packages/harness/deerflow/subagents/`

| 文件 | 作用 |
|------|------|
| `executor.py` | 子 agent 执行引擎（线程池） |
| `builtins/general_purpose.py` | 通用多步骤任务 agent（max 50 turns） |
| `builtins/bash_agent.py` | 命令行专家 agent（max 30 turns） |
| `registry.py` | 子 agent 注册中心 |

**General-Purpose Agent** — 继承所有工具，禁止嵌套委派（`task`、`ask_clarification`、`present_files`）

**Bash Agent** — 仅限沙箱 CLI 工具（`bash`、`ls`、`read_file`、`write_file`、`str_replace`）

---

## 状态定义

**文件**: `backend/packages/harness/deerflow/agents/thread_state.py`

`ThreadState` 继承 `AgentState`，额外字段：

```python
class ThreadState(AgentState):
    sandbox: NotRequired[SandboxState | None]           # Sandbox ID 和状态
    thread_data: NotRequired[ThreadDataState | None]    # 线程隔离目录
    title: NotRequired[str | None]                       # 自动生成的线程标题
    artifacts: Annotated[list[str], merge_artifacts]    # 工件列表（去重）
    todos: NotRequired[list | None]                      # TodoList 项
    uploaded_files: NotRequired[list[dict] | None]      # 已上传文件
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
```

---

## Agent 创建流程

```
make_lead_agent(config)
    ├── 解析运行时配置（model, thinking, subagent_enabled 等）
    ├── 组装工具（sandbox / MCP / builtin / task 工具）
    ├── 构建 13 层中间件链
    ├── 生成系统提示（注入技能、内存、子 agent 指令）
    └── langchain.agents.create_agent(
            model=chat_model,
            tools=tools,
            middleware=middlewares,
            system_prompt=prompt,
            state_schema=ThreadState,
        )
```

**子 Agent 创建流程**：

```
task_tool(description, prompt, subagent_type)
    ├── 从 registry 获取 SubagentConfig
    ├── 应用 config.yaml 超时覆盖
    ├── 创建 SubagentExecutor（传递父 sandbox / thread_data / model）
    └── executor.execute() 或 executor.execute_async()
            ├── 内部创建 Agent (create_agent)
            ├── 流式执行获取 AI 消息
            └── 返回最终文本结果给父 Agent
```

---

## Agent 间关系

```
┌─────────────────────────────────────────────────────┐
│                   Lead Agent                        │
│  (make_lead_agent)                                  │
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
   │  Agent       │   │              │
   │  All tools   │   │  CLI tools   │
   │  Max 50 turns│   │  Max 30 turns│
   │  Timeout 900s│   │  Timeout 900s│
   └──────────────┘   └──────────────┘

所有子 agent 共享：
  - 父 agent 的沙箱实例 (sandbox_id)
  - 线程隔离目录 (thread_id)
  - 继承的模型配置
  - 分布式追踪 ID (trace_id)
```

---

## 中间件链（执行顺序）

`lead_agent/agent.py` 中 `_build_middlewares()` 按以下顺序组装 **13 层中间件**：

| # | 中间件 | 条件 | 作用 |
|---|--------|------|------|
| 1 | ThreadDataMiddleware | 始终 | 创建线程隔离目录 |
| 2 | UploadsMiddleware | 始终 | 追踪并注入上传文件 |
| 3 | SandboxMiddleware | 始终 | 获取/管理沙箱生命周期 |
| 4 | DanglingToolCallMiddleware | 始终 | 修复缺失 ToolMessage |
| 5 | GuardrailMiddleware | 可选 | 工具调用授权检查 |
| 6 | SummarizationMiddleware | 可选 | 接近 token 限制时上下文压缩 |
| 7 | TodoListMiddleware | `is_plan_mode=true` | 任务追踪 |
| 8 | TitleMiddleware | 始终 | 首次交互后自动生成标题 |
| 9 | MemoryMiddleware | 始终 | 队列异步内存更新 |
| 10 | ViewImageMiddleware | 按模型视觉支持 | 注入图像数据 |
| 11 | DeferredToolFilterMiddleware | 可选 | 隐藏延迟工具 |
| 12 | SubagentLimitMiddleware | `subagent_enabled=true` | 限制并发子 agent 数 |
| 13 | LoopDetectionMiddleware | 始终 | 检测并中断重复工具调用 |
| 14 | ClarificationMiddleware | 始终（最后） | 拦截澄清请求，中断执行 |

---

## 工具系统

**文件**: `backend/packages/harness/deerflow/tools/__init__.py`

核心函数 `get_available_tools(groups, model_name, subagent_enabled)` 动态组装：

| 工具类别 | 说明 |
|---------|------|
| 配置定义的工具 | 从 `config.yaml` 解析 |
| MCP 工具 | 从 `extensions_config.json` 的 MCP 服务器 |
| `present_files` | 使输出文件对用户可见 |
| `ask_clarification` | 请求澄清（被 ClarificationMiddleware 拦截） |
| `view_image` | 读取图像为 base64（仅模型支持视觉时） |
| `task` | 委派给子 agent（`subagent_enabled=true` 时） |
| 沙箱工具 | `bash`、`ls`、`read_file`、`write_file`、`str_replace` |

---

## 整体分层架构

```
┌───────────────────────────────────────┐
│       Application Layer               │
│  Gateway API (port 8001)              │
│  /api/models, /api/mcp, /api/skills   │
│  /api/memory, /api/threads/{id}/...   │
└───────────┬───────────────────────────┘
            │ HTTP Client
            ▼
┌───────────────────────────────────────┐
│    LangGraph Server (port 2024)       │
│  make_lead_agent()                    │
│                                       │
│  ┌─ Lead Agent                        │
│  │  ├─ ThreadState                    │
│  │  ├─ 13+ Middlewares                │
│  │  ├─ Tools (sandbox, MCP, etc.)     │
│  │  └─ Task Tool → SubagentExecutor   │
│  │                                    │
│  └─ SubagentExecutor (background)     │
│     ├─ General-Purpose Agent          │
│     └─ Bash Agent                     │
│                                       │
│  Checkpointer: async_provider.py      │
└───────────┬───────────────────────────┘
            │
            ▼
┌───────────────────────────────────────┐
│     Harness Package (deerflow.*)      │
│                                       │
│  agents/                              │
│  ├─ lead_agent/agent.py               │
│  ├─ middlewares/ (13 文件)             │
│  └─ thread_state.py                   │
│                                       │
│  subagents/                           │
│  ├─ executor.py                       │
│  ├─ builtins/ (general_purpose, bash) │
│  └─ registry.py                       │
│                                       │
│  tools/                               │
│  ├─ __init__.py                       │
│  └─ builtins/task_tool.py             │
│                                       │
│  sandbox/ + mcp/ + models/ + config/  │
└───────────────────────────────────────┘
```

---

## 关键文件清单

### Agent 定义
- `backend/packages/harness/deerflow/agents/lead_agent/agent.py` — Lead agent 工厂
- `backend/packages/harness/deerflow/agents/thread_state.py` — 状态定义
- `backend/packages/harness/deerflow/subagents/executor.py` — 子 agent 执行引擎
- `backend/packages/harness/deerflow/subagents/builtins/general_purpose.py`
- `backend/packages/harness/deerflow/subagents/builtins/bash_agent.py`
- `backend/packages/harness/deerflow/subagents/registry.py`

### 中间件（13 个文件）
- `backend/packages/harness/deerflow/agents/middlewares/`
  - `clarification_middleware.py`
  - `loop_detection_middleware.py`
  - `memory_middleware.py`
  - `subagent_limit_middleware.py`
  - `thread_data_middleware.py`
  - `title_middleware.py`
  - `todo_middleware.py`
  - `token_usage_middleware.py`
  - `tool_error_handling_middleware.py`
  - `uploads_middleware.py`
  - `view_image_middleware.py`
  - `dangling_tool_call_middleware.py`
  - `deferred_tool_filter_middleware.py`

### 工具系统
- `backend/packages/harness/deerflow/tools/__init__.py`
- `backend/packages/harness/deerflow/tools/builtins/task_tool.py`

### 配置
- `backend/langgraph.json` — LangGraph 服务器配置
- `config.yaml` — 应用配置（子 agent 超时、内存、上下文压缩等）

### Skill 系统
- `backend/packages/harness/deerflow/agents/lead_agent/prompt.py` — 系统提示生成（skill 注入入口）
- `backend/packages/harness/deerflow/skills/loader.py` — Skill 加载器
- `backend/packages/harness/deerflow/skills/parser.py` — YAML frontmatter 解析
- `backend/packages/harness/deerflow/skills/types.py` — Skill 数据类定义
- `backend/packages/harness/deerflow/skills/validation.py` — Skill 元数据验证
- `backend/packages/harness/deerflow/skills/installer.py` — 自定义 skill 安装
- `backend/packages/harness/deerflow/config/extensions_config.py` — 启用状态管理
- `backend/app/gateway/routers/skills.py` — Skills REST API

---

## Skill 系统：加载到 Agent 的机制

**核心结论：Skill 以系统提示（System Prompt）形式注入，而不是工具。**

---

### Skill 定义和存储

```
deer-flow/skills/
├── public/          # 内置 skill（纳入版本管理）
│   ├── data-analysis/SKILL.md
│   ├── deep-research/SKILL.md
│   └── ...
└── custom/          # 用户自定义（gitignore，不受版本管理）
    └── ...
```

每个 skill 是一个目录，包含 `SKILL.md`，格式为 YAML frontmatter + Markdown 正文：

```yaml
---
name: data-analysis
description: Use this skill when the user uploads Excel/CSV files...
license: MIT          # 可选
version: 1.0.0        # 可选
allowed-tools:        # 可选
  - bash
  - read_file
---
# Skill 实现说明（正文内容由 agent 按需读取）
```

| 字段 | 必需 | 说明 |
|------|------|------|
| `name` | 是 | hyphen-case，最长 64 字符 |
| `description` | 是 | 最多 1024 字符，不含 `<>` |
| `license` | 否 | 许可证信息 |
| `version` | 否 | 版本号 |
| `allowed-tools` | 否 | 允许使用的工具列表 |

---

### 完整加载链

```
make_lead_agent(config)
    └─ apply_prompt_template()                   # prompt.py
        └─ get_skills_prompt_section()
            └─ load_skills(enabled_only=True)    # loader.py
                ├─ 扫描 skills/public/ 和 skills/custom/
                ├─ parse_skill_file()             # 读取 YAML frontmatter → Skill 对象
                └─ 检查 extensions_config.json 中的启用状态
            └─ 生成 <skill_system>...</> XML 块
        └─ 注入到 SYSTEM_PROMPT_TEMPLATE
    └─ 返回完整系统提示给 create_agent()
```

---

### 注入到系统提示的内容

`get_skills_prompt_section()` 生成的 XML 块被插入系统提示的 `<skill_system>` 区域：

```xml
<skill_system>
You have access to skills that provide optimized workflows for specific tasks.

**Progressive Loading Pattern:**
1. When a user query matches a skill's use case, immediately call `read_file`
   on the skill's main file using the path in the <location> tag
2. Read and understand the skill's workflow and instructions
3. Follow the skill's instructions precisely

**Skills are located at:** /mnt/skills

<available_skills>
    <skill>
        <name>data-analysis</name>
        <description>Use this skill when the user uploads Excel/CSV files...</description>
        <location>/mnt/skills/public/data-analysis/SKILL.md</location>
    </skill>
    <skill>
        <name>deep-research</name>
        <description>Comprehensive research skill for in-depth investigations...</description>
        <location>/mnt/skills/public/deep-research/SKILL.md</location>
    </skill>
</available_skills>
</skill_system>
```

系统提示整体结构（按注入顺序）：

| 区域 | 内容 |
|------|------|
| `<role>` | Agent 角色定义 |
| `<soul>` | Agent 个性（来自 SOUL.md，可选） |
| `<memory>` | 内存上下文（如果启用） |
| `<skill_system>` | **Skill 列表和使用指南** |
| `<available-deferred-tools>` | 延迟工具列表（tool_search） |
| `<subagent_system>` | 子 agent 指令（如果启用） |
| `<current_date>` | 当前日期 |

---

### Agent 使用 Skill 的方式（Progressive Loading）

Skill 不预先加载内容，Agent 按需读取：

```
用户："分析我上传的 Excel 文件"
    ↓
Agent 在系统提示中找到匹配的 skill：data-analysis
    ↓
Agent 调用 read_file("/mnt/skills/public/data-analysis/SKILL.md")
    ↓
Agent 读取 skill 正文中的工作流说明
    ↓
Agent 按 skill 指令依次调用工具（bash、read_file 等）执行任务
```

---

## Skill 渐进式加载的实现原理

### 核心结论

渐进式加载**完全基于 LLM 决策**，无代码层面的自动触发。代码只负责：
1. 把 skill 元数据注入系统提示（告知 LLM "有什么可用"）
2. 提供 `read_file` 工具并处理路径转换（让 LLM "能读取"）
3. 对 `/mnt/skills/` 路径强制只读（安全保障）

---

### `/mnt/skills` 的挂载机制

**本地沙箱（LocalSandboxProvider）**：无需挂载，直接通过路径映射访问主机文件系统。

**Docker 沙箱（AioSandboxProvider）**：启动容器时通过 `-v` 参数挂载：

```python
# community/aio_sandbox/aio_sandbox_provider.py
def _get_skills_mount(self):
    host_path = get_skills_host_path()        # 主机上的 skills/ 目录
    container_path = get_skills_container_path()  # /mnt/skills
    return (host_path, container_path, True)  # True = read_only

# 生成的 Docker 命令
docker run -v /host/deer-flow/skills:/mnt/skills:ro ...
```

两种沙箱对 agent 暴露的路径完全一致，差异对上层透明。

---

### `read_file` 工具的路径转换

**文件**：`sandbox/tools.py`

Agent 使用虚拟路径，工具内部自动转换为真实路径：

```
Agent 调用:  read_file("/mnt/skills/public/data-analysis/SKILL.md")
                ↓
validate_local_tool_path()  →  检查路径合法性，/mnt/skills/ 强制只读
                ↓
_is_skills_path()  →  true
                ↓
_resolve_skills_path()
  /mnt/skills/public/data-analysis/SKILL.md
    → {host_skills_root}/public/data-analysis/SKILL.md
                ↓
LocalSandbox.read_file(real_path)  →  open(path).read()
```

安全约束（`validate_local_tool_path()`）：
- `/mnt/skills/` 路径：只读，拒绝写入
- `/mnt/user-data/` 路径：读写均可
- 其他路径：拒绝访问
- 拒绝 `..` 目录遍历

---

### SKILL.md 文件结构与执行方式

Skill 目录的典型结构：

```
skills/public/data-analysis/
├── SKILL.md                   # 主文件（agent 首先读取）
└── scripts/
    ├── analyze.py             # 被 SKILL.md 中的命令引用
    └── requirements.txt

skills/public/bootstrap/
├── SKILL.md
├── references/
│   └── conversation-guide.md  # SKILL.md 中要求按需读取
└── templates/
    └── SOUL.template.md
```

SKILL.md 正文直接嵌入可执行命令和子文件引用，agent 读取后按文本指令行动：

```markdown
## Step 2: Inspect File Structure

python /mnt/skills/public/data-analysis/scripts/analyze.py \
  --files /mnt/user-data/uploads/data.xlsx \
  --action inspect
```

```markdown
## Before your first response, read both:
1. `references/conversation-guide.md`
2. `templates/SOUL.template.md`
```

Agent 看到这些指令后会依次调用 `bash` 或 `read_file` 执行，这就是"渐进式"的含义——**按需、逐步**加载所需内容。

---

### 完整执行链（以 data-analysis 为例）

```
用户上传 sales_2024.xlsx，提问"帮我分析这个文件"
    ↓
Agent 系统提示中有 <skill_system>，识别到 data-analysis 匹配
    ↓
Agent 调用 read_file("/mnt/skills/public/data-analysis/SKILL.md")
    │   sandbox/tools.py: 路径转换 → open(real_path)
    └── 返回 SKILL.md 全文
    ↓
Agent 解析工作流：Step1(inspect) → Step2(query) → Step3(export)
    ↓
Agent 调用 bash("python /mnt/skills/.../analyze.py --action inspect ...")
    │   sandbox/tools.py: 替换虚拟路径 → 真实路径
    │   LocalSandbox.execute_command() → subprocess.run()
    └── 返回文件结构信息
    ↓
Agent 继续执行后续步骤（按需加载 scripts/ 中的其他脚本）
    ↓
Agent 生成最终分析结果返回给用户
```

---

### 与 Tool Search（DeferredToolFilterMiddleware）的对比

两者都是"按需加载"，但作用对象不同：

| 维度 | Skill Progressive Loading | Tool Search |
|------|--------------------------|-------------|
| 加载对象 | Skill 工作流文档（SKILL.md） | MCP 工具的 schema/签名 |
| 触发方式 | Agent 调用 `read_file` | Agent 调用 `tool_search` |
| 存储位置 | 文件系统（`/mnt/skills/`） | `DeferredToolRegistry`（内存） |
| 目的 | 提供工作流指导 | 节省 token，运行时发现工具 |
| Agent 初始可见 | 名称 + 描述 + 文件路径 | 仅工具名称列表 |

`DeferredToolFilterMiddleware` 在 `model.bind_tools()` 前过滤掉 deferred 工具的完整 schema，只保留名称放在 `<available-deferred-tools>` 块中，与 skill 系统无直接关联。

---

### 关键代码文件（渐进式加载相关）

| 功能 | 文件 | 关键函数 |
|------|------|---------|
| 路径转换 | `sandbox/tools.py` | `_resolve_skills_path()` |
| 路径安全验证 | `sandbox/tools.py` | `validate_local_tool_path()` |
| read_file 工具 | `sandbox/tools.py` | `read_file_tool()` |
| bash 路径替换 | `sandbox/tools.py` | `replace_virtual_paths_in_command()` |
| LocalSandbox 读取 | `sandbox/local/local_sandbox.py` | `read_file()` |
| Docker 挂载配置 | `community/aio_sandbox/aio_sandbox_provider.py` | `_get_skills_mount()` |
| 系统提示注入指导 | `agents/lead_agent/prompt.py` | `get_skills_prompt_section()` |

---

### 启用状态管理

`extensions_config.json` 控制每个 skill 的启用状态：

```json
{
  "skills": {
    "data-analysis": { "enabled": true },
    "deep-research": { "enabled": false }
  }
}
```

通过 `PUT /api/skills/{skill_name}` 修改，下次 `make_lead_agent()` 调用时生效。

---

### 内置 Skill vs 自定义 Skill

| 特性 | 内置 Skill | 自定义 Skill |
|------|-----------|-------------|
| 存储位置 | `skills/public/` | `skills/custom/` |
| 版本管理 | Git 管理 | gitignore |
| 安装方式 | 随项目部署 | `POST /api/skills/install`（上传 `.skill` ZIP） |
| 启用状态 | `extensions_config.json` | 同左 |

自定义 skill 安装流程（`installer.py`）：

```
上传 .skill 文件（ZIP 格式）
    ├─ 安全验证（拒绝绝对路径、目录遍历、符号链接、zip bomb）
    ├─ 定位并解析 SKILL.md
    ├─ 验证 name / description 字段
    ├─ 检查名称冲突
    └─ 复制到 skills/custom/{skill_name}/
```

---

### Skills REST API

| 方法 | 端点 | 功能 |
|------|------|------|
| GET | `/api/skills` | 列出所有 skill（含启用状态） |
| GET | `/api/skills/{name}` | 获取单个 skill 详情 |
| PUT | `/api/skills/{name}` | 更新启用状态 |
| POST | `/api/skills/install` | 从 `.skill` 存档安装自定义 skill |
