# Skill 继承体系与版本迭代设计

> 版本: v1.0 | 日期: 2026-04-14 | 作者: 大迈
> 隶属于: 企业级多智能体网关架构方案

---

## 一、设计目标

解决 ToB 场景下的三个核心问题：

1. **组织架构 Skill 继承** — 集团统一管控，各层按需覆盖，末端灵活扩展
2. **Skill 版本迭代** — 安全发布、灰度验证、快速回滚、多版本并存
3. **运行时动态解析** — 根据请求上下文（租户+部门+角色+Agent）实时计算可用 Skill 集

---

## 二、组织架构 Skill 继承模型

### 2.1 核心思想：树形级联 + 就近覆盖

类比 CSS 级联规则 / Linux 文件系统挂载 / K8s RBAC 继承：

```
                        ┌─────────────┐
                        │   平台层     │  ← 全局基础 Skill（所有租户共享）
                        │  (Platform)  │     web_search, weather, file_reader...
                        └──────┬──────┘
                               │ 继承
                        ┌──────▼──────┐
                        │   集团层     │  ← 集团级 Skill（本集团所有 BU 共享）
                        │  (Tenant)   │     crm_query, erp_lookup, approval_flow...
                        └──────┬──────┘
                               │ 继承
                  ┌────────────┼────────────┐
                  │                         │
           ┌──────▼──────┐          ┌──────▼──────┐
           │   BU / 事业部 │          │   BU / 事业部 │  ← BU 级定制
           │  (Division)  │          │  (Division)  │     sales_forecast, mkt_analytics...
           └──────┬──────┘          └─────────────┘
                  │ 继承
           ┌──────▼──────┐
           │    部门层     │  ← 部门级定制
           │ (Department) │     code_review, deploy_pipeline...
           └──────┬──────┘
                  │ 继承
           ┌──────▼──────┐
           │   团队/角色   │  ← 最细粒度
           │ (Team/Role)  │     debug_tool, admin_console...
           └─────────────┘
```

**继承规则（三条铁律）：**

1. **自动继承**：子节点自动获得父节点的所有 enabled Skill
2. **就近覆盖**：子节点可以 override 父节点的同名 Skill（配置、版本、启禁用）
3. **禁止上穿**：子节点新增的 Skill 不会向上传播

### 2.2 数据模型

