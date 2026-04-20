# Skill 创建、优化、评估完整说明

---

## 一、Skill 创建

### 触发时机

由 LLM 自主判断，满足以下条件时主动创建：
- 完成复杂任务（5+ 工具调用）
- 克服了困难的报错
- 发现了非显而易见的工作流程
- 用户明确要求"记住这个做法"

### 创建流程

```
skill_manage(action="create", name="xxx", content="...")
    ├── 验证名称（小写、连字符、最多64字符）
    ├── 验证 YAML frontmatter（name/description 必填）
    ├── 检查重名（跨所有目录）
    ├── 创建目录 ~/.hermes/skills/{category}/{name}/
    ├── 原子写入 SKILL.md（tmpfile + os.replace）
    └── 安全扫描 → 通不过则回滚删除
```

### SKILL.md 格式

```yaml
---
name: my-skill
description: 一句话描述
platforms: [macos, linux]
prerequisites:
  commands: [curl, jq]
  env_vars: [API_KEY]
metadata:
  hermes:
    tags: [tag1]
    requires_toolsets: []       # 需要特定工具集才激活
    fallback_for_tools: [xxx]   # 当某工具不可用时才激活
---

## 触发条件
## 步骤
## 注意事项
```

### 目录结构

```
~/.hermes/skills/
├── my-skill/
│   ├── SKILL.md                 # 主内容（必需）
│   ├── references/              # 参考文档
│   ├── templates/               # 模板文件
│   ├── scripts/                 # 可执行脚本
│   └── assets/                  # 其他资源
└── category-name/
    └── another-skill/
        └── SKILL.md
```

---

## 二、Skill 加载到 Agent

### 加载到 System Prompt

```
build_skills_system_prompt()
    ├── 第1层：进程内 LRU 缓存（最快）
    ├── 第2层：磁盘快照 ~/.skills_prompt_snapshot.json（验证 mtime/size）
    └── 第3层：冷启动全量扫描 → 写入快照

过滤规则：
    ├── 平台兼容性（platforms 字段）
    ├── 禁用列表（config.yaml skills.disabled）
    └── 条件激活（fallback_for_tools / requires_toolsets）

最终注入 system prompt：
    "可用 skill：
     • mlops/axolotl: Fine-tune LLMs
     • base: Query Base blockchain..."
```

### 使用时（/slash 命令 or skill_view）

```
用户输入 /axolotl 微调 Llama
    └── skill_view("axolotl")
        ├── 定位 SKILL.md
        ├── 检查平台/环境变量
        ├── 扫描提示注入安全检查
        ├── 收集 references/ templates/ scripts/
        └── 返回完整内容，注入激活提示：
            "[SYSTEM: 用户已调用 axolotl skill，请按说明操作...]"
```

---

## 三、Skill 优化

**核心设计思路：** Agent 在使用 skill 过程中发现问题，**立即自主修补**，不等用户要求。

`prompt_builder.py` 的 `SKILLS_GUIDANCE` 里写着：
> "当使用 skill 发现其过时、不完整或错误时，立即用 `skill_manage(action='patch')` 修补它 — 不要等待用户要求。未维护的 skill 会成为负资产。"

### 三种更新方式

| 方式 | 场景 | 实现 |
|------|------|------|
| `patch` | 局部修改（推荐） | 模糊匹配引擎，容忍空白/缩进差异 |
| `edit` | 全量重写 | 完整替换 SKILL.md |
| `write_file` | 增加支持文件 | 写入 references/templates/scripts/ |

修补后自动 `clear_skills_system_prompt_cache()`，下次对话立即生效。

### patch 流程

```
skill_manage(action="patch", name="xxx", old_string="...", new_string="...")
    ├── 查找现有 skill
    ├── 验证可写性（只允许本地 skill）
    ├── 读取目标文件
    ├── 模糊匹配查找 old_string（容忍空白/缩进差异）
    ├── 验证结果（大小 + frontmatter 格式）
    ├── 原子写入
    ├── 安全扫描 → 失败则回滚
    └── 清除系统提示缓存
```

---

## 四、Skill 评估

项目**没有打分系统**，评估是隐式的，通过以下机制实现：

### 就绪状态评估（SkillReadinessStatus）

```python
AVAILABLE     # 可直接使用
SETUP_NEEDED  # 缺少环境变量/凭证文件
UNSUPPORTED   # 当前平台不支持
```

### 条件激活（动态可见性）

```yaml
# 有 web_search 工具时隐藏这个 skill（工具更好）
fallback_for_tools: [web_search]

# 没有 terminal 工具集时隐藏（无法执行）
requires_toolsets: [terminal]
```

