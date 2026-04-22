# AI 中台 Skill 管理方案

> 参考实现: [[raw/hermes-skill管理.md]] / [[raw/skill迭代.md]]
> 更新日期: 2026-04-22

## 一句话定位
以 Hermes 的 Skill 机制为参考基准，结合 ToB 多租户场景，设计企业级 Skill 管理体系：树形继承 + 审核发布 + 渐进加载 + 6 级信任 Hub，存储层用 PostgreSQL（元数据/权限/版本）+ OSS 对象存储（Skill 文件内容）。

---

## 功能清单

| #   | 功能                                        | Hermes             | 中台 ToB                               | 价值  |
| --- | ----------------------------------------- | ------------------ | ------------------------------------ | --- |
| 1   | Skill 创建（验证 + 安全扫描 + 原子写入）                | ✅ 自由创建             | ✅ 草稿→审核→发布                           | ※※※ |
| 2   | 目录结构（references/templates/scripts/assets） | ✅ 本地文件系统           | ✅ OSS 对象存储                           |     |
| 3   | Slash 命令调用（`/skill-name`）                 | ✅                  | ✅                                    |     |
| 4   | 渐进式加载三层（索引→完整→子文件）                        | ✅                  | ✅                                    |     |
| 5   | 条件激活（工具/平台/渠道/时间/等级）                      | ✅ 工具/平台            | ✅ 扩展更多维度                             |     |
| 6   | 三层缓存（LRU → 磁盘快照 → 全量扫描）                   | ✅ 本地文件             | ✅ Redis + PostgreSQL                 |     |
| 7   | Skill 优化（patch / edit / write_file）       | ✅ Agent 自由修改       | ✅ 修改需走审核                             |     |
| 8   | Skill 评估（十步 skill_view 流程）                | ✅                  | ✅                                    |     |
| 9   | 降级机制（fallback_for_tools）                  | ✅                  | ✅ + fallback_for_skills              |     |
| 10  | API Key / 凭证管理                            | ✅ `~/.hermes/.env` | ✅ Vault 集成                           |     |
| 11  | Skills Hub 多源安装                           | ✅ 4 级              | ✅ 6 级信任体系                            |     |
| 12  | 安全扫描（注入/危险命令/凭证硬编码）                       | ✅ 安装时              | ✅ 安装+运行时+定期审计                        |     |
| 13  | 两级继承（个人 + 直接父节点）                          | ❌                  | ✅ parent_node 简单继承                   |     |
| 14  | 版本管理                                      | ❌                  | ✅ 语义化版本 + current_version 指针，旧版本保留回滚 |     |
| 15  | Agent 提议 + 审核流程                           | ❌                  | ✅ SkillProposal 工作流                  |     |

---

## 1. Skill 创建

### Hermes 实现

**触发时机**：通过 `SKILLS_GUIDANCE` 常量注入 system prompt（`agent/prompt_builder.py:164-171`），LLM 读到后自发执行，无需用户提醒：

```python
SKILLS_GUIDANCE = (
    "After completing a complex task (5+ tool calls), fixing a tricky error, "
    "or discovering a non-trivial workflow, save the approach as a skill with skill_manage.\n"
    "When using a skill and finding it outdated, incomplete, or wrong, "
    "patch it immediately with skill_manage(action='patch') — don't wait to be asked."
)
```

**创建流程**：

```
skill_manage(action="create", name="xxx", content="...")
    ├── 验证名称（小写、连字符、最多 64 字符）
    ├── 验证 YAML frontmatter（name/description 必填）
    ├── 检查重名（跨所有目录）
    ├── 创建目录 ~/.hermes/skills/{category}/{name}/
    ├── 原子写入 SKILL.md（tmpfile + os.replace）
    └── 安全扫描 → 不通过则回滚删除
```

**SKILL.md 格式**：

```yaml
---
name: my-skill
description: 一句话描述
platforms: [macos, linux]
prerequisites:
  commands: [curl, jq]
required_environment_variables:
  - name: API_KEY
    prompt: "API Key 提示文字"
    optional: false
metadata:
  hermes:
    tags: [tag1, tag2]
    requires_toolsets: [terminal]
    fallback_for_tools: [web_search]
---
## 触发条件
## 步骤
## 注意事项
## Pitfalls
```