```sql
-- ============ 组织节点表（树形结构）============
CREATE TABLE org_nodes (
    node_id         VARCHAR(64) PRIMARY KEY,
    parent_id       VARCHAR(64) REFERENCES org_nodes(node_id),
    node_type       VARCHAR(16) NOT NULL,      -- platform / tenant / division / department / team / role
    node_name       VARCHAR(128) NOT NULL,
    tenant_id       VARCHAR(64),               -- 所属租户（platform 层为 NULL）
    path            LTREE NOT NULL,            -- 物化路径 (PostgreSQL ltree)
                                               -- 例: "platform.tenant_001.div_sales.dept_华东"
    depth           INT NOT NULL,              -- 层级深度（platform=0, tenant=1, ...）
    status          VARCHAR(16) DEFAULT 'active',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_org_path ON org_nodes USING GIST (path);
CREATE INDEX idx_org_parent ON org_nodes(parent_id);
CREATE INDEX idx_org_tenant ON org_nodes(tenant_id);


-- ============ Skill 定义表（全局 Skill 注册中心）============
CREATE TABLE skills (
    skill_id        VARCHAR(64) PRIMARY KEY,
    skill_name      VARCHAR(128) NOT NULL,
    display_name    VARCHAR(128),
    description     TEXT,
    skill_type      VARCHAR(32) NOT NULL,      -- tool / knowledge / workflow / composite
    category        VARCHAR(64),               -- 分类标签: "数据分析" / "代码" / "办公"
    
    -- 创建者信息
    created_by      VARCHAR(64),               -- 创建者 user_id
    owner_node_id   VARCHAR(64) REFERENCES org_nodes(node_id),  -- 归属组织节点
    visibility      VARCHAR(16) DEFAULT 'inherit',  -- public / inherit / private
                                               -- public: 所有人可见
                                               -- inherit: 遵循组织继承规则
                                               -- private: 仅创建者/指定节点可见
    
    -- 元数据
    tags            JSONB DEFAULT '[]',        -- ["sales", "analytics", "internal"]
    requires        JSONB,                     -- 运行时依赖 {"bins": [], "env": [], "config": []}
    
    status          VARCHAR(16) DEFAULT 'active',  -- active / deprecated / archived
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_skills_owner ON skills(owner_node_id);
CREATE INDEX idx_skills_type ON skills(skill_type, status);


-- ============ Skill 版本表 ============
CREATE TABLE skill_versions (
    version_id      SERIAL PRIMARY KEY,
    skill_id        VARCHAR(64) NOT NULL REFERENCES skills(skill_id),
    version         VARCHAR(32) NOT NULL,      -- 语义化版本 "1.0.0" / "1.1.0-beta.1"
    
    -- 版本内容
    definition      JSONB NOT NULL,            -- 完整的 Skill 执行定义
                                               -- Tool 类: {endpoint, method, params_schema, ...}
                                               -- Workflow 类: {steps: [...], ...}
                                               -- Knowledge 类: {knowledge_base_id, query_template, ...}
    
    system_prompt   TEXT,                      -- Skill 专用 system prompt 片段
    tool_schema     JSONB,                     -- OpenAI function calling 格式的 tool 定义
    
    -- 变更信息
    changelog       TEXT,                      -- 本版本变更说明
    author          VARCHAR(64),               -- 发布者
    
    -- 版本状态
    status          VARCHAR(16) DEFAULT 'draft',   -- draft / canary / stable / deprecated / yanked
    published_at    TIMESTAMPTZ,
    deprecated_at   TIMESTAMPTZ,
    
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    
    UNIQUE(skill_id, version)
);

CREATE INDEX idx_sv_skill ON skill_versions(skill_id, status);
CREATE INDEX idx_sv_published ON skill_versions(published_at);


-- ============ 组织节点 Skill 绑定表（继承 + 覆盖的核心）============
CREATE TABLE org_skill_bindings (
    binding_id      SERIAL PRIMARY KEY,
    node_id         VARCHAR(64) NOT NULL REFERENCES org_nodes(node_id),
    skill_id        VARCHAR(64) NOT NULL REFERENCES skills(skill_id),
    
    -- 绑定策略
    action          VARCHAR(16) NOT NULL,      -- grant / deny / override
                                               -- grant: 授予此 Skill
                                               -- deny: 禁止此 Skill（阻断继承）
                                               -- override: 覆盖父级配置
    
    -- 版本策略
    version_policy  VARCHAR(16) DEFAULT 'stable',  -- stable / canary / pinned / latest
    pinned_version  VARCHAR(32),               -- 仅 version_policy=pinned 时生效
    
    -- 配置覆盖（合并到 Skill 默认配置之上）
    config_override JSONB,                     -- 节点级配置覆盖
                                               -- 例: {"max_results": 50, "default_db": "sales_华东"}
    
    -- 约束
    agent_filter    VARCHAR(64)[],             -- 限定 Agent（NULL = 所有 Agent）
    role_filter     VARCHAR(32)[],             -- 限定角色（NULL = 所有角色）
    
    -- 配额
    quota_daily     INT,                       -- 每日调用上限（NULL = 继承父级 / 无限）
    quota_monthly   INT,                       -- 每月调用上限
    
    -- 时间窗口
    active_from     TIMESTAMPTZ,               -- 生效开始时间（NULL = 立即）
    active_until    TIMESTAMPTZ,               -- 生效结束时间（NULL = 永久）
    
    enabled         BOOLEAN DEFAULT TRUE,
    priority        INT DEFAULT 0,             -- 同节点多条绑定时的优先级
    
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    
    UNIQUE(node_id, skill_id, action)
);

CREATE INDEX idx_osb_node ON org_skill_bindings(node_id);
CREATE INDEX idx_osb_skill ON org_skill_bindings(skill_id);
```

### 2.3 继承解析算法

