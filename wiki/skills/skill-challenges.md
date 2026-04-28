# 企业级 Skill 管理挑战汇总

> 基于 [[skills/skill-manage.md]] 的中台方案提炼，覆盖上下文工程 / 多租户 / 安全 / 自进化四个主要挑战域
> 更新日期: 2026-04-22

## 一句话定位
从 Hermes 单用户本地 Skill 到中台多租户 Skill 体系，要跨越**上下文预算、权限边界、运行时安全、自进化治理**四道鸿沟。本文梳理每项挑战的本质、难点、中台方案、残余风险。

---

## 挑战全景

| # | 挑战域 | 具体问题 | 方案 | 残余风险 |
|---|---|---|---|---|
| 1 | 上下文工程 | Skill 多了 token 预算爆 | 三级过滤链 #13→#5→#4 | LLM 选择疲劳，Level 0 索引描述质量依赖作者 |
| 2 | 多租户权限 | 既要组织共享，又要个人私有，还要避免深层继承混乱 | 两级继承（个人 + 直接父节点） | 跨部门协作时 Skill 复制冗余 |
| 3 | Skill 演进 | Agent 自由迭代 vs ToB 审核门控 | 草稿→审核→发布 + 沙箱评估 | 审核人成瓶颈；自动放行边界判定难 |
| 4 | 版本管理 | 出问题要秒级回滚；历史版本要永久可追溯 | current_version 指针 + OSS 版本对象永久保留 | OSS 存储成本长期累积 |
| 5 | 运行时安全 | 凭证不能进 LLM 进程；注入攻击要拦截 | MCP Manager-Worker 分离 + 三重扫描 | Gateway 本身成为新靶点 |
| 6 | 配额与降级 | 高峰期配额耗尽，高配 Skill 需自动降级 | `skill_fallbacks` 链 + `skill_quotas` 表 | 降级链过长影响用户体验 |
| 7 | 分布式一致性 | 多实例 Worker + PVC 挂载可能有同步延迟 | Redis 主动失效 + 进程内 LRU 按 skill_id 清除 | PVC CSI 驱动缓存行为依赖实现 |
| 8 | MCP 与 Skill 协同 | 原子工具 vs 场景编排，边界模糊 | MCP 管权限治理，Skill 管场景演进 | 初期命名规范易乱，治理责任需说清 |

---

## 一、上下文工程挑战

### 问题本质

一个 Agent 可能绑定几十上百个 Skill，但 LLM 的 system prompt token 预算有限（典型 32K 里 Skill 索引只能分 ~1.5K）。Skill 数量增长是线性的，token 预算是刚性的。

### 难点

- 不能"一刀切"限制 Skill 数量（企业场景确实需要很多工具）
- 不能全部 Level 0 注入（几百个 name+desc 就爆了）
- 不能让 LLM 看到用不了的 Skill（选择疲劳 + 误调用）

### 中台方案：三级过滤链

```
#13 两级继承  → 权限边界（用户可见的 50 个）
    ↓
#5 条件激活   → 场景相关（当前适用的 15 个）
    ↓
#4 渐进加载   → token 压缩（Level 0 ~1500 token）
```

详见 [[context/context-management.md]] §Skill 索引的上下文工程职责。

### 残余风险

- **Level 0 描述质量**：description 字段由 Skill 作者撰写，写得含糊会影响 LLM 的选择准确度
- **条件激活规则冲突**：同时配多个 fallback / feature_flag 可能产生意外过滤
- **选择疲劳**：即使过滤后有 15 个 Skill，LLM 仍可能在相似 Skill 间摇摆

---

## 二、多租户权限边界

### 问题本质

企业级要求：
- 平台级 Skill（所有租户可用）
- 集团级 Skill（仅本企业可用）
- 团队级 Skill（仅某部门可用）
- 个人级 Skill（仅本人可用）

同一个 Skill 名字在不同层级可能有不同实现，需要清晰的覆盖规则。

### 难点

- 深层组织树（平台→集团→BU→部门→团队→人）继承语义复杂，用户难以预期"我能看到哪些"
- ltree 祖先查询需要 PostgreSQL 扩展，不是所有环境都支持
- 跨部门协作场景：A 部门想用 B 部门的 Skill，如何既不违反隔离又能协作？

