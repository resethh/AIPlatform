# 开源 Agent 框架功能对比

> 对比对象：Hermes Agent (Nous Research) / OpenClaw / DeerFlow (字节)
> 更新日期：2026-04-20

---

## 一、总览

| 维度 | Hermes | OpenClaw | DeerFlow |
|---|---|---|---|
| **定位** | 自我改进 ToC Agent | 企业级多 Agent 网关（ToB/ToC） | 研究型多 Agent 系统 |
| **架构风格** | 自定义 Agent Loop | 消息队列 + Worker 执行集群 | LangGraph StateGraph |
| **主要语言** | Python ≥3.11 | Python (FastAPI) | Python (LangChain) |
| **多 Agent** | 无（单 Agent + 40+ 工具） | 多 Agent，路由分发 | Lead Agent + 子 Agent 并发 |
| **目标用户** | 个人开发者/极客 | 企业多部门、多渠道 | 研究/复杂推理任务 |

---

## 二、架构风格

| 特性 | Hermes | OpenClaw | DeerFlow |
|---|---|---|---|
| 核心执行模型 | 自定义 `run_conversation()` 循环，最多 90 次 API 调用 | `MessagePipeline` → Redis Streams → `AgentWorker` 消费 | `create_agent()` 工厂 + LangGraph StateGraph |
| 多 Agent 协调 | ❌ 无内置协调 | ✅ 路由引擎（5 级优先级） | ✅ Lead Agent 委派子 Agent，最大并发 3 |
| 异步处理 | 工具执行同步，记忆更新异步 | Webhook 同步 ≤40ms 响应，Agent 执行完全异步 | 记忆更新/Token 统计异步，关键路径不阻塞 |
| 状态持久化 | 文件系统（MEMORY.md/SKILL.md） | Redis（热数据）+ PostgreSQL（持久化） | LangGraph Checkpointer（async_provider） |
| 工具生态 | 40+ 内置工具 + 动态 MCP 工具 | 自定义工具（sql_query/chart_gen 等） | LangChain 工具生态 |
| 扩展机制 | MCP server（stdio/HTTP），长驻进程 | 工具注册表 + Redis 队列扩容 | 子 Agent 线程池（最多并发 3） |

---

## 三、记忆系统

| 特性 | Hermes | OpenClaw | DeerFlow |
|---|---|---|---|
| 记忆层次 | 2 层（MEMORY.md 笔记 + USER.md 用户画像） | 4 层（Session JSONL / Session Memory / Workspace Files / Boot） | 3 层（workContext / history / facts JSON） |
| 短期记忆 | Session 内对话历史 | JSONL 逐行追加，Session 存续期 | 当前 Session messages（可被压缩） |
| 中期记忆 | ❌ 无 | `memory/YYYY-MM-DD-{slug}.md`，/new 时触发 | 防抖队列批量 LLM 提取 |
| 长期记忆 | MEMORY.md（2200 字符上限）+ USER.md（1375 字符） | MEMORY.md/USER.md/TOOLS.md/AGENTS.md/SOUL.md | facts 数组，含 confidence 分和 category |
| 更新机制 | Agent 自主 read/edit/write（非阻塞） | Agent 自主文件读写；Session 结束时 LLM 生成 slug | 对话结束后 LLM 提取 → 防抖异步队列写入 JSON |
| 注入方式 | **冻结快照**（Session 开始时固定，中途不变，保 Prompt Cache 稳定） | 每次 Session 启动注入 workspace 文件内容 | `format_memory_for_injection()` 写入 System Prompt |
| 衰减策略 | ❌ 无（靠字符上限控制） | ❌ 无显式衰减 | ✅ 时间衰减 + 访问强化 + 睡眠巩固（低峰期整合） |
| 外部 Provider | ✅ 插件化（Honcho/Hindsight/Mem0） | ❌ 无 | ❌ 无 |
| 跨设备同步 | ❌ 本地文件 | ❌ 本地文件 | ✅ 可（JSON 文件，可挂载云存储） |

---

## 四、上下文管理