```python
# resolver/skill_inheritance.py

from dataclasses import dataclass, field
from typing import Optional, List, Dict, Set
import logging

log = logging.getLogger(__name__)


@dataclass
class ResolvedSkill:
    """继承解析后的最终 Skill 描述"""
    skill_id: str
    skill_name: str
    version: str                        # 解析后的具体版本号
    tool_schema: dict                   # 给 LLM 的 tool 定义
    config: dict                        # 合并后的配置
    quota_daily: Optional[int]          # 有效配额
    granted_by: str                     # 哪个节点授予的
    override_chain: List[str]           # 覆盖链路 ["platform", "tenant_001", "dept_华东"]


@dataclass
class InheritanceContext:
    """继承解析的请求上下文"""
    tenant_id: str
    user_id: str
    user_role: str                      # admin / manager / staff / viewer
    department_id: Optional[str]
    team_id: Optional[str]
    agent_id: str
    org_path: str                       # 用户在组织树中的完整路径


class SkillInheritanceResolver:
    """
    组织架构 Skill 继承解析器
    
    算法核心：从根到叶遍历组织路径，逐层合并，后者覆盖前者
    
    类比:
    - CSS: body → .container → .card → #specific (层叠覆盖)
    - Python MRO: 越具体的类越优先
    - 文件系统: /etc/config → ~/.config → ./config (就近覆盖)
    """
    
    def __init__(self, db_pool, cache):
        self.db = db_pool
        self.cache = cache
    
    
    async def resolve(self, ctx: InheritanceContext) -> List[ResolvedSkill]:
        """
        主入口：根据上下文解析最终可用的 Skill 列表
        
        步骤:
        1. 获取用户的组织路径（从根到叶的节点链）
        2. 从根开始，逐层收集 Skill 绑定
        3. 应用继承规则（grant/deny/override）
        4. 解析版本号
        5. 过滤 Agent + 角色 + 时间窗口
        6. 返回最终列表
        """
        
        # ── Step 1: 获取组织路径 ──
        path_nodes = await self._resolve_org_path(ctx)
        # 结果: [platform_node, tenant_node, div_node, dept_node, team_node]
        
        # ── Step 2: 逐层收集并合并 ──
        # 用 dict 存储，key=skill_id，后面的层覆盖前面的
        merged: Dict[str, _SkillAccumulator] = {}
        
        for node in path_nodes:
            bindings = await self._load_node_bindings(node.node_id)
            
            for binding in bindings:
                skill_id = binding["skill_id"]
                action = binding["action"]
                
                if action == "deny":
                    # 禁止：从已继承的集合中移除，并标记为 denied
                    if skill_id in merged:
                        merged[skill_id].denied = True
                        merged[skill_id].denied_by = node.node_id
                    else:
                        merged[skill_id] = _SkillAccumulator(
                            skill_id=skill_id,
                            denied=True,
                            denied_by=node.node_id,
                        )
                    continue
                
                if action == "grant":
                    if skill_id in merged and merged[skill_id].denied:
                        # 父级 deny 了，子级不能 re-grant（安全原则）
                        log.debug(f"Skill {skill_id} denied by {merged[skill_id].denied_by}, "
                                  f"cannot re-grant at {node.node_id}")
                        continue
                    
                    if skill_id not in merged:
                        merged[skill_id] = _SkillAccumulator(skill_id=skill_id)
                    
                    acc = merged[skill_id]
                    acc.granted_by = node.node_id
                    acc.override_chain.append(node.node_id)
                    
                    # 合并配置（浅合并，子级覆盖父级同名 key）
                    if binding.get("config_override"):
                        acc.config.update(binding["config_override"])
                    
                    # 版本策略：子级覆盖父级
                    if binding.get("version_policy"):
                        acc.version_policy = binding["version_policy"]
                    if binding.get("pinned_version"):
                        acc.pinned_version = binding["pinned_version"]
                    
                    # 配额：取最严格的（最小值）
                    if binding.get("quota_daily") is not None:
                        if acc.quota_daily is None:
                            acc.quota_daily = binding["quota_daily"]
                        else:
                            acc.quota_daily = min(acc.quota_daily, binding["quota_daily"])
                    
                    # Agent / 角色过滤：子级收窄（取交集）
                    if binding.get("agent_filter"):
                        if acc.agent_filter is None:
                            acc.agent_filter = set(binding["agent_filter"])
                        else:
                            acc.agent_filter &= set(binding["agent_filter"])
                    
                    if binding.get("role_filter"):
                        if acc.role_filter is None:
                            acc.role_filter = set(binding["role_filter"])
                        else:
                            acc.role_filter &= set(binding["role_filter"])
                
                elif action == "override":
                    # override 只修改配置/版本，不改变 grant/deny 状态
                    if skill_id in merged and not merged[skill_id].denied:
                        acc = merged[skill_id]
                        acc.override_chain.append(f"{node.node_id}(override)")
                        if binding.get("config_override"):
                            acc.config.update(binding["config_override"])
                        if binding.get("version_policy"):
                            acc.version_policy = binding["version_policy"]
                        if binding.get("pinned_version"):
                            acc.pinned_version = binding["pinned_version"]
        
        # ── Step 3: 过滤 denied + 不匹配的 ──
        active_skills = {}
        for skill_id, acc in merged.items():
            if acc.denied:
                continue
            if not acc.granted_by:
                continue
            
            # Agent 过滤
            if acc.agent_filter and ctx.agent_id not in acc.agent_filter:
                continue
            
            # 角色过滤
            if acc.role_filter and ctx.user_role not in acc.role_filter:
                continue
            
            active_skills[skill_id] = acc
        
        # ── Step 4: 解析版本 → 构建最终结果 ──
        results = []
        for skill_id, acc in active_skills.items():
            skill_info = await self._load_skill_info(skill_id)
            if not skill_info or skill_info["status"] != "active":
                continue
            
            version = await self._resolve_version(skill_id, acc.version_policy, acc.pinned_version)
            if not version:
                continue
            
            results.append(ResolvedSkill(
                skill_id=skill_id,
                skill_name=skill_info["skill_name"],
                version=version["version"],
                tool_schema=version["tool_schema"],
                config=acc.config,
                quota_daily=acc.quota_daily,
                granted_by=acc.granted_by,
                override_chain=acc.override_chain,
            ))
        
        return results
    
    
    async def _resolve_org_path(self, ctx: InheritanceContext) -> list:
        """
        解析用户的组织路径：从 platform 根节点到用户所在的最叶子节点
        
        利用 PostgreSQL ltree 的 @> 操作符高效查询
        """
        cache_key = f"org_path:{ctx.tenant_id}:{ctx.department_id}:{ctx.team_id}"
        cached = await self.cache.get(cache_key)
        if cached:
            return cached
        
        # 查询从根到叶的所有祖先节点
        rows = await self.db.fetch("""
            SELECT node_id, node_type, path, depth
            FROM org_nodes
            WHERE path @> (
                SELECT path FROM org_nodes 
                WHERE node_id = $1
            )
            OR path <@ (
                SELECT path FROM org_nodes 
                WHERE node_id = $1
            )
            ORDER BY depth ASC
        """, ctx.team_id or ctx.department_id or ctx.tenant_id)
        
        nodes = [dict(r) for r in rows]
        await self.cache.set(cache_key, nodes, ttl=300)
        return nodes
    
    
    async def _resolve_version(self, skill_id: str, 
                                policy: str, pinned: str = None) -> Optional[dict]:
        """
        根据版本策略解析具体版本
        
        策略:
        - stable: 最新的 status=stable 版本
        - canary: 最新的 canary 版本（灰度）
        - pinned: 精确指定版本号
        - latest: 最新已发布版本（不管状态）
        """
        if policy == "pinned" and pinned:
            return await self.db.fetchrow("""
                SELECT * FROM skill_versions 
                WHERE skill_id = $1 AND version = $2 AND status != 'yanked'
            """, skill_id, pinned)
        
        status_filter = {
            "stable": "stable",
            "canary": "canary",
            "latest": None,
        }.get(policy, "stable")
        
        if status_filter:
            return await self.db.fetchrow("""
                SELECT * FROM skill_versions 
                WHERE skill_id = $1 AND status = $2
                ORDER BY published_at DESC LIMIT 1
            """, skill_id, status_filter)
        else:
            return await self.db.fetchrow("""
                SELECT * FROM skill_versions 
                WHERE skill_id = $1 AND status NOT IN ('draft', 'yanked')
                ORDER BY published_at DESC LIMIT 1
            """, skill_id)
    
    
    async def _load_node_bindings(self, node_id: str) -> list:
        """加载某个组织节点的所有 Skill 绑定（带缓存）"""
        cache_key = f"bindings:{node_id}"
        cached = await self.cache.get(cache_key)
        if cached is not None:
            return cached
        
        rows = await self.db.fetch("""
            SELECT b.*, s.skill_name, s.skill_type, s.status as skill_status
            FROM org_skill_bindings b
            JOIN skills s ON b.skill_id = s.skill_id
            WHERE b.node_id = $1 
              AND b.enabled = TRUE
              AND (b.active_from IS NULL OR b.active_from <= NOW())
              AND (b.active_until IS NULL OR b.active_until >= NOW())
            ORDER BY b.priority DESC
        """, node_id)
        
        result = [dict(r) for r in rows]
        await self.cache.set(cache_key, result, ttl=60)
        return result
    
    
    async def _load_skill_info(self, skill_id: str) -> Optional[dict]:
        """加载 Skill 基本信息（带缓存）"""
        cache_key = f"skill:{skill_id}"
        cached = await self.cache.get(cache_key)
        if cached:
            return cached
        
        row = await self.db.fetchrow(
            "SELECT * FROM skills WHERE skill_id = $1", skill_id
        )
        if row:
            result = dict(row)
            await self.cache.set(cache_key, result, ttl=300)
            return result
        return None


@dataclass
class _SkillAccumulator:
    """内部累积器：逐层合并 Skill 绑定"""
    skill_id: str
    denied: bool = False
    denied_by: Optional[str] = None
    granted_by: Optional[str] = None
    override_chain: list = field(default_factory=list)
    config: dict = field(default_factory=dict)
    version_policy: str = "stable"
    pinned_version: Optional[str] = None
    quota_daily: Optional[int] = None
    quota_monthly: Optional[int] = None
    agent_filter: Optional[Set[str]] = None
    role_filter: Optional[Set[str]] = None
```