### 中台方案：两级继承

简化为**只继承用户的直接父节点**（团队/部门），不做深层传递：

```
个人 Skill (personal) ← 用户本人
    ↑ override 同名时优先
直接父节点 (public)    ← 团队/部门
```

平台级 / 租户级通用 Skill 由管理员**批量创建到各团队节点**，省去运行时的深层遍历。查询只需 B-tree 索引即可命中。

详见 [[skills/skill-manage.md]] #13 两级继承。

### 残余风险

- **Skill 冗余**：平台级 Skill 批量复制到每个团队节点，存储翻倍
- **跨团队协作**：用户 A 想用 B 团队的 Skill 必须显式复制到 A 团队，不能临时借用
- **节点变更成本**：用户调岗后，原团队 Skill 不再可见（符合权限模型，但用户可能不理解）

---

## 三、Skill 演进与质量

### 问题本质

- Hermes 模式：Agent 自由创建 / 修改 / 删除 Skill，质量靠"用坏了自己改"
- ToB 场景：出错代价高，必须有人审核；但审核慢又扼杀 Agent 的自进化潜力

### 难点

- 怎么评估"新版本比旧版本好"？靠人工读 diff 不现实
- Agent 产出的 Skill 草稿质量参差，审核人负担重
- 审核通过后仍有可能退化，怎么持续监控？

### 中台方案

**三道关卡**：

1. **提议门（#15）**：Agent 只能提议，不能直接写。四类触发条件（complex_task_completed / error_recovery / user_correction / repeated_workflow）过滤噪声
2. **沙箱评估（#8）**：拉起独立 K8s Namespace，用脱敏后的历史调用重放对比新旧版本，6 项指标（success_rate / tool_calls / tokens / latency / pitfall_hits / security_incidents）
3. **自动放行规则**：所有指标不劣化 + 至少一项改进 → `auto_approvable=true`，Owner 审核界面显示"建议批准"

### 残余风险

- **审核人瓶颈**：即使有 auto_approvable 辅助，大批量 Skill 更新仍可能积压
- **沙箱代表性**：脱敏历史数据未必覆盖新 corner case
- **指标盲区**：选 6 项指标时可能漏掉关键维度（如"用户满意度"无法沙箱评估）

---

## 四、版本管理与回滚

### 问题本质

出问题要秒级回滚；但又不能删除历史版本（审计 / 合规）。

### 中台方案

- **current_version 指针**：`skills.current_version` 字段指向当前活跃版本
- **OSS 对象永久保留**：所有版本的 SKILL.md + 子文件都存 OSS，按 `v{version}` 子目录组织
- **回滚 = 改指针**：一行 SQL `UPDATE skills SET current_version=...` 即刻生效，配合 Redis 失效秒级传播

```
发布新版本  → OSS 写入 + skill_versions 新增一行 + 更新 current_version 指针
回滚旧版本  → UPDATE skills SET current_version = '1.2.0' WHERE skill_id = ...
```

详见 [[skills/skill-manage.md]] #14 版本管理。

### 残余风险

- **OSS 长期成本**：所有版本永久保留，累积存储费用
- **版本兼容**：回滚后，依赖新版本新增字段的 Agent 可能报错
- **缓存一致性**：Redis DEL + LRU clear 如果某一环节漏了，会出现新旧版本混用

---

## 五、运行时安全

### 问题本质

Skill 的运行时风险有三类：
1. **凭证泄露**：API Key / Token 一旦进入 LLM 进程就可能被 Prompt 注入导出
2. **注入攻击**：SKILL.md 内容本身可能被植入恶意指令
3. **恶意代码执行**：`scripts/` 目录下脚本可能含 `rm -rf` / `curl | bash`

### 中台方案

**分层防御**：

| 层 | 方案 | 对应章节 |
|---|---|---|
| 凭证层 | MCP Manager-Worker 分离，Worker 拿 consumer token 代替真实凭证 | [[应用网关设计方案]] |
| 静态扫描 | 创建 + 安装时阻断（10+ 种注入模式 / 危险命令 / 硬编码凭证 / 隐形 Unicode） | #12 安全扫描 |
| 运行时校验 | 每次 load_skill 校验 OSS 对象 SHA256 防篡改 | #12 |
| 定期审计 | 每日扫所有 current_version，发现新注入模式追溯预警 | #12 |
| 沙箱执行 | 高 risk_level 的 Skill 强制在独立 Namespace 跑 | #8 |

