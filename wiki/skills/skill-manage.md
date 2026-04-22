# AI 中台 Skill 管理方案

> 参考实现: [[raw/hermes-skill管理.md]] / [[raw/skill迭代.md]]
> 更新日期: 2026-04-22

## 一句话定位
以 Hermes 的 Skill 机制为参考基准，结合 ToB 多租户场景，设计企业级 Skill 管理体系：树形继承 + 审核发布 + 渐进加载 + 6 级信任 Hub，存储层用 PostgreSQL（元数据/权限/版本）+ OSS 对象存储（Skill 文件内容）。

---

## 功能清单

| # | 功能 | Hermes | 中台 ToB |
|---|---|---|---|
| 1 | Skill 创建（验证 + 安全扫描 + 原子写入） | ✅ 自由创建 | ✅ 草稿→审核→发布 |
| 2 | 目录结构（references/templates/scripts/assets） | ✅ 本地文件系统 | ✅ OSS 对象存储 |
| 3 | Slash 命令调用（`/skill-name`） | ✅ | ✅ |
| 4 | 渐进式加载三层（索引→完整→子文件） | ✅ | ✅ |
| 5 | 条件激活（工具/平台/渠道/时间/等级） | ✅ 工具/平台 | ✅ 扩展更多维度 |
| 6 | 三层缓存（LRU → 磁盘快照 → 全量扫描） | ✅ 本地文件 | ✅ Redis + PostgreSQL |
| 7 | Skill 优化（patch / edit / write_file） | ✅ Agent 自由修改 | ✅ 修改需走审核 |
| 8 | Skill 评估（十步 skill_view 流程） | ✅ | ✅ |
| 9 | 降级机制（fallback_for_tools） | ✅ | ✅ + fallback_for_skills |
| 10 | API Key / 凭证管理 | ✅ `~/.hermes/.env` | ✅ Vault 集成 |
| 11 | Skills Hub 多源安装 | ✅ 4 级 | ✅ 6 级信任体系 |
| 12 | 安全扫描（注入/危险命令/凭证硬编码） | ✅ 安装时 | ✅ 安装+运行时+定期审计 |
| 13 | 多租户组织继承 | ❌ | ✅ PostgreSQL ltree |
| 14 | 版本管理 + 灰度发布 | ❌ | ✅ canary→beta→stable |
| 15 | Agent 提议 + 审核流程 | ❌ | ✅ SkillProposal 工作流 |

---

## 一、Hermes 核心机制（参考基准）

### 1.1 Skill 创建

**触发时机**：通过 `SKILLS_GUIDANCE` 常量注入 system prompt，LLM 读到后自发执行，无需用户提醒：

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
    ├── 原子写入 SKILL.md（tmpfile + os.replace）
    └── 安全扫描 → 不通过则回滚删除
```

### 1.2 SKILL.md 格式

```yaml
---
name: my-skill
description: 一句话描述
platforms: [macos, linux]
prerequisites:
  commands: [curl, jq]
metadata:
  hermes:
    tags: [tag1, tag2]
    requires_toolsets: [terminal]
    fallback_for_tools: [web_search]
    related_skills: [other-skill]
required_environment_variables:
  - name: API_KEY
    prompt: "API Key 提示文字"
    optional: false
---
## 触发条件
## 步骤
## 注意事项
## Pitfalls
```

### 1.3 渐进式加载（三层）

```
Level 0  skills_list()           → {name, description, category}（~1500 token）
Level 1  skill_view(name)        → 完整 SKILL.md + 元数据 + linked_files 列表
Level 2  skill_view(name, path)  → references/templates/scripts/ 内子文件
```

系统提示只注入 Level 0 索引，LLM 决定用哪个 Skill 后才调 `skill_view()`，**节省 token ~80%**（~9000 → ~1500）。

### 1.4 三层缓存

```
第1层 LRU（进程内，OrderedDict 最多 8 条）
  存：最终 prompt 字符串（已过滤）
  失效：skill_manage 操作后主动 clear()
        ↓ miss
第2层 磁盘快照（.skills_prompt_snapshot.json）
  存：所有 skill 元数据 + 文件指纹（mtime/size）
  失效：任意 SKILL.md 的 mtime/size 变化
        ↓ miss
第3层 全量文件扫描（冷路径）
  直接读 SKILL.md → 写回第2层 → 过滤后写入第1层