### 中台改造

**草稿→审核→发布**工作流替代 Hermes 的"立即生效"：

```
skill_manage(create) → 写入 skill_proposals 表（status=draft）
    ↓
LLM 提炼草稿（name/description/procedure/pitfalls）
    ↓
通知 Skill Owner / Admin 审核
    ↓
批准：写入 skills 表 + OSS，直接发布新版本
拒绝：归档，附理由
```

**SKILL.md 格式扩展** `enterprise` 块：

```yaml
enterprise:
  trust_level: enterprise_official
  owner_node_id: <团队/部门在 org_nodes 的主键>
  security:
    risk_level: medium
    data_classification: internal
    pii_involved: true
  activation:
    requires_channel: [feishu, wecom]
    time_window: {from: "08:00", to: "22:00", weekdays: [1,2,3,4,5]}
  quotas:
    per_user_daily: 200
    per_tenant_daily: 10000
```

---

## 2. 目录结构

### Hermes 实现

```
~/.hermes/skills/
└── {category}/{name}/
    ├── SKILL.md          # 主内容（必需）
    ├── references/       # 参考文档
    ├── templates/        # 输出模板
    ├── scripts/          # 辅助脚本
    └── assets/           # 其他资源
```

### 中台改造

元数据 PostgreSQL，内容 OSS 对象存储。Skill 分两大类：**公共 Skill**（组织内共享）和**个人 Skill**（用户私有）。

| 存储层 | 存什么 | 为什么 |
|---|---|---|
| **PostgreSQL** | Skill 元数据、版本记录、组织/用户绑定、权限、配额、审计日志 | 结构化查询、事务保障、简单父子关系即可 |
| **OSS 对象存储** | SKILL.md、references/、templates/、scripts/、assets/ 内容 | 文件大小不固定、天然版本化、多地域复制、成本低 |

**OSS 对象路径规范（按租户 → scope → 草稿/已发布 分层）**：

```
# 公共 Skill - 草稿（审核前，审核通过后迁移到 public 路径）
skills/{tenant_id}/drafts/{proposal_id}/SKILL.md

# 公共 Skill - 已发布（审核通过后直接写入对应版本目录）
skills/{tenant_id}/public/{node_id}/{skill_name}/v{version}/SKILL.md


# 个人 Skill（无草稿阶段，用户自助创建直接版本化）
skills/{tenant_id}/personal/{user_id}/{skill_name}/v{version}/SKILL.md

```

**草稿 → 审核 → 发布 流程**：

```
Agent/用户提议创建
    ↓
写入 skill_proposals 表（status=draft）
    ↓
内容写入 OSS: skills/{tenant_id}/drafts/{proposal_id}/
    ↓
通知 Skill Owner 审核（status=under_review）
    ↓
批准 → 复制 OSS 对象到 skills/{tenant_id}/public/{node_id}/{skill_name}/v{version}/
       → 写入 skills 表，更新 current_version 指向新版本
       → 草稿路径保留 30 天供审计，之后归档
拒绝 → status=rejected，草稿保留 30 天后删除
```

**两类 Skill 对比**：

| 维度 | 公共 Skill | 个人 Skill |
|---|---|---|
| 归属 | 组织节点（团队/部门） | 用户个人 |
| 可见范围 | 仅用户直接父节点可见（不向下深层继承） | 仅本人 |
| 创建流程 | 草稿 → 审核 → 发布 | 用户自助创建，无需审核（Hermes 式） |
| 版本管理 | 语义化版本 + current_version 指针，旧版本保留回滚 | 同左 |
| 安全扫描 | 强制（阻断） | 强制（阻断） |
| 覆盖关系 | 个人 Skill 同名时优先级最高（override 直接父节点） | — |

**合并查询**（当前用户 `user_id=u123`，直接父节点 `parent_node_id=dept_east`）：

