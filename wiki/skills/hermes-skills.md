# Hermes Skill 管理

> 来源: [[raw/hermes-skill管理.md]] | 分析日期: 2026-04-16

## 功能清单

| # | 功能 | 状态 |
|---|---|---|
| 1 | Skill 创建（验证 + 安全扫描 + 原子写入） | ✅ |
| 2 | Skill 目录结构（references/templates/scripts/assets） | ✅ |
| 3 | Slash 命令调用（`/skill-name`） | ✅ |
| 4 | 渐进式加载三层（skills_list → skill_view → skill_view+path） | ✅ |
| 5 | 条件激活（platforms / requires_toolsets / fallback_for_tools） | ✅ |
| 6 | 三层缓存（LRU 内存 → 磁盘快照 → 全量扫描） | ✅ |
| 7 | Skill 优化（patch / edit / write_file 三种更新） | ✅ |
| 8 | Skill 评估（skill_view 完整流程，SkillReadinessStatus） | ✅ |
| 9 | 降级机制（fallback_for_tools 纯代码过滤，LLM 不可见） | ✅ |
| 10 | API Key 收集（required_environment_variables / collect_secrets） | ✅ |
| 11 | 凭证文件注册（required_credential_files，沙箱自动挂载） | ✅ |
| 12 | Skills Hub 多源安装（builtin/official/trusted/community） | ✅ |
| 13 | 安全扫描（创建/安装时检测注入模式/危险命令/凭证硬编码） | ✅ |
| 14 | RL 训练集成（Trajectory 生成，GRPO 优化） | ✅ |

---

## 一、Skill 创建

### 触发时机（LLM 自主判断）

- 完成复杂任务（5+ 工具调用）
- 克服了困难的报错
- 发现了非显而易见的工作流程
- 用户明确要求"记住这个做法"

### 实现机制：硬编码 Prompt 驱动

"LLM 自主判断"不是凭感觉，而是通过 `SKILLS_GUIDANCE` 常量注入 system prompt 实现的（`agent/prompt_builder.py:164-171`）：

```python
SKILLS_GUIDANCE = (
    "After completing a complex task (5+ tool calls), fixing a tricky error, "
    "or discovering a non-trivial workflow, save the approach as a "
    "skill with skill_manage so you can reuse it next time.\n"
    "When using a skill and finding it outdated, incomplete, or wrong, "
    "patch it immediately with skill_manage(action='patch') — don't wait to be asked. "
    "Skills that aren't maintained become liabilities."
)
```

**设计要点**：两段指令分别对应两个行为——"完成后主动保存"和"发现过时立即修补"。LLM 读到这段 prompt 后会在满足条件时自发调用 `skill_manage`，无需用户提醒。

### 创建流程

```
skill_manage(action="create", name="xxx", content="...")
    ├── 验证名称（小写、连字符、最多 64 字符）
    ├── 验证 YAML frontmatter（name/description 必填）
    ├── 检查重名（跨所有目录）
    ├── 创建目录 ~/.hermes/skills/{category}/{name}/
    ├── 原子写入 SKILL.md（tmpfile + os.replace）
    └── 安全扫描 → 不通过则回滚删除
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
required_environment_variables:
  - name: API_KEY
    prompt: "API Key 提示文字"
    help: "获取地址说明"
    optional: false
required_credential_files:
  - path: google_token.json
    description: "OAuth 令牌"
metadata:
  hermes:
    tags: [tag1, tag2]
    requires_toolsets: [terminal]
    fallback_for_tools: [web_search]
    related_skills: [other-skill]
---

## 触发条件
## 步骤
## 注意事项
## Pitfalls
```

### Skill 目录结构

```
~/.hermes/skills/
└── {category}/{name}/
    ├── SKILL.md          # 主内容（必需）
    ├── references/       # 参考文档（可选）
    ├── templates/        # 输出模板（可选）
    ├── scripts/          # 辅助脚本（可选）
    └── assets/           # 其他资源（可选）
```

---

## 二、渐进式加载（三层）

```
Level 0  skills_list()             → {name, description, category}（~3k token）
Level 1  skill_view(name)          → 完整 SKILL.md + 元数据 + linked_files 列表
Level 2  skill_view(name, path)    → references/templates/scripts/ 内的子文件
```