| 特性 | Hermes | OpenClaw | DeerFlow |
|---|---|---|---|
| System Prompt 组装 | `prompt_builder.py`：角色 + 记忆快照 + Skill 目录 + 工具 schema | AGENTS.md/SOUL.md/USER.md 注入；MEMORY.md 仅 main session | `ContextAssembler`：角色 + 渠道提示 + 记忆注入 + Tool Schema |
| 对话历史压缩 | `context_compressor.py` — 滑动窗口 + LLM Summarization | JSONL 归档 + Session Memory Hook | `Summarization` 中间件（满了压缩为摘要段落） |
| 渐进式加载 | ✅ Skill 三级加载（list → view → view+path） | ❌ 无显式分级 | ✅ 工具/技能按需读取，System Prompt 只含目录 |
| Memory Flush | ❌ 无（记忆更新独立于压缩） | ❌ 无 | ✅ 压缩前提取关键信息写入记忆层 |
| Token 预算控制 | Skill 索引 ~3k token；总体无硬预算 | ❌ 无显式控制 | ✅ 明确 token budget 管理，Tool 输出截断 |
| 多渠道适配 | 多平台 CLI/Telegram/Discord 等 | 钉钉/企微/飞书，渠道格式适配提示 | ❌ 无多渠道概念 |
| Tool 输出截断 | ❌ 无 | ❌ 无 | ✅ 超阈值截断前后 N 行 |

---

## 五、Skills / Tools 管理

| 特性 | Hermes | OpenClaw | DeerFlow |
|---|---|---|---|
| 技能载体 | SKILL.md 文件（程序性记忆） | 无 Skill 概念，靠工具组合 | 无 Skill 概念，靠子 Agent 分工 |
| 技能创建 | ✅ Agent 自主创建（5+ tool calls 触发） | ❌ 无 | ❌ 无 |
| 技能版本 | frontmatter `version` 字段，patch 时递增 | ❌ 无 | ❌ 无 |
| 渐进式加载 | ✅ 三层（list/view/view+path） | ❌ 无 | ✅ 按需加载工具 schema |
| 条件激活 | ✅ platforms/requires_toolsets/fallback_for_tools | ❌ 无 | ❌ 无 |
| 技能自修复 | ✅ Agent 发现问题立即 patch，不等用户 | ❌ 无 | ❌ 无 |
| 三层缓存 | ✅ LRU 内存 → 磁盘快照 → 全量扫描 | ❌ 无 | ❌ 无 |
| 多源安装 | ✅ Hub（builtin/official/trusted/community/GitHub/URL） | ❌ 无 | ❌ 无 |
| 安全扫描 | ✅ 创建/安装时（注入模式/危险命令/凭证/Unicode） | ❌ 无 | ❌ 无 |
| 组织继承 | ❌ 无（单用户文件系统） | ❌ 无 | ❌ 无 |
| RL 自我优化 | ✅ Tinker-Atropos（GRPO），Trajectory 生成 | ❌ 无 | ❌ 无 |

---

## 六、路由机制

| 特性 | Hermes | OpenClaw | DeerFlow |
|---|---|---|---|
| 路由模式 | 无路由（单 Agent） | ✅ 5 级优先级路由（显式/Session/mention/渠道/关键词） | ✅ Lead Agent 决定委派哪个子 Agent |
| Session 延续 | ✅ 对话历史维持 | ✅ Session 锁保证串行，agent_id 绑定 | ✅ LangGraph Checkpointer 保持状态 |
| 多渠道接入 | ✅ 多平台（CLI/IM/WhatsApp/Signal） | ✅ 钉钉/企微/飞书统一 Webhook 适配 | ❌ 无多渠道 |
| 负载均衡 | ❌ 单进程 | ✅ Worker 集群 + Redis Streams 消费者组 | ❌ 无 |
| 优先级队列 | ❌ 无 | ✅ high/normal/low 三级 | ❌ 无 |

---

## 七、安全