### 安全性评估（创建/更新时）

- 扫描提示注入模式（10+ 种恶意模式）
- 检测隐形 Unicode 字符
- 路径遍历防护
- 不通过 → 阻止创建/回滚

---

## 五、完整生命周期

```
发现复杂工作流
    │
    ▼
skill_manage(create)          ← 验证 + 安全扫描 + 原子写入
    │
    ▼
~/.hermes/skills/my-skill/
    SKILL.md
    references/
    templates/
    │
    ▼
启动时 build_skills_system_prompt()
    三层缓存 → 过滤 → 注入 system prompt 索引
    │
    ▼
用户 /my-skill 或 LLM 主动调用 skill_view()
    │
    ▼
完整内容注入上下文，agent 按步骤执行
    │
    ▼
发现 skill 内容过时/有误
    │
    ▼
skill_manage(patch) 立即修补 → 清除缓存
    │
    ▼
（循环往复，skill 越用越准确）
```

**本质：** skill 是 agent 的**自进化程序性记忆** — 创建、使用、自我修补，形成闭环。没有外部评分，质量靠"用坏了自己改"来保证。

---

## 六、关键文件速查

| 文件 | 关键函数 | 职责 |
|------|---------|------|
| `tools/skill_manager_tool.py` | `_create_skill` `_patch_skill` `_edit_skill` | 创建/修改/删除 |
| `agent/skill_utils.py` | `parse_frontmatter` `get_all_skills_dirs` `extract_skill_conditions` | 解析与目录管理 |
| `agent/skill_commands.py` | `scan_skill_commands` `build_skill_invocation_message` | /slash 命令处理 |
| `agent/prompt_builder.py` | `build_skills_system_prompt` `_skill_should_show` | 缓存 + 注入 system prompt |
| `tools/skills_list_tool.py` | `skills_list` | 列出所有 skill（轻量） |
| `tools/skill_view_tool.py` | `skill_view` | 读取完整 skill 内容 |
| `tools/skills_guard.py` | `scan_skill` `should_allow_install` | 安全扫描 |

---

## 七、降级设计原理

### 为什么要有降级

同一件事往往有两种做法：

| 方式 | 优点 | 缺点 |
|------|------|------|
| 原生工具（如 `web_search`） | 快、精准、结构化结果 | 需要 API key / 特定环境 |
| Skill（步骤说明） | 零依赖，任何环境能用 | 慢、靠 agent 手动执行 |

若两者同时出现在 system prompt，agent 会困惑该用哪个。降级机制让 skill **按需显现**：有工具用工具，没工具用 skill，自动切换。

### 实现方式：纯代码过滤，不是提示词

在 `agent/prompt_builder.py:552` 的 `_skill_should_show()` 函数里：

```python
def _skill_should_show(conditions, available_tools, available_toolsets) -> bool:

    # fallback_for：主工具"存在"时 → 返回 False（不显示这个 skill）
    for t in conditions.get("fallback_for_tools", []):
        if t in available_tools:
            return False          # 直接过滤掉，不进 prompt

    # requires：必需工具"不存在"时 → 返回 False（也不显示）
    for t in conditions.get("requires_tools", []):
        if t not in available_tools:
            return False

    return True
```

被排除的 skill 在 `<available_skills>` 文本生成**之前**就被过滤掉，LLM 完全看不到它。

```
build_skills_system_prompt(available_tools={"web_search", ...})
    │
    └─► 遍历所有 skill
        └─► _skill_should_show(conditions, available_tools)
            ├── web_search in available_tools → False → 跳过
            └─► 其他 skill → True → 加入 index_lines
    │
    └─► 生成 <available_skills> 文本（已过滤）注入 system prompt
```

---

## 八、三层缓存详解

### 三层内容完全不同

```
第1层 LRU（内存）
  存：最终 prompt 字符串（已过滤）
  失效：skill_manage 操作后主动 clear()
  ↓ miss
第2层 磁盘快照（JSON）
  存：所有 skill 元数据（未过滤）+ 文件指纹
  失效：任意 SKILL.md 的 mtime 或 size 变化
  ↓ miss（文件被改 或 首次启动）
第3层 文件系统扫描
  读：每个 SKILL.md 实际内容
  → 解析完写回第2层快照
  → 过滤后的结果写入第1层 LRU
```

### 第1层：进程内 LRU（`_SKILLS_PROMPT_CACHE`）

```python
_SKILLS_PROMPT_CACHE: OrderedDict[tuple, str]  # 最多8条
```