```

### 1.5 条件激活

```yaml
requires_toolsets: [terminal]    # 工具集不存在则隐藏
fallback_for_tools: [browser]    # 主工具存在则隐藏（本工具作备选）
platforms: [macos, linux]        # 平台不匹配则隐藏
```

纯 Python 代码过滤，在生成 `<available_skills>` 之前执行，LLM 完全看不到被过滤的 Skill。

### 1.6 Skill 评估（十步 skill_view 流程）

```
① 插件名称路由  → ② 文件查找（三策略）→ ③ 安全检查（warning 不阻断）
→ ④ 平台检查（阻断）→ ⑤ 禁用检查（阻断）→ ⑥ 子文件请求
→ ⑦ 环境变量检测 → ⑧ 凭证文件注册 → ⑨ 环境透传注册
→ ⑩ 返回 SkillReadinessStatus（AVAILABLE / SETUP_NEEDED / UNSUPPORTED）
```

### 1.7 安全扫描

创建 / 安装时强制扫描，检测：提示注入（10+ 种）、危险命令（`rm -rf`、`curl pipe bash`）、硬编码凭证、隐形 Unicode、路径穿越。不通过则回滚，不写磁盘。

---

## 二、ToB 中台设计

### 2.1 树形继承模型

```
平台层 (Platform)   ← 全局基础 Skill（web_search, file_reader...）
    └── 集团层 (Tenant) ← 集团级（crm_query, erp_lookup...）
            └── BU/事业部  ← BU 定制（sales_forecast...）
                    └── 部门层  ← 部门定制（code_review...）
                            └── 团队/角色 ← 最细粒度（admin_console...）
```

**三条继承铁律**：
1. **自动继承**：子节点自动获得父节点所有 enabled Skill
2. **就近覆盖**：子节点可 override 父节点同名 Skill
3. **禁止上穿**：子节点新增的 Skill 不向上传播

### 2.2 存储架构：PostgreSQL + OSS

**职责划分**：

| 存储层 | 存什么 | 为什么 |
|---|---|---|
| **PostgreSQL** | Skill 元数据、版本记录、组织绑定、权限、配额、审计日志 | 结构化查询、组织树（ltree）、事务保障 |
| **OSS 对象存储** | SKILL.md 文件、references/、templates/、scripts/、assets/ 等内容文件 | 文件大小不固定、天然版本化、多地域复制、成本低 |

**PostgreSQL Schema**：

```sql
-- Skill 元数据表
CREATE TABLE skills (
    skill_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id     UUID NOT NULL,
    name          VARCHAR(64) NOT NULL,
    version       VARCHAR(32) NOT NULL,         -- 语义化版本，如 2.0.0
    status        VARCHAR(16) NOT NULL,          -- canary/beta/stable/deprecated
    trust_level   SMALLINT NOT NULL,             -- 0-5
    owner_node    ltree NOT NULL,                -- 组织树路径，如 acme.biz_unit.dept
    oss_key       VARCHAR(512) NOT NULL,         -- OSS 对象路径
    created_at    TIMESTAMPTZ DEFAULT now(),
    updated_at    TIMESTAMPTZ DEFAULT now(),
    INDEX idx_tenant_name (tenant_id, name),
    INDEX idx_owner_node  USING GIST (owner_node)   -- ltree 祖先查询
);

-- 版本历史表
CREATE TABLE skill_versions (
    skill_id      UUID REFERENCES skills(skill_id),
    version       VARCHAR(32) NOT NULL,
    oss_key       VARCHAR(512) NOT NULL,         -- 历史版本对象路径
    changelog     TEXT,
    published_at  TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (skill_id, version)
);

-- 审计日志表
CREATE TABLE skill_audit_logs (
    log_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    skill_id      UUID NOT NULL,
    actor_id      UUID NOT NULL,
    action        VARCHAR(32) NOT NULL,          -- create/update/approve/reject/install/invoke
    detail        JSONB,
    created_at    TIMESTAMPTZ DEFAULT now()
);
```

**OSS 对象路径规范**：

```
skills/{tenant_id}/{skill_name}/{version}/SKILL.md
skills/{tenant_id}/{skill_name}/{version}/references/{file}
skills/{tenant_id}/{skill_name}/{version}/templates/{file}
skills/{tenant_id}/{skill_name}/{version}/scripts/{file}
skills/{tenant_id}/{skill_name}/{version}/assets/{file}
```

**读取链路**：

```
Worker 解析 Skill 集
    ↓
PostgreSQL 查询：按 owner_node 祖先路径 + tenant_id 计算可见 Skill 列表（Level 0 索引）
    ↓
LLM 调用 load_skill(name)
    ↓
PostgreSQL 查询：取 stable 版本的 oss_key
    ↓
OSS 下载 SKILL.md（走 CDN 加速，通常 <10ms）
    ↓