Agent 按需加载，不用的 Skill 不消耗 Token。系统提示里只注入 Level 0 索引，LLM 决定用哪个 Skill 后才调 `skill_view()`。

---

## 三、三层缓存

```
第1层 LRU（进程内，OrderedDict 最多 8 条）
  存：最终 prompt 字符串（已过滤）
  失效：skill_manage 操作后主动 clear()
        ↓ miss
第2层 磁盘快照（.skills_prompt_snapshot.json，跨进程存活）
  存：所有 skill 元数据（未过滤）+ 文件指纹（mtime/size）
  失效：任意 SKILL.md 的 mtime 或 size 变化
        ↓ miss
第3层 全量文件扫描（冷路径）
  直接读 SKILL.md → 写回第2层快照 → 过滤后写入第1层 LRU
```

| | 第1层 LRU | 第2层磁盘快照 |
|---|---|---|
| 存储内容 | 最终 prompt 字符串（已过滤） | 原始元数据（未过滤） |
| 条目数 | 按工具集组合，最多 8 条 | 全局唯一一份 |
| 过滤时机 | 写入时已完成 | 读出后每次实时过滤 |
| 生命周期 | 进程内，skill_manage 后主动清除 | 跨进程，文件变化自动失效 |

---

## 四、条件激活

```yaml
requires_toolsets: [terminal]       # 工具集不存在则隐藏
requires_tools: [web_search]        # 特定工具不存在则隐藏
fallback_for_tools: [browser]       # 主工具存在则隐藏（本工具作为备选）
platforms: [macos, linux]           # 平台不匹配则隐藏
```

**实现方式**：纯 Python 代码过滤，在生成 `<available_skills>` 文本**之前**执行，LLM 完全看不到被过滤的 Skill。

```python
# prompt_builder.py:552
def _skill_should_show(conditions, available_tools, available_toolsets) -> bool:
    for t in conditions.get("fallback_for_tools", []):
        if t in available_tools:
            return False   # 主工具存在，备选隐藏
    for t in conditions.get("requires_tools", []):
        if t not in available_tools:
            return False   # 依赖工具缺失，隐藏
    return True
```

---

## 五、Skill 优化（三种更新方式）

| 方式 | 场景 | 实现 |
|---|---|---|
| `patch` | 局部修改（推荐） | 模糊匹配引擎，容忍空白/缩进差异 |
| `edit` | 全量重写 | 完整替换 SKILL.md |
| `write_file` | 增加支持文件 | 写入 references/templates/scripts/ |

修补后自动 `clear_skills_system_prompt_cache()`，下次对话立即生效。

**Agent 主动修补原则**（`prompt_builder.py` SKILLS_GUIDANCE 中写明）：
> 当使用 Skill 发现其过时、不完整或错误时，立即用 `skill_manage(action='patch')` 修补它——不要等待用户要求。未维护的 Skill 会成为负资产。

---

## 六、Skill 评估（skill_view 完整流程）

```
skill_view("axolotl")
    ├─① 插件名称路由（namespace:bare → _serve_plugin_skill()）
    ├─② 文件查找（三策略：直接路径 → rglob 目录名 → 遗留 .md 文件）
    ├─③ 安全检查（路径信任检测 + 注入模式扫描，仅 warning 不阻断）
    ├─④ 平台检查（阻断性，不兼容 → UNSUPPORTED）
    ├─⑤ 禁用检查（阻断性，已禁用 → 返回错误）
    ├─⑥ 子文件请求（file_path 参数 → 路径穿越检查 → 读取子文件）
    ├─⑦ 环境变量检测与捕获（三路 surface：CLI 交互 / Gateway hint / Remote 说明）
    ├─⑧ 凭证文件注册（register_credential_files，缺失则 setup_needed=True）
    ├─⑨ 环境透传注册（register_env_passthrough，注入 execute_code/terminal）
    └─⑩ 返回 SkillReadinessStatus（AVAILABLE / SETUP_NEEDED / UNSUPPORTED）
```

**关键返回结构**：