- **存的内容**：最终完整 prompt 字符串（已过滤、已格式化）
- **缓存键**：`(skills_dir, external_dirs, available_tools, available_toolsets, platform)` — 工具集不同，缓存条目不同
- **特点**：命中直接 return，零计算；gateway 多平台最多存 8 条不同组合

### 第2层：磁盘快照（`.skills_prompt_snapshot.json`）

```json
{
  "version": 1,
  "manifest": {
    "base/SKILL.md": [mtime_ns, size],
    "mlops/axolotl/SKILL.md": [mtime_ns, size]
  },
  "skills": [
    {
      "skill_name": "axolotl",
      "category": "mlops",
      "description": "Fine-tune LLMs",
      "platforms": [],
      "conditions": {
        "fallback_for_tools": ["web_search"],
        "requires_toolsets": []
      }
    }
  ],
  "category_descriptions": {}
}
```

- **存的内容**：所有 skill 的原始元数据，**不含**过滤结果
- **失效判断**：读出后重新计算 manifest，与快照对比；任意文件 mtime/size 变化即失效
- **特点**：跨进程存活；命中后仍需运行 `_skill_should_show()` 做实时过滤

### 第3层：全量文件扫描（冷路径）

- **存的内容**：不存任何东西，直接读 SKILL.md 文件
- **作用**：前两层都 miss 时执行，扫描完写入第2层快照

### 关键差异

| | 第1层 LRU | 第2层磁盘快照 |
|---|-----------|-------------|
| 存储内容 | 最终 prompt 字符串（已过滤） | 原始元数据（未过滤） |
| 条目数量 | 按工具集组合，最多8条 | 全局唯一一份 |
| 过滤时机 | 写入时已完成 | 读出后每次实时过滤 |
| 生命周期 | 进程内，skill_manage 后主动清除 | 跨进程，文件变化后自动失效 |

---

## 九、Skill 评估详细实现（`skill_view()`）

### 完整逻辑流程

```
skill_view("axolotl")
    │
    ├─① 插件名称路由
    │   "namespace:bare" 格式 → _serve_plugin_skill()
    │   普通名称 → 继续向下
    │
    ├─② 文件查找（三种策略，本地优先）
    │   a. 直接路径匹配：skills_dir / "mlops/axolotl"
    │   b. rglob SKILL.md 按目录名匹配：parent.name == "axolotl"
    │   c. 遗留 flat .md 文件：skills_dir / "axolotl.md"
    │   → 找不到：返回 available_skills 列表
    │
    ├─③ 安全检查（两项）
    │   a. 路径信任检测：skill_md 是否在 trusted_dirs 内
    │   b. 注入模式扫描：_INJECTION_PATTERNS（10+种）
    │   → 仅记录 warning，继续服务（不阻断）
    │
    ├─④ 平台检查（阻断性）
    │   skill_matches_platform(parsed_frontmatter)
    │   → 不兼容 → readiness_status=UNSUPPORTED，返回错误
    │
    ├─⑤ 禁用检查（阻断性）
    │   _is_skill_disabled(resolved_name)
    │   → 被禁用 → 返回错误，提示 "hermes skills" 命令
    │
    ├─⑥ 子文件请求（file_path 参数）
    │   有 file_path → 路径穿越检查 → 读取子文件 → 早期返回
    │   没有 → 继续加载主内容
    │
    ├─⑦ 环境变量检测与捕获
    │   _get_required_environment_variables(frontmatter, legacy_env_vars)
    │       ├─ required_environment_variables 字段（新格式）
    │       ├─ setup.collect_secrets（交互式配置）
    │       └─ prerequisites.env_vars（旧格式，向后兼容）
    │
    │   _capture_required_environment_variables(skill_name, missing)
    │       ├─ 本地 CLI：弹出交互式输入提示，写入 ~/.hermes/.env
    │       ├─ Gateway 表面：返回 gateway_setup_hint（文字提示）
    │       └─ 远程 backend (Modal/Docker)：追加"需在远端环境配置"说明
    │
    ├─⑧ 凭证文件注册
    │   required_credential_files → register_credential_files()
    │   → 文件存在：挂载到远程沙箱
    │   → 文件缺失：setup_needed = True
    │
    ├─⑨ 环境透传注册
    │   register_env_passthrough(available_env_names)
    │   → 已配置的 key 注册进 execute_code/terminal 的 passthrough 列表
    │
    └─⑩ 构建 SkillReadinessStatus 并返回
        AVAILABLE：所有必需项齐全
        SETUP_NEEDED：remaining_missing_required_envs 非空 或 missing_cred_files 非空
        UNSUPPORTED：平台不匹配（第④步阻断）
```

### 返回结构

