# Wiki 索引

| 模块 | 路径 | 页面 | 说明 |
|---|---|---|---|
| 系统架构 | `wiki/architecture/` | [[gateway.md]] | 企业级多智能体网关（渠道适配/路由/队列/Worker） |
| 系统架构 | `wiki/architecture/` | [[agent-router.md]] | Agent 路由引擎（五级优先级规则） |
| 系统架构 | `wiki/architecture/` | [[hermes.md]] | Hermes Agent 框架（自我改进/RL/Skills） |
| 系统架构 | `wiki/architecture/` | [[deerflow.md]] | DeerFlow Agent（LangChain 工厂/多 Agent 协调） |
| 系统架构 | `wiki/architecture/` | [[decision-log.md]] | 架构决策日志（ADR 汇总） |
| 安全 | `wiki/security/` | [[agent-security.md]] | Agent 安全体系（8 层纵深防御，威胁全景） |
| 记忆机制 | `wiki/memory/` | [[memory-management.md]] | AI 中台记忆管理总设计（三层存储/衰减/巩固） |
| 记忆机制 | `wiki/memory/` | [[deerflow-memory.md]] | DeerFlow 记忆实现（JSON + 防抖异步队列） |
| 记忆机制 | `wiki/memory/` | [[hermes-memory.md]] | Hermes 记忆系统（冻结快照/插件化 Provider） |
| 上下文 | `wiki/context/` | [[context-management.md]] | AI 中台上下文管理（组装/压缩/成本控制） |
| 上下文 | `wiki/context/` | [[deerflow-context.md]] | DeerFlow 上下文三层模型（异步/渐进/原子） |
| Skills | `wiki/skills/` | [[skill-iteration.md]] | Skill 继承体系与版本迭代（ToB 企业级方案） |
| Skills | `wiki/skills/` | [[hermes-skills.md]] | Hermes Skill 管理（自主创建/条件激活/安全扫描） |
| 框架对比 | `wiki/` | [[comparison.md]] | Hermes / OpenClaw / DeerFlow 九维度横向对比 |
| 问答 | `wiki/` | [[QA.md]] | 历史提问记录（MCP 机制/Skill 加载/凭证管理） |
| 收录日志 | `wiki/` | [[log.md]] | 资料收录历史 |

## 快速入口

- 想了解**整体架构** → [[architecture/gateway.md]] + [[architecture/decision-log.md]]
- 想了解**安全风险** → [[security/agent-security.md]]
- 想了解**记忆设计** → [[memory/memory-management.md]]
- 想了解**Skill 系统** → [[skills/skill-iteration.md]] + [[skills/hermes-skills.md]]
- 想了解**具体实现对比** → Hermes([[architecture/hermes.md]]) vs DeerFlow([[architecture/deerflow.md]])