```sql
-- 只继承两级：个人 Skill + 直接父节点的公共 Skill
SELECT s.*, v.oss_key
FROM skills s
JOIN skill_versions v
  ON v.skill_id = s.skill_id AND v.version = s.current_version
WHERE s.tenant_id = $1 AND (
    -- 个人 Skill：本人所有
    (s.scope = 'personal' AND s.owner_id = 'u123')
    OR
    -- 公共 Skill：仅用户直接父节点
    (s.scope = 'public'   AND s.owner_id = (
        SELECT parent_node_id FROM users WHERE user_id = 'u123'
    ))
);
-- 同名冲突时在应用层按 scope=personal 优先合并
```

**PostgreSQL Schema**：

```sql
-- 组织节点表（简化为单层父子关系，不用 ltree）
CREATE TABLE org_nodes (
    node_id     UUID PRIMARY KEY,
    tenant_id   UUID NOT NULL,
    parent_id   UUID REFERENCES org_nodes(node_id),   -- NULL 为根节点
    name        VARCHAR(128) NOT NULL
);

-- 用户表（记录直接父节点）
CREATE TABLE users (
    user_id         UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    parent_node_id  UUID REFERENCES org_nodes(node_id)  -- 用户的直接父节点（团队/部门）
);

-- Skill 表（一个 skill 一行，版本通过 current_version 指针）
CREATE TABLE skills (
    skill_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id        UUID NOT NULL,
    scope            VARCHAR(16) NOT NULL,    -- public / personal
    owner_id         UUID NOT NULL,           -- scope=public 时指向 org_nodes.node_id；scope=personal 时指向 users.user_id
    name             VARCHAR(64) NOT NULL,
    current_version  VARCHAR(32) NOT NULL,    -- 指向 skill_versions.version
    trust_level      SMALLINT NOT NULL,       -- 0-5
    created_at       TIMESTAMPTZ DEFAULT now(),
    updated_at       TIMESTAMPTZ DEFAULT now(),
    UNIQUE (tenant_id, scope, owner_id, name),
    INDEX idx_tenant_scope_owner (tenant_id, scope, owner_id)
);

-- 版本表（历史版本都保留，支持回滚）
CREATE TABLE skill_versions (
    skill_id      UUID REFERENCES skills(skill_id),
    version       VARCHAR(32) NOT NULL,       -- 语义化版本号，如 1.0.0
    oss_key       VARCHAR(512) NOT NULL,      -- 该版本的 OSS 对象路径
    changelog     TEXT,
    published_at  TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (skill_id, version)
);

-- 草稿/提议表（仅公共 Skill 走此流程，个人 Skill 不需要）
CREATE TABLE skill_proposals (
    proposal_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id      UUID NOT NULL,
    target_node_id UUID NOT NULL REFERENCES org_nodes(node_id),  -- 提议到哪个组织节点
    proposed_name VARCHAR(64) NOT NULL,
    action        VARCHAR(16) NOT NULL,         -- create / patch / edit
    base_skill_id UUID,                         -- patch/edit 时引用现有 skill
    proposer_id   UUID NOT NULL,                -- 提议人（用户或 Agent）
    trigger_type  VARCHAR(32),                  -- complex_task_completed / error_recovery / user_correction / repeated_workflow
    status        VARCHAR(16) NOT NULL,         -- draft / under_review / approved / rejected
    oss_key       VARCHAR(512) NOT NULL,        -- 草稿 OSS 路径
    reviewer_id   UUID,
    review_note   TEXT,
    created_at    TIMESTAMPTZ DEFAULT now(),
    reviewed_at   TIMESTAMPTZ,
    INDEX idx_tenant_status (tenant_id, status)
);

CREATE TABLE skill_audit_logs (
    log_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    skill_id      UUID NOT NULL,
    actor_id      UUID NOT NULL,
    action        VARCHAR(32) NOT NULL,         -- create/update/approve/reject/install/invoke
    detail        JSONB,
    created_at    TIMESTAMPTZ DEFAULT now()
);
```

---

## 3. Slash 命令调用

### Hermes 实现

用户输入 `/skill-name` 时扫描所有 skill 目录，匹配 slug（SKILL.md 目录名）后触发 `build_skill_invocation_message()`，把 Skill 内容作为 system message 追加给 LLM。

### 中台改造