```json
{
  "success": true,
  "name": "axolotl",
  "content": "---\nname: axolotl\n...",
  "linked_files": {
    "references": ["references/api.md"],
    "templates": ["templates/config.yaml"],
    "scripts": ["scripts/train.sh"]
  },
  "required_environment_variables": [...],
  "missing_required_environment_variables": [],
  "setup_needed": false,
  "readiness_status": "available",
  "usage_hint": "To view linked files, call skill_view(name, file_path)..."
}
```

### 关键设计点

**渐进式披露三层**：`skills_list`（仅索引）→ `skill_view`（完整 SKILL.md）→ `skill_view(name, file_path)`（子文件），按需加载节省 Token。

**安全警告不阻断**：注入检测只 `logger.warning`，不拒绝服务。阻断发生在创建时（`skill_manager_tool.py` 的安全扫描会回滚）。

**环境捕获差异化**：

| 表面 | 行为 |
|------|------|
| 本地 CLI | 弹交互式输入框，写入持久 `.env` |
| Gateway（Web/API） | 无法交互，返回文字 hint |
| 远程 backend（Modal/Docker） | 追加"需在远端环境内设置"说明 |

---

## 十、Skill 加载到 Agent 详细实现（`build_skills_system_prompt()`）

### 完整逻辑流程

```
build_skills_system_prompt(available_tools, available_toolsets)
    │
    ├─① 缓存键构建
    │   (skills_dir, external_dirs_tuple,
    │    sorted_tools_tuple, sorted_toolsets_tuple,
    │    platform_hint)   ← HERMES_PLATFORM 或 session 环境变量
    │
    ├─② Layer 1：进程内 LRU（OrderedDict，最多8条）
    │   命中 → move_to_end（LRU 刷新）→ 直接 return
    │   未命中 → 继续
    │
    ├─③ Layer 2：磁盘快照 .skills_prompt_snapshot.json
    │   _load_skills_snapshot()：
    │       读 JSON → 比对 manifest（mtime_ns + size）
    │       任意 SKILL.md 变动 → 返回 None（快照失效）
    │   快照有效 → 遍历 skills[]：
    │       skill_matches_platform() → 平台过滤
    │       name in disabled → 禁用过滤
    │       _skill_should_show() → 条件激活过滤
    │       → 通过：加入 skills_by_category
    │   快照失效 → 进入 Layer 3
    │
    ├─④ Layer 3：全量文件系统扫描（冷路径）
    │   iter_skill_index_files(skills_dir, "SKILL.md")
    │   同样执行平台/禁用/_skill_should_show 三重过滤
    │   → 写入新快照（原子写入）→ 供下次使用
    │
    ├─⑤ 外部目录扫描（external_dirs，无快照缓存）
    │   seen_skill_names 防重复（本地 skill 优先）
    │
    ├─⑥ 构建 index_lines
    │   按 category 分组，字母序排序
    │   格式：
    │       mlops:
    │         - axolotl: Fine-tune LLMs with Axolotl
    │         - unsloth: Fast LoRA training
    │
    └─⑦ 注入强制指令 + 写入 LRU 缓存
        "## Skills (mandatory)\n"
        "Before replying, scan the skills below. If a skill matches...
         you MUST load it with skill_view(name)..."
        "<available_skills>\n" + index_lines + "\n</available_skills>"
        "Only proceed without loading a skill if genuinely none are relevant."
```

### 两者协作的完整链路

```
Agent 启动
    │
    ▼
build_skills_system_prompt()          ← 仅索引（name+description）
    → "<available_skills>             ← 注入 system prompt
         mlops:
           - axolotl: Fine-tune LLMs
       </available_skills>"
    │
    ▼
用户发起任务："帮我微调 Llama"
    │
    ▼
LLM 扫描 <available_skills>，发现 axolotl 匹配
    → tool_call: skill_view("axolotl")
    │
    ▼
skill_view() 执行：
    路径查找 → 安全检查 → 平台检查 → 禁用检查
    → 环境变量捕获（CLI 交互 or Gateway hint）
    → register_env_passthrough（透传给 execute_code）
    → 返回完整 SKILL.md + linked_files 列表
    │
    ▼
LLM 按 SKILL.md 步骤执行任务
发现 references/dataset-formats.md 需要看
    → tool_call: skill_view("axolotl", "references/dataset-formats.md")
    │
    ▼
任务完成，skill 有误 → skill_manage(action="patch") 自动修补
```

### 设计解决的问题场景

