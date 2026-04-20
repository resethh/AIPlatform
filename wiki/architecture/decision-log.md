# 架构决策日志 (ADR)

按时间倒序记录关键架构决策。

---

## ADR-005: Hermes 系统提示冻结快照 (2026-04-16)
**决策**: Session 启动时固定系统提示快照，中途记忆写入不生效。
**背景**: LLM 前缀缓存（Prompt Cache）依赖系统提示稳定，频繁变化导致缓存失效、token 成本飙升。
**后果**: 新记忆需等到下次 Session 才能注入；但每次 Session 的 LLM 调用前缀缓存命中率显著提升。
**出处**: [[memory/hermes-memory.md]]

---

## ADR-004: ToB Skill 继承采用树形级联 + 就近覆盖 (2026-04-14)
**决策**: 组织架构映射为树形节点，Skill 自动向下继承，子节点可 override 同名 Skill，修改不向上传播。
**背景**: ToB 场景需要集团统一管控基础 Skill，同时允许 BU/部门灵活定制，类比 CSS 级联规则。
**后果**: 运行时解析需遍历 org_path（ltree 物化路径），需做缓存；灵活性好但继承链过深时调试复杂。
**出处**: [[skills/skill-iteration.md]]

---

## ADR-003: 接入层与执行层通过 Redis Streams 解耦 (2026-04-05)
**决策**: Webhook 同步响应 ≤40ms 后立即返回，Agent 执行异步通过 Redis Streams 队列进行。
**背景**: 各 IM 平台 Webhook 有严格超时限制（飞书 3 秒），LLM 调用通常需要 2-10 秒，同步处理必然超时。
**后果**: 用户体验好（不超时）；引入消息队列复杂性，需处理消费者组、ACK、死信、消息幂等。
**出处**: [[architecture/gateway.md]]

---

## ADR-002: Session 使用分布式锁保证串行执行 (2026-04-05)
**决策**: 同一 Session 同一时刻只允许一个 Worker 执行，通过 Redis SET NX 实现。
**背景**: 并发请求可能打乱对话上下文顺序（消息 A 的 Session 还没保存，消息 B 已经开始读取）。
**后果**: 同一会话串行，高并发单用户场景有排队延迟；锁超时（30s）可能导致长任务重复执行。
**出处**: [[architecture/gateway.md]]

---

## ADR-001: DeerFlow 使用 `create_agent()` 工厂而非直接操作 StateGraph (2026-04-16)
**决策**: 使用 LangChain 高级工厂函数封装 Agent，不直接操作 LangGraph StateGraph API。
**背景**: 工厂函数封装了工具绑定、消息处理、错误恢复等样板代码，开发效率更高。
**后果**: 深度定制需要绕过工厂函数；但对于快速迭代的探索阶段，这是合理权衡。
**出处**: [[architecture/deerflow.md]]