### 2.4 继承示例：一条完整的解析链路

```
场景: 华东销售部员工 张三 使用 数据分析Agent，请求 crm_query Skill

组织路径: platform → tenant_acme → div_sales → dept_华东

各层绑定:

┌─ platform ─────────────────────────────────────────────────────────┐
│  grant: web_search (stable, quota=1000/day)                       │
│  grant: weather (stable)                                          │
│  grant: file_reader (stable)                                      │
└───────────────────────────────────────────────┬───────────────────┘
                                                │ 继承 ↓
┌─ tenant_acme ─────────────────────────────────▼───────────────────┐
│  grant: crm_query (stable, config={db: "crm_main"})              │
│  grant: erp_lookup (stable)                                       │
│  grant: approval_flow (stable)                                    │
│  deny:  web_search  ← 企业安全策略：禁止公网搜索                      │
└───────────────────────────────────────────────┬───────────────────┘
                                                │ 继承 ↓
┌─ div_sales ───────────────────────────────────▼───────────────────┐
│  grant: sales_forecast (canary, agent_filter=["data-analyst"])    │
│  override: crm_query (config={default_view: "pipeline"})          │
└───────────────────────────────────────────────┬───────────────────┘
                                                │ 继承 ↓
┌─ dept_华东 ───────────────────────────────────▼───────────────────┐
│  override: crm_query (config={region: "east_china"})              │
│  grant: local_market_report (stable, role_filter=["manager"])     │
└───────────────────────────────────────────────────────────────────┘

张三 (role=staff) 最终可用 Skill:

✅ weather           ← platform 继承
✅ file_reader       ← platform 继承
❌ web_search        ← tenant_acme deny（不可恢复）
✅ crm_query         ← tenant 授予, div 覆盖 config, dept 再覆盖
                       最终 config = {db: "crm_main", default_view: "pipeline", region: "east_china"}
✅ erp_lookup        ← tenant 继承
✅ approval_flow     ← tenant 继承
✅ sales_forecast    ← div 授予 (canary 版本, 仅 data-analyst Agent)
❌ local_market_report ← dept 授予但 role_filter=["manager"], 张三是 staff → 过滤掉
```