### 残余风险

- **Gateway 单点**：MCP Gateway 本身成了新的高价值靶点，一旦被攻破后果比单 Worker 严重
- **扫描器覆盖**：新型注入模式需要扫描规则持续更新
- **人肉社工**：管理员上传 SKILL.md 时被诱导接受恶意内容，扫描器未必拦得住

---

## 六、配额与降级

### 问题本质

- 高配 Skill（`premium-crm`）价格贵，需要配额限制
- 配额耗尽 / 用户未授权 / 非工作时间，需要自动降级到备选 Skill

### 中台方案

- **`skill_quotas` 表**：per_user_daily / per_user_hourly / per_tenant_daily / burst_limit 四档限流
- **`skill_fallbacks` 表**：显式声明降级链（主 Skill 不可用时启用备选）
- **3 个典型场景**：
  - 配额耗尽：premium → basic
  - 非工作时间：workday-support → off-hour-support
  - 租户未授权：enterprise-bi → community-bi

详见 [[skills/skill-manage.md]] #9 降级 + #10 凭证。

### 残余风险

- **降级链过长**：主 → fallback1 → fallback2 → ... 用户体验跳变
- **配额精度**：per_user_daily 是按天重置，还是按滚动窗口？不同算法边界行为不同
- **突发限流**：burst_limit 的阈值调低了影响体验，调高了失去保护意义

---

## 七、分布式缓存一致性

### 问题本质

中台是多 Worker 实例部署，每个实例都有进程内 LRU。Skill 更新后，如何保证所有实例同步看到新版本？

### 中台方案

**三层缓存 + 失效策略**：

| 层 | 失效触发 | 失效范围 |
|---|---|---|
| Level 0 索引（Redis） | 发布 / 回滚 / 激活规则改 | DEL 相关 tenant 的 key |
| Level 1 SKILL.md（进程内 LRU） | 同上 | 按 skill_id 从所有 Worker 实例清除（广播失效） |
| Level 2 子文件（PVC 挂载） | OSS 对象更新 | CSI 驱动同步（可能有秒级延迟） |

### 残余风险

- **LRU 广播失效成本**：Worker 实例多的时候广播消息量大
- **PVC 挂载同步延迟**：不同 CSI 驱动（阿里云 OSS / MinIO / s3fs）的缓存行为不一致
- **失效漏配**：改了 `skill_quotas` 忘了 DEL 索引 key，会有短暂不一致

---

## 八、MCP 与 Skill 协同

### 问题本质

MCP（原子工具）和 Skill（场景编排）容易混淆：
- "查 CRM 客户"是 MCP 工具还是 Skill？
- 同一能力既写成 MCP 又写成 Skill 会不会冗余？

### 中台方案

**定位边界**（见 [[应用网关设计方案]]）：

| 层 | 职责 | 粒度 | 治理重点 |
|---|---|---|---|
| MCP 工具 | 单个 API 端点 | 细粒度 | 权限 + 凭证 + 速率限制 |
| Skill | 多工具编排 + 场景知识 | 场景级 | 演进 + 审核 + 沙箱 |

**协同链路**：

```
原始 API（curl / Swagger）
    ↓ Manager 批量转换
MCP 工具（Gateway 托管凭证）
    ↓ Worker 首次调用
自动生成 SKILL.md（个人 Skill，含工具使用示例）
    ↓ 提议提升
审核通过 → 公共 Skill（复用 MCP 工具，不重复造轮子）
```

### 残余风险

- **命名冲突**：MCP 工具叫 `get_customer`，Skill 也叫 `get-customer`，用户可能困惑
- **变更协同**：MCP 工具接口改了，所有依赖它的 Skill 需要同步更新
- **权限下游传递**：用户对 Skill 有权限，但 Skill 调用的 MCP 工具没权限，报错逻辑需要清晰

---

## 上下游依赖

- 核心方案: [[skills/skill-manage.md]]
- 上下文工程视角: [[context/context-management.md]]
- 凭证与 MCP: [[应用网关设计方案]]
- 安全体系: [[security/agent-security.md]]
- 时序流程: [[flows/skill-loading.md]]