多租户场景下 `/skill-name` 按"个人 Skill + 直接父节点公共 Skill"两级可见性过滤，同名时个人 Skill 优先。

---

## 4. 渐进式加载（三层）

### Hermes 实现

```
Level 0  skills_list()           → {name, description, category}（~1500 token）
Level 1  skill_view(name)        → 完整 SKILL.md + 元数据 + linked_files 列表
Level 2  skill_view(name, path)  → references/templates/scripts/ 内子文件
```

系统提示只注入 Level 0 索引，LLM 决定用哪个 Skill 后才调 `skill_view()`，**节省 token ~80%**（~9000 → ~1500）。

### 中台改造

读取链路走 PostgreSQL + OSS：

```
Worker 解析 Skill 集
    ↓
PostgreSQL 查询：按 (个人 owner_id=user_id) + (公共 owner_id=用户直接父节点) 计算可见 Skill 列表（Level 0 索引）
    ↓
LLM 调用 load_skill(name)
    ↓
PostgreSQL 查询：取 current_version 指向的 oss_key
    ↓
OSS 下载 SKILL.md（走 CDN 加速，通常 <10ms）
    ↓
返回 Level 1 内容，按需继续加载子文件
```

---

## 5. 条件激活

### Hermes 实现

```yaml
requires_toolsets: [terminal]    # 工具集不存在则隐藏
requires_tools: [web_search]     # 特定工具不存在则隐藏
fallback_for_tools: [browser]    # 主工具存在则隐藏（本工具作备选）
platforms: [macos, linux]        # 平台不匹配则隐藏
```

纯 Python 代码过滤，在生成 `<available_skills>` 文本之前执行，LLM 完全看不到被过滤的 Skill：

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

### 中台改造

在 Hermes 维度基础上新增 ToB 维度：

```python
requires_channel: ["feishu", "wecom"]                       # 仅特定渠道
requires_chat_type: ["direct", "group"]                     # 仅群聊/仅私聊
time_window: {from: "09:00", to: "18:00", weekdays: [1-5]}  # 时间窗口
feature_flags: ["beta_feature_x"]                           # 功能开关
fallback_for_skills: ["premium_crm"]                        # 某 Skill 不可用时的替代
min_user_tier: "standard"                                   # 最低用户等级
```

---

## 6. 三层缓存

### Hermes 实现

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
| 生命周期 | 进程内，skill_manage 后主动清除 | 跨进程，文件变化自动失效 |

### 中台改造

分布式多实例场景，缓存层替换为 Redis + 进程内 LRU：

| 层 | 替代方案 | key |
|---|---|---|
| Level 0 索引 | Redis，TTL 5 分钟，skill_manage 写操作主动失效 | `skills:index:{tenant_id}:{user_id}` |
| Level 1 SKILL.md | 进程内 LRU，最多 32 条 | `{skill_id}:{version}` |
| Level 2 子文件 | OSS CDN 自带缓存 | — |

---

## 7. Skill 优化（三种更新方式）

### Hermes 实现

| 方式 | 场景 | 实现 |
|---|---|---|
| `patch` | 局部修改（推荐） | 模糊匹配引擎，容忍空白/缩进差异 |
| `edit` | 全量重写 | 完整替换 SKILL.md |
| `write_file` | 增加支持文件 | 写入 references/templates/scripts/ |

修补后自动 `clear_skills_system_prompt_cache()`，下次对话立即生效。

### 中台改造

修改同样走**审核流程**，与创建对称：

```
skill_manage(patch/edit) → skill_proposals 表（action=patch, base_skill_id=xxx）
    ↓
审核
    ↓
批准：生成新版本号（patch 递增），写入 OSS 新路径，skills.current_version 指向新版本
```

旧版本 OSS 对象保留，供回滚与审计。

---

## 8. Skill 评估（十步 skill_view 流程）

### Hermes 实现

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

### 中台改造

十步流程整体保留，扩展两项阻断检查：

- **配额检查**：命中 `quotas.per_user_daily` / `per_tenant_daily` → QUOTA_EXCEEDED
- **可见性检查**：Skill 不属于用户本人（personal）也不属于用户直接父节点（public）→ FORBIDDEN

---

## 9. 降级机制

### Hermes 实现

