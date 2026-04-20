# Skill 继承体系与版本迭代（AI 中台设计）

> 来源: [[raw/skill迭代.md]] + [[raw/Skill 迭代设计 — Hermes Agent 参考对标.md]]
> 参考框架: [[skills/hermes-skills.md]]

## 设计目标

解决 ToB 场景三个核心问题：

1. **组织架构继承** — 集团统一管控，各层按需覆盖，末端灵活扩展
2. **版本迭代** — 安全发布、灰度验证、快速回滚、多版本并存
3. **运行时动态解析** — 按（租户 + 部门 + 角色 + Agent）实时计算可用 Skill 集

---

## 一、树形继承模型

```
平台层 (Platform)   ← 全局基础 Skill（web_search, file_reader...）
    └── 集团层 (Tenant) ← 集团级（crm_query, erp_lookup...）
            └── BU/事业部  ← BU 定制（sales_forecast...）
                    └── 部门层  ← 部门定制（code_review...）
                            └── 团队/角色 ← 最细粒度（admin_console...）
```

**三条继承铁律：**
1. **自动继承**：子节点自动获得父节点所有 enabled Skill
2. **就近覆盖**：子节点可 override 父节点同名 Skill
3. **禁止上穿**：子节点新增的 Skill 不向上传播

数据库用 PostgreSQL `ltree` 物化路径索引支持高效祖先/后代查询。

---

## 二、版本迭代策略

| 发布类型 | 触发条件 | 发布方式 |
|---|---|---|
| 热修复 | 紧急 Bug | 直接发布到 stable，无灰度 |
| 常规迭代 | 功能优化 | canary(5%) → beta(20%) → stable |
| 重大重构 | 破坏性变更 | 新版本号并存，逐步迁移，旧版 deprecated |

灰度路由键：`tenant_id`，同一租户在灰度期间始终用同一版本。

---

## 三、渐进式加载（参考 Hermes，ToB 版）

**问题**：Worker 在 `run_agent_loop()` 时一次性注入 30+ Skill 的 tool_schema，浪费大量 token（~9000 token），有效负载不到 20%。

**方案**：

```
Level 0  系统提示只注入 Skill 名称+描述索引（~1500 token）
          + 注册 load_skill meta-tool

Level 1  LLM 决定用某个 Skill → 调 load_skill(name)
          → 动态注入 tool_schema 到后续 LLM 调用

Level 2  Skill 需要参考文档 → 按需加载 references/
```

**节省效果**：~9000 token → ~1500 token，节省 80%+。

**Agent Loop 集成要点**：
- `active_tools` 初始只含 `load_skill` meta-tool
- LLM 调用 `load_skill("crm_query")` 后，把 Skill 的 tool_schema 追加到 `active_tools`
- 已加载的 Skill 在后续轮次继续可用

---

## 四、条件激活（参考 Hermes，ToB 扩展）

Hermes 原生支持：`requires_toolsets`、`requires_tools`、`fallback_for_tools`、`platforms`

**ToB 新增维度**：

```python
requires_channel: ["feishu", "wecom"]         # 仅特定渠道可用
requires_chat_type: ["direct", "group"]        # 仅群聊/仅私聊
time_window: {from:"09:00", to:"18:00", weekdays:[1-5]}  # 时间窗口
feature_flags: ["beta_feature_x"]             # 功能开关
fallback_for_skills: ["premium_crm"]          # 某 Skill 不可用时的替代
min_user_tier: "standard"                     # 最低用户等级
```

---

## 五、Agent-Managed Skills（受控自学习）

Hermes 允许 Agent 自由创建/修改 Skill。ToB 必须加审核流程：

```
Agent 完成复杂任务（≥5 tool calls 或经历报错后成功）
    ↓
Agent 提议创建 Skill（SkillProposal，仅提议不直接写库）
    ↓
LLM 从执行记录提炼 Skill 草稿（name/description/procedure/pitfalls）
    ↓
通知团队管理员 / Skill Owner 审核
    ↓
批准 → status=canary → 灰度验证 → stable
拒绝 → 归档，附理由
```

**触发条件（四类）**：

| 触发类型 | 条件 |
|---|---|
| complex_task_completed | tool_call_count ≥ 5 且成功 |
| error_recovery | 经历报错后找到正确路径 |
| user_correction | 用户纠正后掌握正确方法 |
| repeated_workflow | 相似工作流已执行 3+ 次 |

---

## 六、Skills Hub 企业版（6 级信任体系）

| 等级 | 来源 | 自动批准 | 扫描 | 沙箱测试 |
|---|---|---|---|---|
| 5 platform_builtin | 平台内置 | ✅ | ❌ | ❌ |
| 4 enterprise_official | 企业 IT 发布 | ✅ | ✅ | ❌ |
| 3 division_shared | BU 级共享，经 BU 审核 | ❌ | ✅ | ✅ |
| 2 department_custom | 部门定制，经部门审核 | ❌ | ✅ | ✅ |
| 1 marketplace_verified | 第三方市场（已审核） | ❌ | ✅ | ✅ |
| 0 marketplace_community | 社区贡献 | ❌ | ✅ | ✅，先隔离 |

对比 Hermes：builtin=5，official=4，trusted=1，community=0。

---

## 七、ToB SKILL.md 格式扩展

在 Hermes 标准格式上增加 `enterprise` 块：

```yaml
---
name: crm-query
version: 2.0.0
# ... Hermes 标准字段 ...

enterprise:
  trust_level: enterprise_official
  owner_node: tenant_acme
  security:
    risk_level: medium
    data_classification: internal
    pii_involved: true
    requires_approval: false
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
    vault_ref: "secret/crm/service_token"   # ToB 扩展：走 Vault 而非 .env 文件
---
```

---

## 八、Hermes (ToC) vs 中台 (ToB) 核心差异

| 特性 | Hermes ToC | 中台 ToB |
|---|---|---|
| Skill 存储 | 文件系统（`~/.hermes/skills/`） | 数据库 + 组织树绑定 |
| Agent 创建 Skill | 自由创建，立即生效 | 草稿 → 审核 → 发布 |
| 版本管理 | 直接覆盖文件 | 语义化版本 + 灰度发布 |
| 多租户 | 无（单用户） | 组织树继承 + 权限隔离 |
| 凭证管理 | `~/.hermes/.env` 文件 | Vault 集成 + 租户级隔离 |
| 安全扫描 | 安装时一次 | 安装 + 运行时 + 定期审计 |
| 条件激活 | 工具/平台维度 | 工具/平台 + 渠道/时间/等级 |

**保留 Hermes 精华**：渐进式加载、条件激活、SKILL.md 格式、多源 Hub、安全扫描分级、Agent 提议机制。

---

## 上下游依赖

- 上游: [[architecture/gateway.md]]（Worker 执行时解析 Skill 集）
- 安全: [[security/agent-security.md]]（Skill 供应链安全）
- 参考实现: [[skills/hermes-skills.md]]