返回 Level 1 内容，按需继续加载子文件
```

**缓存策略**：
- Level 0 索引：Redis 缓存，TTL 5 分钟，`skill_manage` 写操作后主动失效
- Level 1 SKILL.md：进程内 LRU（最多 32 条），以 `skill_id + version` 为 key

### 2.3 版本迭代策略

| 发布类型 | 触发条件 | 发布方式 |
|---|---|---|
| 热修复 | 紧急 Bug | 直接发布到 stable，无灰度 |
| 常规迭代 | 功能优化 | canary(5%) → beta(20%) → stable |
| 重大重构 | 破坏性变更 | 新版本号并存，逐步迁移，旧版 deprecated |

灰度路由键：`tenant_id`，同一租户在灰度期间始终用同一版本。

### 2.4 条件激活（ToB 扩展维度）

在 Hermes 基础上新增：

```python
requires_channel: ["feishu", "wecom"]
requires_chat_type: ["direct", "group"]
time_window: {from: "09:00", to: "18:00", weekdays: [1,2,3,4,5]}
feature_flags: ["beta_feature_x"]
fallback_for_skills: ["premium_crm"]     # 某 Skill 不可用时的替代
min_user_tier: "standard"
```

### 2.5 Agent-Managed Skills（受控自学习）

Hermes 允许 Agent 自由创建/修改，ToB 加审核流程：

```
Agent 完成复杂任务（≥5 tool calls 或经历报错后成功）
    ↓
Agent 提议创建 SkillProposal（仅提议，不直接写库）
    ↓
LLM 从执行记录提炼草稿（name/description/procedure/pitfalls）
    ↓
通知团队管理员 / Skill Owner 审核
    ↓
批准 → status=canary → 灰度验证 → stable
拒绝 → 归档，附理由
```

**四类触发条件**：

| 触发类型 | 条件 |
|---|---|
| `complex_task_completed` | tool_call_count ≥ 5 且成功 |
| `error_recovery` | 经历报错后找到正确路径 |
| `user_correction` | 用户纠正后掌握正确方法 |
| `repeated_workflow` | 相似工作流已执行 3+ 次 |

### 2.6 Skills Hub 企业版（6 级信任体系）

| 等级 | 来源 | 自动批准 | 安全扫描 | 沙箱测试 |
|---|---|---|---|---|
| 5 `platform_builtin` | 平台内置 | ✅ | ❌ | ❌ |
| 4 `enterprise_official` | 企业 IT 发布 | ✅ | ✅ | ❌ |
| 3 `division_shared` | BU 级共享，经 BU 审核 | ❌ | ✅ | ✅ |
| 2 `department_custom` | 部门定制，经部门审核 | ❌ | ✅ | ✅ |
| 1 `marketplace_verified` | 第三方市场（已审核） | ❌ | ✅ | ✅ |
| 0 `marketplace_community` | 社区贡献 | ❌ | ✅ | ✅，先隔离 |

### 2.7 ToB SKILL.md 格式扩展

在 Hermes 标准格式上增加 `enterprise` 块：

```yaml
---
name: crm-query
version: 2.0.0
# ... Hermes 标准字段 ...

enterprise:
  trust_level: enterprise_official
  owner_node: tenant_acme.sales
  security:
    risk_level: medium
    data_classification: internal
    pii_involved: true
    audit_level: full
  activation:
    requires_channel: [feishu, wecom]
    time_window: {from: "08:00", to: "22:00", weekdays: [1,2,3,4,5]}
    min_user_tier: standard
  quotas:
    per_user_daily: 200
    per_tenant_daily: 10000
  version_policy:
    min_compatible: "1.5.0"

required_environment_variables:
  - name: CRM_SERVICE_TOKEN
    vault_ref: "secret/crm/service_token"   # ToB：走 Vault，不用 .env 文件
---
```

---

## 三、ToC vs ToB 核心对比

| 特性 | Hermes ToC | 中台 ToB |
|---|---|---|
| Skill 存储 | 本地文件系统 `~/.hermes/skills/` | PostgreSQL（元数据）+ OSS（文件内容） |
| 缓存 | 进程内 LRU + 本地磁盘快照 | Redis（Level 0 索引）+ 进程内 LRU（Level 1） |
| Agent 创建 Skill | 自由创建，立即生效 | 草稿 → 审核 → canary → stable |
| 版本管理 | 直接覆盖文件 | 语义化版本 + 灰度发布 + OSS 历史版本保留 |
| 多租户 | 无（单用户） | PostgreSQL ltree 组织树继承 + 权限隔离 |
| 凭证管理 | `~/.hermes/.env` 文件 | Vault 集成 + 租户级隔离 |
| 安全扫描 | 安装时一次 | 安装 + 运行时 + 定期审计 |
| 条件激活 | 工具/平台 | 工具/平台 + 渠道/时间/等级/功能开关 |

**保留 Hermes 精华**：渐进式加载（三层）、条件激活机制、SKILL.md 格式、安全扫描分级、Agent 提议机制、十步 skill_view 评估流程。

---

## 上下游依赖

- 上游: [[architecture/gateway.md]]（Worker 执行时解析 Skill 集，触发 Level 0 注入）
- 安全: [[security/agent-security.md]]（Skill 供应链安全、注入扫描）
- 时序: [[flows/skill-loading.md]]（渐进加载时序图）