`fallback_for_tools: [browser]` — 主工具（browser）存在时本 Skill 隐藏，作为主工具失效时的备选。

### 中台改造

新增 `fallback_for_skills`：

```yaml
fallback_for_skills: ["premium_crm"]   # premium_crm 不可用（配额/未授权）时启用本 Skill
```

解决企业场景下高配 Skill 不可用时的降级（如 premium_crm → basic_crm）。

---

## 10. API Key / 凭证管理

### Hermes 实现

**声明方式**：

```yaml
required_environment_variables:
  - name: STRIPE_API_KEY
    prompt: "Stripe Secret Key"
    help: "从 https://dashboard.stripe.com/apikeys 获取"
    optional: false
```

**执行流程**：

| 表面 | 行为 |
|---|---|
| 本地 CLI | TUI 弹框（getpass，屏幕不显示），写入 `~/.hermes/.env`，`chmod 0o600` |
| Gateway/Web | 无法交互，返回文字 hint |
| 远程 backend（Docker/Modal） | 追加"需在远端环境配置"说明 |

**设计要点**：值不暴露给 LLM，原子写入（mkstemp + fsync + os.replace），非 ASCII 字符 strip，只输入一次。

### 中台改造

凭证走 **Vault 集成**，租户级隔离：

```yaml
required_environment_variables:
  - name: CRM_SERVICE_TOKEN
    vault_ref: "secret/crm/{tenant_id}/service_token"
```

Worker 执行 Skill 时通过 Vault API 拉取凭证注入环境变量，值不落盘。租户间凭证路径天然隔离，Vault 审计日志记录所有访问。

---

## 11. Skills Hub 多源安装

### Hermes 实现（4 级）

| 来源 | 信任等级 | 安全扫描 | 安装方式 |
|---|---|---|---|
| builtin | 最高 | 无 | 随包内置 |
| official | 高 | 有 | 官方可选 |
| trusted | 中 | 有 | 知名第三方（openai/skills 等） |
| community | 低 | 有，危险即 blocked | `--force` 才能安装 |

### 中台改造（6 级）

| 等级 | 来源 | 自动批准 | 安全扫描 | 沙箱测试 |
|---|---|---|---|---|
| 5 `platform_builtin` | 平台内置 | ✅ | ❌ | ❌ |
| 4 `enterprise_official` | 企业 IT 发布 | ✅ | ✅ | ❌ |
| 3 `division_shared` | BU 级共享，经 BU 审核 | ❌ | ✅ | ✅ |
| 2 `department_custom` | 部门定制，经部门审核 | ❌ | ✅ | ✅ |
| 1 `marketplace_verified` | 第三方市场（已审核） | ❌ | ✅ | ✅ |
| 0 `marketplace_community` | 社区贡献 | ❌ | ✅ | ✅，先隔离 |

企业场景多了组织内层级（BU / 部门），每一级都要对应审核角色。

---

## 12. 安全扫描

### Hermes 实现

创建 / 安装 Skill 时强制扫描，检测：

- 提示注入模式（10+ 种恶意模式）
- 危险命令（`rm -rf`、`curl pipe bash` 等）
- 硬编码凭证（API Key、密码）
- 隐形 Unicode 字符
- 路径穿越（`../` 等）

不通过则回滚，Skill 不写磁盘。`skill_view()` 时也会扫描，但仅 warning 不阻断（阻断在创建时）。

### 中台改造

三重扫描：

| 时机 | 动作 |
|---|---|
| 安装时 | Hermes 原生扫描（阻断）+ 沙箱试运行 |
| 运行时 | 每次 `load_skill` 拉取 OSS 对象时校验 SHA256 防篡改 |
| 定期审计 | 每日批量扫描所有 current_version，发现新型注入模式追溯预警 |

---

## 13. 两级继承（中台新增）

Hermes 无此能力。中台简化为**两级继承**——避免深层组织树带来的查询复杂度和冲突管理成本：

```
个人 Skill (personal)   ← 用户本人
    ↑ override
直接父节点 Skill (public) ← 用户所属的团队/部门
```

**继承规则**：