```json
{
  "success": true,
  "name": "axolotl",
  "content": "---\nname: axolotl\n...",
  "linked_files": {"references": [...], "templates": [...], "scripts": [...]},
  "missing_required_environment_variables": [],
  "setup_needed": false,
  "readiness_status": "available"
}
```

---

## 七、API Key 收集与存储

### 声明方式

```yaml
# 方式一（推荐）
required_environment_variables:
  - name: STRIPE_API_KEY
    prompt: "Stripe Secret Key"
    help: "从 https://dashboard.stripe.com/apikeys 获取"
    optional: false

# 方式二
setup:
  collect_secrets:
    - env_var: NOTION_API_KEY
      prompt: "Notion Integration Token"
      secret: true
```

### 执行流程

| 表面 | 行为 |
|---|---|
| 本地 CLI | TUI 弹框（getpass，屏幕不显示），写入 `~/.hermes/.env`，`chmod 0o600` |
| Gateway/Web | 无法交互，返回文字 hint |
| 远程 backend（Docker/Modal） | 追加"需在远端环境配置"说明 |

**设计要点**：
- 值不暴露给 LLM（只把"已存储"结果返回）
- 原子写入（mkstemp + fsync + os.replace）
- 非 ASCII 字符 strip（防 PDF 复制混入相似字符）
- 只输入一次，后续无感复用

---

## 八、Skills Hub 多源集成

| 来源 | 信任等级 | 安全扫描 | 安装方式 |
|---|---|---|---|
| builtin | 最高 | 无 | 随包内置 |
| official | 高 | 有 | 官方可选 |
| trusted | 中 | 有 | 知名第三方（openai/skills 等） |
| community | 低 | 有，危险即 blocked | `--force` 才能安装 |
| well-known | - | 有 | URL 发现（`/.well-known/skills/index.json`） |
| GitHub | - | 有 | 直接从 repo 安装 |

---

## 九、安全扫描

创建 / 安装 Skill 时强制扫描，检测：

- 提示注入模式（10+ 种恶意模式）
- 危险命令（rm -rf、curl pipe bash 等）
- 硬编码凭证（API Key、密码）
- 隐形 Unicode 字符
- 路径穿越（`../` 等）

通不过则回滚，Skill 不写入磁盘。`skill_view()` 时也会扫描，但仅 warning 不阻断（阻断在创建时）。

---

## 十、完整生命周期

```
发现复杂工作流
    ↓
skill_manage(create)   验证 + 安全扫描 + 原子写入
    ↓
~/.hermes/skills/my-skill/SKILL.md
    ↓
启动时 build_skills_system_prompt()
    三层缓存 → 过滤 → 注入 system prompt 索引（Level 0）
    ↓
用户 /my-skill 或 LLM 调用 skill_view()（Level 1）
    ↓
LLM 按步骤执行，按需加载子文件（Level 2）
    ↓
发现内容过时 → skill_manage(patch) 立即修补 → 清除缓存
    ↓
（循环：skill 越用越准确）
```

**本质**：Skill 是 Agent 的**自进化程序性记忆**，质量靠"用坏了自己改"来保证，没有外部评分系统。

---

## 十一、关键文件速查

| 文件 | 关键函数 | 职责 |
|---|---|---|
| `tools/skill_manager_tool.py` | `_create_skill` `_patch_skill` `_edit_skill` | 创建/修改/删除 |
| `agent/skill_utils.py` | `parse_frontmatter` `get_all_skills_dirs` `extract_skill_conditions` | 解析与目录管理 |
| `agent/skill_commands.py` | `scan_skill_commands` `build_skill_invocation_message` | /slash 命令处理 |
| `agent/prompt_builder.py` | `build_skills_system_prompt` `_skill_should_show` | 缓存 + 注入 system prompt |
| `tools/skills_list_tool.py` | `skills_list` | 列出所有 skill（轻量） |
| `tools/skill_view_tool.py` | `skill_view` | 读取完整 skill 内容 |
| `tools/skills_guard.py` | `scan_skill` `should_allow_install` | 安全扫描 |
| `agent/callbacks.py` | `prompt_for_secret` | API Key 交互式采集 |

---

## 对中台设计的启示

- 参考点 [[skills/skill-iteration.md]]：渐进式加载、条件激活、Agent 草稿审核流程、6 级信任 Hub