| 问题 | 解决方案 |
|------|---------|
| Gateway 服务多平台，相同工具集不同平台 skill 列表不同 | 缓存键含 `platform_hint`，每平台独立缓存条目 |
| SKILL.md 被修改但 LRU 还是旧内容 | `skill_manage` 成功后调 `clear_skills_system_prompt_cache(clear_snapshot=True)`，双层全清 |
| Agent 重启后冷启动扫描太慢 | 磁盘快照跨进程存活，mtime/size 验证，无文件变动直接用 |
| 本地 skill 与外部目录 skill 同名冲突 | `seen_skill_names` 集合，本地优先，外部跳过重名 |
| 网关/远程沙箱无法弹出 secret 输入 | 三路 surface 检测：CLI 交互输入、Gateway 文字 hint、Remote backend 追加说明 |
| skill 的 API Key 不被 execute_code 看到 | `register_env_passthrough()` 在 `skill_view()` 时注册，沙箱工具自动透传 |
| LLM 跳过 skill 直接用通用工具 | system prompt 用强硬语气："you MUST load it"，"Err on the side of loading" |

---

## 十一、对接外部 SaaS API Key 的收集与存储

### 功能入口

Skill 通过 `required_environment_variables` 或 `setup.collect_secrets` 声明需要哪些 key，用户第一次调用 skill 时系统自动弹框收集并持久化，后续无感复用。

### SKILL.md 声明方式

**方式一：`required_environment_variables`（推荐）**

```yaml
---
name: stripe-payment
description: Stripe 支付 API 对接
required_environment_variables:
  - name: STRIPE_API_KEY
    prompt: "Stripe Secret Key (https://dashboard.stripe.com/apikeys)"
    help: "填写 sk-live_xxx 格式的 Secret Key"
  - name: STRIPE_WEBHOOK_SECRET
    prompt: "Stripe Webhook Secret"
    optional: true   # 没有也能运行
---
```

**方式二：`setup.collect_secrets`**

```yaml
---
name: notion-tool
description: Notion API 对接
setup:
  help: "在 https://www.notion.so/my-integrations 创建 Integration 获取 Token"
  collect_secrets:
    - env_var: NOTION_API_KEY
      prompt: "Notion Integration Token"
      provider_url: "https://www.notion.so/my-integrations"
      secret: true
---
```

### 执行流程

```
skill_view("stripe-payment")
    │
    ├─ ~/.hermes/.env 里有 STRIPE_API_KEY 吗？
    │   有 → readiness_status=AVAILABLE，直接执行
    │   没有 → _capture_required_environment_variables()
    │
    ▼
【本地 CLI】
    TUI 弹出输入框（getpass，屏幕不显示）：
    "Stripe Secret Key (https://dashboard.stripe.com/apikeys)"
    ESC / 直接回车 → 跳过（setup_skipped=True）
    输入后回车 → save_env_value_secure()
              → 原子写入 ~/.hermes/.env
              → os.chmod(.env, 0o600)
              → os.environ[key] = value（当前进程立即生效）
              → 提示："✓ Stored secret in ~/.hermes/.env as STRIPE_API_KEY"
              → 值不暴露给 LLM

【Gateway / Web 端】
    无法弹输入框 → 返回文字提示：
    "请在本地 CLI 运行以完成配置，或手动添加到 ~/.hermes/.env"

【跳过的情况】
    readiness_status = "setup_needed"
    setup_note = "Setup needed: missing $STRIPE_API_KEY"
    → skill 内容正常返回，附带配置说明
```

### 关键实现（`callbacks.py:prompt_for_secret`）

```python
def prompt_for_secret(cli, var_name, prompt, metadata=None):
    # 有 TUI app → 弹输入面板（prompt_toolkit）
    # 无 TUI app → getpass 命令行输入
    value = getpass.getpass(f"{prompt} (hidden, ESC or empty Enter to skip): ")

    if not value:
        return {"success": True, "skipped": True}

    stored = save_env_value_secure(var_name, value)
    # 返回给 LLM 的内容：
    # {"success": True, "stored_as": "STRIPE_API_KEY", "validated": False}
    # 实际 value 不在返回值里
```

### 设计要点

| 设计点 | 说明 |
|--------|------|
| **值不暴露给模型** | `getpass` 输入，只把"已存储"的结果返回给 LLM，实际 key 不进上下文 |
| **只输入一次** | 存入 `.env` 后下次 `_is_env_var_persisted()` 返回 True，不再弹框 |
| **optional 支持** | `optional: true` 的 key 缺失不影响 AVAILABLE 状态 |
| **沙箱透传** | `register_env_passthrough()` 自动把 key 注入 `execute_code`/`terminal` 工具 |
| **凭证文件** | `required_credential_files` 声明的文件注册后自动挂载进远程沙箱（Docker/Modal） |