---

## 三、Skill 版本迭代体系

### 3.1 版本生命周期

```
                    ┌─────────┐
                    │  draft   │  ← 开发中，仅创建者可见
                    └────┬────┘
                         │ 发布到灰度
                    ┌────▼────┐
                    │ canary   │  ← 灰度阶段，指定节点/用户可用
                    └────┬────┘
                         │ 验证通过，正式发布
                    ┌────▼────┐
                    │ stable   │  ← 生产稳定版，默认使用
                    └────┬────┘
                         │ 有新版本替代
                    ┌────▼────────┐
                    │ deprecated   │  ← 废弃但仍可用（pinned 用户不受影响）
                    └────┬────────┘
                         │ 强制下线
                    ┌────▼────┐
                    │ yanked   │  ← 完全不可用（安全问题等紧急场景）
                    └─────────┘
```

### 3.2 版本解析策略

| 策略 | 说明 | 适用场景 |
|---|---|---|
| `stable` | 自动使用最新 stable 版本 | 默认策略，大多数节点 |
| `canary` | 使用最新 canary 版本 | 灰度验证节点 |
| `pinned` | 锁定到指定版本号 | 关键业务、合规场景 |
| `latest` | 使用最新已发布版本（含 canary） | 测试环境 |

### 3.3 灰度发布流程

```
发布 crm_query v2.0.0 的完整流程:

Step 1: 创建 draft 版本
────────────────────────
POST /admin/skills/crm_query/versions
{
    "version": "2.0.0",
    "definition": { ... 新的执行定义 ... },
    "changelog": "新增: 支持跨区域联合查询; 修复: 大数据量分页问题"
}
→ status = "draft"


Step 2: 发布 canary
────────────────────────
POST /admin/skills/crm_query/versions/2.0.0/promote
{ "target_status": "canary" }

同时，选择灰度节点:
PUT /admin/org/dept_华东/bindings/crm_query
{
    "action": "override",
    "version_policy": "canary"     ← 华东部门先用 canary
}
→ 华东部门用 v2.0.0 (canary), 其他部门继续用 v1.x (stable)


Step 3: 灰度观察（3-7 天）
────────────────────────
监控指标:
- 错误率对比（canary vs stable）
- 响应延迟 P50/P99
- 用户反馈/工单
- Tool calling 成功率

自动化守卫:
- 错误率 > 5% → 自动回滚（yank canary, 灰度节点回退到 stable）
- 延迟 P99 > 2x 基线 → 告警 + 暂停推广


Step 4: 全量发布
────────────────────────
POST /admin/skills/crm_query/versions/2.0.0/promote
{ "target_status": "stable" }

同时:
- v1.x 自动标记为 deprecated
- 所有 version_policy=stable 的节点自动切换到 v2.0.0
- 灰度节点的 override 可以清理（回归 stable 策略）


Step 5: 废弃旧版本（30 天后）
────────────────────────
POST /admin/skills/crm_query/versions/1.3.2/promote
{ "target_status": "yanked" }
→ 即使 pinned 到 v1.3.2 的节点也会被强制回退到最新 stable
→ 发送通知给相关管理员
```

### 3.4 版本管理 API 与数据流

```python
# api/routes/skill_versions.py

from fastapi import APIRouter

router = APIRouter(prefix="/admin/skills", tags=["Skill 版本管理"])


@router.post("/{skill_id}/versions")
async def create_version(skill_id: str, req: CreateVersionRequest):
    """
    创建新版本（draft 状态）
    
    必须包含:
    - version: 语义化版本号（不能与已有版本重复）
    - definition: 完整的执行定义
    - tool_schema: LLM tool calling 格式
    可选:
    - changelog: 变更说明
    - system_prompt: Skill 专用 prompt 片段
    """
    # 校验版本号格式（semver）
    # 校验 definition 完整性
    # 校验 tool_schema 合法性
    # 写入 skill_versions 表，status = "draft"
    ...


@router.post("/{skill_id}/versions/{version}/promote")
async def promote_version(skill_id: str, version: str, req: PromoteRequest):
    """
    版本状态推进
    
    合法的状态转换:
    - draft → canary
    - canary → stable
    - stable → deprecated
    - 任意 → yanked (紧急下线)
    
    不允许:
    - 反向推进（deprecated → stable 请创建新版本）
    - draft → stable（必须经过 canary）
    """
    ...


@router.post("/{skill_id}/versions/{version}/rollback")
async def rollback_version(skill_id: str, version: str):
    """
    回滚到上一个 stable 版本
    
    操作:
    1. 当前 canary/stable 版本 → yanked
    2. 找到上一个 stable 版本 → 恢复为 stable
    3. 清理所有 canary override 绑定
    4. 刷新 Redis 缓存
    5. 记录审计日志
    """
    ...


@router.get("/{skill_id}/versions")
async def list_versions(skill_id: str, status: str = None):
    """列出所有版本"""
    ...


@router.get("/{skill_id}/versions/{version}/diff")
async def diff_versions(skill_id: str, version: str, compare_to: str):
    """
    对比两个版本的差异
    
    返回:
    - definition 的 JSON diff
    - tool_schema 的变更
    - system_prompt 的文本 diff
    """
    ...
```

