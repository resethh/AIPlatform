# Wiki 索引

## 模块目录

| 模块 | 路径 | 页面 | 核心内容 |
|---|---|---|---|
| 系统架构 | `wiki/architecture/` | [[gateway.md]] | 企业级多智能体网关：渠道适配 / 消息队列 / 路由 / Worker 执行集群 |
| 系统架构 | `wiki/architecture/` | [[agent-router.md]] | Agent 路由引擎：五级优先级规则（显式 > Session > mention > 渠道 > 关键词） |
| 系统架构 | `wiki/architecture/` | [[hermes.md]] | Hermes Agent 框架：自我改进 / RL 训练 / Skill 系统 / 40+ 工具 |
| 系统架构 | `wiki/architecture/` | [[deerflow.md]] | DeerFlow Agent：LangGraph StateGraph / Lead + 子 Agent 并发 / 异步记忆 |
| 系统架构 | `wiki/architecture/` | [[deerflow-agent.md]] | DeerFlow Agent 执行架构：13 层中间件 / 子 Agent 线程池 / Skill 渐进加载实现 |
| 系统架构 | `wiki/architecture/` | [[decision-log.md]] | 架构决策日志（ADR 汇总）：记录每个关键设计的背景与取舍 |
| 系统架构 | `wiki/architecture/` | [[mcp-management.md]] | MCP Server 管理方案：为何选 MCP / Server 设计模式 / Manager-Worker + Gateway 凭证零暴露 / Client 上下文优化（融合 HiClaw 与 Anthropic 实践） |
| 安全 | `wiki/security/` | [[agent-security.md]] | Agent 安全体系：8 层纵深防御，覆盖 Prompt 注入 / 供应链 / 权限 / 审计 |
| 记忆机制 | `wiki/memory/` | [[memory-management.md]] | AI 中台记忆总设计：三层存储 / 时间衰减 / 睡眠巩固 / 跨 Session 持久化 |
| 记忆机制 | `wiki/memory/` | [[deerflow-memory.md]] | DeerFlow 记忆实现：facts JSON + 防抖异步队列 + 时间衰减权重 |
| 记忆机制 | `wiki/memory/` | [[hermes-memory.md]] | Hermes 记忆系统：冻结快照 / 插件化 Provider（Honcho/Mem0）/ 字符上限控制 |
| 上下文 | `wiki/context/` | [[context-management.md]] | AI 中台上下文管理：System Prompt 组装 / 滑动窗口压缩 / Token 预算控制 |
| 上下文 | `wiki/context/` | [[deerflow-context.md]] | DeerFlow 上下文三层模型：异步更新 / 渐进加载 / 原子状态切换 |
| Skills | `wiki/skills/` | [[skill-manage.md]] | AI 中台 Skill 管理：Hermes 核心机制 + ToB 改造（PostgreSQL+OSS 存储 / 组织树继承 / 审核发布 / 6 级信任 Hub） |
| Skills | `wiki/skills/` | [[skill-challenges.md]] | 企业级 Skill 管理挑战汇总：上下文工程 / 多租户 / 安全 / 自进化 等 8 项挑战与残余风险 |
| 时序流程 | `wiki/flows/` | [[message-pipeline.md]] | 消息全链路时序：用户发消息 → Gateway → 路由 → Worker → Agent Loop → 回复 |
| 时序流程 | `wiki/flows/` | [[agent-loop.md]] | Agent 执行循环：Hermes 自定义 Loop vs DeerFlow LangGraph，工具调度与状态管理 |
| 时序流程 | `wiki/flows/` | [[skill-loading.md]] | Skill 渐进加载：Hermes load_skill meta-tool vs DeerFlow read_file，token 节省对比 |
| 时序流程 | `wiki/flows/` | [[multi-agent.md]] | 多 Agent 协调：Lead 委派 → SubagentExecutor 并发（最多 3）→ 结果汇总 |
| 时序流程 | `wiki/flows/` | [[memory-lifecycle.md]] | 记忆生命周期：Session 启动注入 → 防抖异步更新 → 结束巩固 → 睡眠深度整合 |
| 框架对比 | `wiki/` | [[comparison.md]] | Hermes / OpenClaw / DeerFlow 九维度横向对比 + 中台选型建议 |
| 问答归档 | `wiki/` | [[用户问题.md]] | 完整问答归档（问题 + 回答 + 相关页面），按时间顺序追加 |
| 提问时间线 | `wiki/` | [[QA.md]] | 只记录用户问题 + 日期的骨架索引（完整答案在 用户问题.md） |
| 收录日志 | `wiki/` | [[log.md]] | 资料收录历史 |

---

## 模块依赖关系

```
用户消息（IM / CLI）
      ↓
  [Gateway]  ──────────────────────── 渠道适配 / 消息队列 / 优先级
      ↓
  [Agent Router]  ─────────────────── 5 级优先级路由
      ↓
  [AgentWorker]  ──────────────────── 执行编排
      ↓
  [Agent Loop]  ───────────────────── Hermes / DeerFlow
   ↙       ↘
[Memory]   [Skill Manager]  ───────── 渐进加载 Level 0/1/2
   ↕             ↕
[记忆快照]   [工具执行]  ──────────── LLM API / 业务工具
```

跨切面依赖：
- `Agent Security` — 覆盖 Gateway / Agent Router / Skill Manager / Agent Loop 全链路
- `Context Management` — 由 AgentWorker 触发，组装 System Prompt 注入 Agent Loop

---

## 快速入口

**了解整体架构**
- 骨架流程 → [[flows/message-pipeline.md]]
- 组件设计 → [[architecture/gateway.md]] + [[architecture/agent-router.md]]
- 决策背景 → [[architecture/decision-log.md]]

**深入 Agent 内部**
- Hermes（自我改进型）→ [[architecture/hermes.md]]
- DeerFlow 框架概述 → [[architecture/deerflow.md]]
- DeerFlow 执行细节（中间件/子 Agent/Skill）→ [[architecture/deerflow-agent.md]]
- 框架横向对比 → [[comparison.md]]

**了解记忆与上下文**
- 中台总设计 → [[memory/memory-management.md]] + [[context/context-management.md]]
- DeerFlow 实现细节 → [[memory/deerflow-memory.md]] + [[context/deerflow-context.md]]
- Hermes 实现细节 → [[memory/hermes-memory.md]]

**了解 Skill 体系**
- 中台完整方案（Hermes 机制 + ToB 改造）→ [[skills/skill-manage.md]]
- 企业级挑战汇总（8 项挑战 + 残余风险）→ [[skills/skill-challenges.md]]
- MCP Server 管理（凭证零暴露架构）→ [[architecture/mcp-management.md]]

**了解安全**
- 威胁全景 + 防御体系 → [[security/agent-security.md]]

**用户问答记录**
- 完整问答归档（问题 + 回答）→ [[用户问题.md]]
- 提问时间线（只有问题）→ [[QA.md]]

**查看系统时序图**
- 消息全链路（骨架）→ [[flows/message-pipeline.md]]
- Agent 执行循环 → [[flows/agent-loop.md]]
- Skill 渐进加载 → [[flows/skill-loading.md]]
- 多 Agent 协调 → [[flows/multi-agent.md]]
- 记忆生命周期 → [[flows/memory-lifecycle.md]]