| 特性 | Hermes | OpenClaw | DeerFlow |
|---|---|---|---|
| Prompt Injection 防护 | ✅ skill_view 时检测（10+ 模式，仅 warning） | ❌ 无 | ❌ 无 |
| 工具权限控制 | ✅ toolset 白名单 | ✅ Agent tools 字段硬编码 | ✅ 子 Agent 工具集隔离 |
| 凭证管理 | `~/.hermes/.env`（0o600），原子写入，值不暴露给 LLM | ❌ 无专门机制 | ❌ 无专门机制 |
| 跨租户隔离 | ❌ 单用户 | ✅ Session 隔离 + tenant_id 贯穿链路 | ❌ 无 |
| 限流 | ❌ 无 | ✅ 用户级 10 req/min + Agent 级并发上限 | ❌ 无 |
| 审计日志 | ❌ 无 | ✅ message_logs 表（全量审计） | ❌ 无 |
| HITL 审批 | ❌ 无 | ⚠️ 设计中 | ❌ 无 |
| 供应链安全 | ✅ Skill 安装扫描，信任分级 | ❌ 无 | ❌ 无 |

---

## 八、自我改进能力

| 特性 | Hermes | OpenClaw | DeerFlow |
|---|---|---|---|
| 程序性记忆 | ✅ SKILL.md（从经验提炼，可复用） | ❌ 无 | ❌ 无 |
| 自主创建知识 | ✅ Agent 自主创建/修补 Skill | ✅ Agent 自主维护 MEMORY.md/USER.md 等 | ❌ 无 |
| RL 训练 | ✅ GRPO（Tinker-Atropos） | ❌ 无 | ❌ 无 |
| 记忆巩固 | ❌ 无 | ✅ Session Memory Hook（/new 时摘要） | ✅ 睡眠巩固（低峰期深度整合） |
| 错误自修复 | ✅ 发现 Skill 有误立即 patch | ❌ 无 | ❌ 无 |

---

## 九、部署与运维

| 特性 | Hermes | OpenClaw | DeerFlow |
|---|---|---|---|
| 部署形态 | 单机进程，CLI/多 IM 平台接入 | 分布式（API + Worker + Dispatcher + Redis + PG） | 单机/容器，LangGraph Server |
| 水平扩展 | ❌ 进程内 Semaphore，单机并发 | ✅ Worker 集群横向扩展 | ❌ 有限（子 Agent 线程池） |
| 监控指标 | ❌ 无 | ✅ 完整（消息量/队列延迟/token 用量/工具耗时） | ❌ 无 |
| 多租户 | ❌ 单用户 | ✅ tenant_id 贯穿全链路 | ❌ 无 |
| 最小资源 | 单台 ≥ 4GB RAM | 单台 4核8G 起步 | 单台，无特殊要求 |

---

## 十、中台设计选型建议

基于以上对比，AI 中台设计的取舍原则：

| 维度 | 采用来源 | 理由 |
|---|---|---|
| **接入层架构** | OpenClaw | 消息队列解耦、多渠道统一适配是 ToB 刚需 |
| **Skill 格式标准** | Hermes | SKILL.md 是事实标准，agentskills.io 兼容 |
| **渐进式加载** | Hermes | Token 节省 80%+，ToB 30+ Skill 场景收益显著 |
| **条件激活** | Hermes → 扩展 | 增加渠道/时间/等级等 ToB 维度 |
| **记忆更新机制** | DeerFlow | 防抖异步队列比 Hermes 的即时写入更适合高并发 |
| **记忆衰减/巩固** | DeerFlow | 时间衰减 + 睡眠巩固是中台长期运营的必要能力 |
| **多 Agent 协调** | DeerFlow 思路 + OpenClaw 路由 | DeerFlow 的 Lead/Sub 分工 + OpenClaw 的路由优先级 |
| **安全体系** | 自建（参考 OpenClaw） | OpenClaw 有完整分层，Hermes 只有 Skill 安全扫描 |
| **RL 自改进** | Hermes | Trajectory + GRPO 是中长期能力建设方向 |

---

## 相关页面

- [[architecture/gateway.md]] — OpenClaw 接入层参考
- [[architecture/hermes.md]] — Hermes 完整框架分析
- [[architecture/deerflow.md]] — DeerFlow 完整框架分析
- [[skills/hermes-skills.md]] — Hermes Skill 系统详解
- [[skills/skill-iteration.md]] — 中台 Skill 设计（ToB 改造）
- [[memory/memory-management.md]] — 中台记忆管理设计
- [[security/agent-security.md]] — 安全体系设计