---

## 四、运行时集成

### 4.1 在 Worker 中集成 Skill 继承解析

```python
# worker/agent_worker.py 中的修改

class AgentWorker:
    
    def __init__(self, ...):
        ...
        # 新增: Skill 继承解析器
        self.skill_resolver = SkillInheritanceResolver(db_pool, redis_cache)
    
    
    async def execute_run(self, task: dict, stream: str, msg_id: str):
        """执行一个 Agent Run（增加动态 Skill 解析）"""
        
        # ... 前面的获取锁、加载 Agent 等步骤不变 ...
        
        # ── 新增: 动态解析可用 Skill ──
        ctx = InheritanceContext(
            tenant_id=task["reply_tenant_id"],
            user_id=task["user_id"],
            user_role=task.get("user_role", "staff"),
            department_id=task.get("department_id"),
            team_id=task.get("team_id"),
            agent_id=task["agent_id"],
            org_path=task.get("org_path", ""),
        )
        
        resolved_skills = await self.skill_resolver.resolve(ctx)
        
        # 构建 Tool 列表给 LLM
        tools = [skill.tool_schema for skill in resolved_skills if skill.tool_schema]
        
        # 构建 Skill 配置映射（Tool 执行时需要）
        skill_configs = {
            skill.skill_id: skill.config 
            for skill in resolved_skills
        }
        
        # ... 后续 LLM 调用时传入 tools ...
        result = await self.run_agent_loop(
            agent, messages, task, trace_id,
            tools=tools,
            skill_configs=skill_configs,
        )
```

### 4.2 缓存策略

```python
# 三级缓存: L1 进程内存 → L2 Redis → L3 PostgreSQL

class SkillResolutionCache:
    """
    缓存策略:
    
    L1 (进程内存, TTL=30s):
        - org_path 到节点列表的映射（组织架构很少变）
        - skill_info 基本信息
    
    L2 (Redis, TTL=60s):
        - 各节点的 binding 列表（绑定偶尔变）
        - 版本解析结果（版本发布时刷新）
    
    L3 (PostgreSQL):
        - 数据源，只有缓存 miss 时才查
    
    缓存失效触发:
        - 管理 API 修改绑定 → 删除对应节点的 L2 缓存
        - 版本发布/回滚 → 删除对应 skill 的 L2 缓存
        - 组织架构变更 → 全量刷新 L1
    """
    
    BINDING_TTL = 60       # 绑定缓存 60 秒
    SKILL_INFO_TTL = 300   # Skill 信息缓存 5 分钟
    ORG_PATH_TTL = 300     # 组织路径缓存 5 分钟
    VERSION_TTL = 120      # 版本解析缓存 2 分钟
    
    async def invalidate_node_bindings(self, node_id: str):
        """绑定变更时调用"""
        await self.redis.delete(f"bindings:{node_id}")
        # 同时失效该节点所有子节点的解析结果缓存
        child_keys = await self.redis.keys(f"resolved:*:{node_id}:*")
        if child_keys:
            await self.redis.delete(*child_keys)
    
    async def invalidate_skill_version(self, skill_id: str):
        """版本发布/回滚时调用"""
        await self.redis.delete(f"skill:{skill_id}")
        # 删除所有包含该 skill 的解析结果缓存
        keys = await self.redis.keys(f"resolved:*")
        if keys:
            await self.redis.delete(*keys)
```

---

## 五、Skill 迭代最佳实践

### 5.1 Skill 定义规范

每个 Skill 版本的 `definition` 必须包含：

