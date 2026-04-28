# 收录日志

---

## 2026-04-20 — 首次全量收录（15 个文件）

**来源**: `raw/` 全部已跟踪文件（初始提交，wiki/ 目录之前为空）

**新建模块**: 全部为新建

| 收录文件 | 归属模块 | 新建/更新页面 |
|---|---|---|
| 中台网关.md | `wiki/architecture/` | [[gateway.md]] 新建 |
| 智能体路由.md | `wiki/architecture/` | [[agent-router.md]] 新建 |
| hermes-agent.md | `wiki/architecture/` | [[hermes.md]] 新建 |
| deerflow-agent执行.md | `wiki/architecture/` | [[deerflow.md]] 新建 |
| Agent 安全体系设计.md | `wiki/security/` | [[agent-security.md]] 新建 |
| 记忆管理.md | `wiki/memory/` | [[memory-management.md]] 新建 |
| deerflow-记忆.md | `wiki/memory/` | [[deerflow-memory.md]] 新建 |
| deerflow-记忆管理.md | `wiki/memory/` | [[deerflow-memory.md]] 补充 |
| hermes-记忆.md | `wiki/memory/` | [[hermes-memory.md]] 新建 |
| 上下文管理.md | `wiki/context/` | [[context-management.md]] 新建 |
| deerflow-上下文管理.md | `wiki/context/` | [[deerflow-context.md]] 新建 |
| skill迭代.md | `wiki/skills/` ⭐新模块 | [[skill-iteration.md]] 新建 |
| hermes-skill管理.md | `wiki/skills/` | [[hermes-skills.md]] 新建 |
| Skill 迭代设计 — Hermes Agent 参考对标.md | `wiki/skills/` | [[hermes-skills.md]] 补充渐进式加载章节 |
| hermes-QA.md | `wiki/QA.md` ⭐新文件 | 迁移 14 条 Q&A，合并为 3 个主题条目 |

**新建模块**: `wiki/architecture/`、`wiki/security/`、`wiki/memory/`、`wiki/context/`、`wiki/skills/`

---

## 2026-04-22 — HiClaw MCP 管理 + Skill 挑战汇总

| 收录文件 | 归属模块 | 新建/更新页面 |
|---|---|---|
| HiClaw 1.0.6：企业级 MCP Server 管理 — 凭证零暴露，工具全接入.md | `wiki/architecture/` | [[应用网关设计方案]] 新建（中台设计页） |
| —（基于现有 wiki 汇总） | `wiki/skills/` | [[skill-challenges.md]] 新建 — 8 项挑战域汇总 |

**重点**：
- HiClaw 提供 Manager-Worker 分离 + AI Gateway 凭证代理模式，凭证零暴露
- MCP 与 Skill 分层协同：MCP 管权限，Skill 管场景
- Skill 挑战汇总涵盖上下文工程 / 多租户 / 安全 / 自进化四个主要问题域，每项含残余风险

---

## 2026-04-24 — MCP 管理融合 Anthropic 行业实践

| 收录文件 | 归属模块 | 新建/更新页面 |
|---|---|---|
| Building agents that reach production systems with MCP.md（Anthropic Claude 团队博客） | `wiki/architecture/` | [[应用网关设计方案]] 补充 4 个章节 |

**补充内容**：
- 新增"为什么选 MCP"：Direct API / CLI / MCP 三路径对比 + M×N 集成问题
- 新增"MCP Server 设计模式"：远程优先 / 按 intent 分组 / code orchestration（Cloudflare 2 工具覆盖 2500 端点）/ 富语义（MCP Apps + Elicitation）
- 动态凭证章节补"行业对照"：CIMD + Claude Managed Agents Vaults 与 HiClaw Gateway 同构
- 新增"Client 侧上下文优化"：Tool Search（-85% token）+ Programmatic Tool Calling（-37% token）
- Skill 协同补"Plugin 打包分发" 和 "从 MCP Server 分发 Skill"（MCP 社区 experimental-ext-skills 扩展）

### 同日新增 `wiki/insights/` 综合洞察模块

| 新建文件 | 内容 |
|---|---|
| `wiki/insights/综合问题.md` ⭐新模块 | Skill × MCP × Agent 三者区别（首条） |

**规则确立**：CLAUDE.md 新增"分类不明确的问答归入 `wiki/insights/综合问题.md`"规则，INDEX.md 增加"综合洞察"模块入口。