1. 用户可见 Skill = **本人 personal 全部** + **直接父节点 public 全部**
2. 同名冲突时 **personal 优先**（override public）
3. 不继承祖父及以上节点的 Skill

**为什么不做深层继承**：
- 深层继承的"就近覆盖"语义复杂，用户难以预期自己能用哪些 Skill
- 平台级 / 租户级通用 Skill 由管理员批量创建到各团队节点，省去运行时的继承解析
- 简化后查询不需要 ltree 和 nlevel，普通 B-tree 索引即可命中

**查询示例**：

```sql
-- 已在 #2 给出完整 SQL，两级 UNION 即可
SELECT ... WHERE scope='personal' AND owner_id = :user_id
UNION
SELECT ... WHERE scope='public'   AND owner_id = :user_parent_node_id;
```

---

## 14. 版本管理（中台新增）

Hermes 直接覆盖文件，无版本管理。中台用**语义化版本 + current_version 指针**管理：

**模型**：

```
skills 表             一行记录 = 一个 Skill
  ├── current_version  指向当前活跃版本（如 "1.2.0"）
  └── skill_versions 表 存所有历史版本，每行 = 一个版本

发布新版本：
  1. 写 OSS: skills/{tenant_id}/public/{node_id}/{skill_name}/v1.3.0/...
  2. 插入 skill_versions 一行（skill_id, version=1.3.0, oss_key, changelog）
  3. 更新 skills.current_version = "1.3.0"

回滚旧版本：
  只需改 skills.current_version 指回旧版本号，旧 OSS 对象永久保留
```

**版本号规则**（语义化版本）：
- `major.minor.patch`，如 `2.1.3`
- patch 递增：bug 修复、文档修正
- minor 递增：新增功能、兼容性改动
- major 递增：破坏性变更（触发 Owner 审核更严格）

不做 canary / beta / stable 多阶段灰度——发布即生效，出问题直接回滚。

---

## 15. Agent 提议 + 审核流程（中台新增）

Hermes 允许 Agent 自由创建/修改，ToB 必须加审核：

```
Agent 完成复杂任务（≥5 tool calls 或经历报错后成功）
    ↓
Agent 提议创建 SkillProposal（仅提议，不直接写库）
    ↓
LLM 从执行记录提炼草稿（name/description/procedure/pitfalls）
    ↓
通知团队管理员 / Skill Owner 审核
    ↓
批准 → 复制到 public 路径，更新 skills.current_version
拒绝 → 归档，附理由
```

**四类触发条件**：

| 触发类型 | 条件 |
|---|---|
| `complex_task_completed` | tool_call_count ≥ 5 且成功 |
| `error_recovery` | 经历报错后找到正确路径 |
| `user_correction` | 用户纠正后掌握正确方法 |
| `repeated_workflow` | 相似工作流已执行 3+ 次 |

触发条件检测由 Worker 在 Agent Loop 结束后执行，命中则写入 `skill_proposals` 表并通知审核方。

---

## 关键文件速查（Hermes）

| 文件 | 关键函数 | 职责 |
|---|---|---|
| `tools/skill_manager_tool.py` | `_create_skill` `_patch_skill` `_edit_skill` | 创建/修改/删除 |
| `agent/skill_utils.py` | `parse_frontmatter` `get_all_skills_dirs` | 解析与目录管理 |
| `agent/skill_commands.py` | `scan_skill_commands` | /slash 命令处理 |
| `agent/prompt_builder.py` | `build_skills_system_prompt` `_skill_should_show` | 缓存 + 注入 system prompt |
| `tools/skills_list_tool.py` | `skills_list` | 列出所有 skill（轻量） |
| `tools/skill_view_tool.py` | `skill_view` | 读取完整 skill 内容 |
| `tools/skills_guard.py` | `scan_skill` | 安全扫描 |
| `agent/callbacks.py` | `prompt_for_secret` | API Key 交互式采集 |

---

## 上下游依赖

- 上游: [[architecture/gateway.md]]（Worker 执行时解析 Skill 集，触发 Level 0 注入）
- 安全: [[security/agent-security.md]]（Skill 供应链安全、注入扫描）
- 时序: [[flows/skill-loading.md]]（渐进加载时序图）