```json
{
    "skill_id": "crm_query",
    "version": "2.0.0",
    
    "tool_schema": {
        "type": "function",
        "function": {
            "name": "crm_query",
            "description": "查询 CRM 系统中的客户、商机、合同数据",
            "parameters": {
                "type": "object",
                "properties": {
                    "query_type": {
                        "type": "string",
                        "enum": ["customer", "opportunity", "contract"],
                        "description": "查询类型"
                    },
                    "filters": {
                        "type": "object",
                        "description": "过滤条件"
                    },
                    "limit": {
                        "type": "integer",
                        "default": 20,
                        "description": "返回数量上限"
                    }
                },
                "required": ["query_type"]
            }
        }
    },
    
    "execution": {
        "type": "http",
        "endpoint": "http://crm-service:8080/api/query",
        "method": "POST",
        "auth": {
            "type": "service_account",
            "secret_ref": "crm_service_token"
        },
        "timeout_ms": 10000,
        "retry": {
            "max_attempts": 2,
            "backoff_ms": 1000
        }
    },
    
    "input_transform": {
        "body": {
            "type": "{{query_type}}",
            "filters": "{{filters}}",
            "limit": "{{limit}}",
            "region": "{{config.region}}"
        }
    },
    
    "output_transform": {
        "success": "查询到 {{result.total}} 条记录:\n{{#each result.items}}...\n{{/each}}",
        "error": "查询失败: {{error.message}}"
    },
    
    "test_cases": [
        {
            "name": "基础客户查询",
            "input": {"query_type": "customer", "filters": {"name": "测试"}},
            "expected": {"status": "success", "result.total": "> 0"}
        }
    ]
}
```

### 5.2 Skill 迭代检查清单

```markdown
## 发布新版本前必检

### 兼容性
- [ ] tool_schema 参数变更是否向后兼容？（新增可选参数 ✅，删除必填参数 ❌）
- [ ] 返回格式是否兼容旧版本的下游消费者？
- [ ] 是否更新了 system_prompt 片段？

### 测试
- [ ] test_cases 全部通过？
- [ ] 灰度节点已配置？
- [ ] 灰度期至少 3 天？
- [ ] 错误率基线已记录？

### 安全
- [ ] 新增的 API 调用是否经过安全审查？
- [ ] 敏感数据是否有脱敏处理？
- [ ] 配额限制是否合理？

### 文档
- [ ] changelog 已填写？
- [ ] 如有 breaking change，是否通知了 pinned 版本的管理员？
```

### 5.3 自动化迭代守卫

```python
# guards/version_guard.py

class VersionGuard:
    """
    自动化版本迭代守卫
    与 CI/CD 管道集成，在版本推进时自动校验
    """
    
    async def pre_promote_check(self, skill_id: str, 
                                 from_status: str, to_status: str,
                                 version_data: dict) -> GuardResult:
        """发布前自动检查"""
        
        checks = []
        
        # 1. Schema 兼容性检查
        if to_status in ("canary", "stable"):
            prev_stable = await self.get_current_stable(skill_id)
            if prev_stable:
                compat = self.check_schema_compatibility(
                    prev_stable["tool_schema"],
                    version_data["tool_schema"]
                )
                checks.append({
                    "name": "schema_compatibility",
                    "passed": compat.is_compatible,
                    "detail": compat.breaking_changes,
                })
        
        # 2. 测试用例执行
        if version_data.get("test_cases"):
            test_result = await self.run_test_cases(
                version_data["definition"],
                version_data["test_cases"]
            )
            checks.append({
                "name": "test_cases",
                "passed": test_result.all_passed,
                "detail": f"{test_result.passed}/{test_result.total} passed",
            })
        
        # 3. canary → stable 需要灰度数据
        if from_status == "canary" and to_status == "stable":
            canary_metrics = await self.get_canary_metrics(skill_id, version_data["version"])
            checks.append({
                "name": "canary_metrics",
                "passed": canary_metrics.error_rate < 0.05 and canary_metrics.sample_size > 100,
                "detail": f"error_rate={canary_metrics.error_rate:.2%}, "
                          f"samples={canary_metrics.sample_size}",
            })
        
        all_passed = all(c["passed"] for c in checks)
        return GuardResult(passed=all_passed, checks=checks)
    
    
    def check_schema_compatibility(self, old_schema: dict, new_schema: dict) -> CompatResult:
        """
        检查 tool_schema 兼容性
        
        允许（向后兼容）:
        - 新增可选参数
        - 放宽参数约束（enum 增加选项等）
        - 新增返回字段
        
        禁止（breaking change）:
        - 删除或重命名参数
        - 新增必填参数
        - 收窄参数约束
        - 改变参数类型
        """
        breaking = []
        
        old_params = old_schema.get("function", {}).get("parameters", {}).get("properties", {})
        new_params = new_schema.get("function", {}).get("parameters", {}).get("properties", {})
        old_required = set(old_schema.get("function", {}).get("parameters", {}).get("required", []))
        new_required = set(new_schema.get("function", {}).get("parameters", {}).get("required", []))
        
        # 删除的参数
        for param in old_params:
            if param not in new_params:
                breaking.append(f"参数 '{param}' 被删除")
        
        # 新增的必填参数
        added_required = new_required - old_required
        for param in added_required:
            if param not in old_params:
                breaking.append(f"新增必填参数 '{param}'")
        
        # 类型变更
        for param in old_params:
            if param in new_params:
                if old_params[param].get("type") != new_params[param].get("type"):
                    breaking.append(f"参数 '{param}' 类型变更: "
                                    f"{old_params[param].get('type')} → {new_params[param].get('type')}")
        
        return CompatResult(
            is_compatible=len(breaking) == 0,
            breaking_changes=breaking,
        )
```

---

## 六、审计与可观测

### 6.1 Skill 调用审计

```sql
-- Skill 调用日志表（补充到现有 message_logs 或独立表）
CREATE TABLE skill_invocation_logs (
    id              BIGSERIAL PRIMARY KEY,
    trace_id        VARCHAR(64) NOT NULL,
    session_id      VARCHAR(128),
    
    -- 调用方
    tenant_id       VARCHAR(64) NOT NULL,
    user_id         VARCHAR(128),
    agent_id        VARCHAR(64),
    
    -- Skill 信息
    skill_id        VARCHAR(64) NOT NULL,
    skill_version   VARCHAR(32) NOT NULL,
    granted_by      VARCHAR(64),           -- 授予此 Skill 的组织节点
    
    -- 调用详情
    input_params    JSONB,                 -- 输入参数（脱敏后）
    output_summary  TEXT,                  -- 输出摘要（截断）
    config_snapshot JSONB,                 -- 当时的合并配置快照
    
    -- 结果
    status          VARCHAR(16),           -- success / error / timeout / quota_exceeded
    error_detail    TEXT,
    latency_ms      INT,
    
    -- 继承追溯
    override_chain  JSONB,                 -- 完整的继承覆盖链路
    
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_sil_tenant ON skill_invocation_logs(tenant_id, created_at);
CREATE INDEX idx_sil_skill ON skill_invocation_logs(skill_id, skill_version, created_at);
CREATE INDEX idx_sil_trace ON skill_invocation_logs(trace_id);
```

### 6.2 继承可视化 API

```python
@router.get("/org/{node_id}/effective-skills")
async def get_effective_skills(node_id: str, agent_id: str = None, role: str = None):
    """
    查看某个组织节点的最终有效 Skill 列表
    
    返回每个 Skill 的完整继承链路，方便排查 "为什么我看不到某个 Skill"
    """
    ...
    return {
        "node_id": node_id,
        "node_path": "platform → tenant_acme → div_sales → dept_华东",
        "effective_skills": [
            {
                "skill_id": "crm_query",
                "version": "2.0.0",
                "status": "active",
                "inheritance_chain": [
                    {"node": "platform", "action": "-", "note": "未配置"},
                    {"node": "tenant_acme", "action": "grant", "config": {"db": "crm_main"}},
                    {"node": "div_sales", "action": "override", "config": {"default_view": "pipeline"}},
                    {"node": "dept_华东", "action": "override", "config": {"region": "east_china"}},
                ],
                "final_config": {"db": "crm_main", "default_view": "pipeline", "region": "east_china"},
                "quota": {"daily": 500, "used_today": 123},
            },
            {
                "skill_id": "web_search",
                "status": "denied",
                "denied_by": "tenant_acme",
                "reason": "企业安全策略禁止公网搜索",
            },
        ]
    }


@router.get("/org/{node_id}/inheritance-tree")
async def get_inheritance_tree(node_id: str):
    """
    可视化组织架构 Skill 继承树
    返回从 platform 到指定节点的完整继承视图
    """
    ...
```

---

## 七、与网关主架构的集成点

```
已有架构文档               →  本文档的集成点
──────────────────────────────────────────────────────
消息处理管道               →  authenticate_user() 时补充 org_path, department_id
  (message_pipeline.py)       到 UnifiedMessage.metadata 中

Agent Router              →  路由到 Agent 后, Worker 用 SkillInheritanceResolver
  (agent_router.py)           解析该 Agent + 该用户的可用 Skill

Worker 执行层              →  run_agent_loop() 的 tools 参数改为动态解析
  (agent_worker.py)           而非从 Agent 配置中静态读取

回复投递层                 →  无变更
  (reply_dispatcher.py)

Agent 配置                 →  agents.tools 字段从 ["sql_query", ...] 静态列表
  (agents 表)                 改为可选的 Skill 白名单过滤（与继承体系配合）

管理 API                   →  新增 /admin/skills/*, /admin/org/*/bindings 等接口
```

---

## 八、与 OpenClaw 对比总结

| 维度 | OpenClaw (ToC) | 本方案 (ToB) |
|---|---|---|
| 继承层级 | 6 层目录层叠（文件系统） | N 层组织树（数据库 ltree） |
| 覆盖粒度 | 同名 Skill 整体替换 | 字段级合并 (config, version, quota) |
| deny 语义 | `enabled: false`（可随时恢复） | `deny` action（子级不可恢复，安全硬阻断） |
| 版本管理 | 无版本概念（文件就是版本） | draft → canary → stable → deprecated → yanked |
| 灰度发布 | 不支持 | canary 版本 + 节点级 override |
| 配额计量 | 不支持 | 节点级 quota + 实时计量 |
| 审计追溯 | 日志文件 | 完整调用记录 + 继承链路快照 |
| 缓存失效 | 重启 / 快照刷新 | 事件驱动精确失效 |
