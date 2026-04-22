# 知识库 QA 记录

---

## 2026-04-20

### 优化 CLAUDE.md 工作流指令
**请求**: 优化 CLAUDE.md 的工作流指令模块，新文件判断用 git 工具，提问记录到 QA 文档并判断是否有类似问题进行合并
**回答概要**: 重写了收录新资料工作流（改为 `git status --short raw/` 检测新文件），新增 QA 去重合并逻辑，工作流名称改为口语化中文，整体语言风格统一
**相关页面**: [[CLAUDE.md]]

### 收录 raw/ 下所有资料
**请求**: 处理新资料（首次全量收录 15 份 raw 文件）
**回答概要**: 从 15 份原始资料编译生成完整 wiki/，创建 5 个模块目录（architecture/security/memory/context/skills），共 16 个页面，同步更新 INDEX.md 和 log.md
**相关页面**: [[INDEX.md]]、[[log.md]]

### 补齐 hermes-skills.md 并新增框架对比文档
**请求**: wiki 文档不够全面，先列 hermes skill 管理的功能清单再补齐，并增加不同 Agent 框架的功能对比文档
**回答概要**: 生成 14 项功能清单（✅/❌ 标注），全量补写 hermes-skills.md（三层缓存、skill_view 十步流程、完整生命周期等），新建 wiki/comparison.md（九维度横向对比 + 中台选型建议）
**相关页面**: [[skills/hermes-skills.md]]、[[comparison.md]]

### 优化 CLAUDE.md（新增工作流 + 自我优化规则）
**请求**: 基于以上工作优化 CLAUDE.md，增加文档补齐、框架对比工作流，并增加判断是否需要修改 CLAUDE.md 的机制
**回答概要**: 新增§2 文档补齐、§3 框架对比两条工作流，重新编号；新增"CLAUDE.md 自我优化规则"章节（4 触发信号表），明确哪些对话触发 CLAUDE.md 修改
**相关页面**: [[CLAUDE.md]]

### 提交 git
**请求**: 提交 git
**回答概要**: `git add CLAUDE.md wiki/` 后 commit，18 个文件 1423 行新增，commit `511b27a`

---

## 2026-04-21

### git push 未推送到 GitHub
**请求**: 提交成功了吗？可是 GitHub 没看到提交，问题出在哪里
**回答概要**: commit 只在本地，未 push 到远端；执行 `git push origin main` 后推送成功

### 优化 INDEX.md 内容丰富度
**请求**: INDEX 文件的内容不够丰富具体，帮我优化下
**回答概要**: 重写 INDEX.md，新增核心内容摘要列（每页一句话）、文本版模块依赖关系图、按场景分组的快速入口（7 个场景）
**相关页面**: [[INDEX.md]]

### 规划时序图放置位置
**请求**: 作为系统架构，我还想看到时序图，应该放在哪里，帮我规划下
**回答概要**: 混合方案：系统级跨组件时序 → 新建 `wiki/flows/` 目录；组件内部时序 → 嵌入对应组件页面；优先建 message-pipeline.md（骨架时序图）
**相关页面**: [[flows/message-pipeline.md]]

### 自动判断是否需要修改 CLAUDE.md 并执行
**请求**: 实现用户发的 query 模型自动判断是否要修改 CLAUDE.md，如需要则自动修改，并执行画图和修改 INDEX.md
**回答概要**: 判断本次对话触发了两条规则空白（时序图工作流、INDEX.md 规范），更新 CLAUDE.md 新增§5 时序图工作流 + wiki/flows/ 模板 + INDEX.md 三部分规范；创建 flows/message-pipeline.md（完整 Mermaid 时序图）；重写 INDEX.md
**相关页面**: [[CLAUDE.md]]、[[flows/message-pipeline.md]]、[[INDEX.md]]

### 巡检 raw/ 文件覆盖情况和关注领域范围
**请求**: 再看下 raw/ 下的文件，判断 wiki 下的目录是否有类似时序图这样缺的内容，以及是否超出 CLAUDE.md 里的关注领域
**回答概要**: 15 份 raw 文件全部在关注领域内，无越界；发现 4 处缺口：DeerFlow 执行架构无专页、hermes-QA.md 未入库、路由级数误差（wiki 写"五级"实为七级）、flows/ 还有 4 张子时序图待建

### 补齐全部 4 处缺口
**请求**: 从缺口 3 开始，全部补齐
**回答概要**: ①修复 agent-router.md 路由级数（P0–P6 七级，补入 P4 LLM 意图分类和路由规则数据模型）；②QA.md 新增 Hermes Agent 生命周期条目；③新建 deerflow-agent.md（13 层中间件、子 Agent 系统、Skill 加载链）；④新建 4 张时序图（agent-loop/skill-loading/multi-agent/memory-lifecycle）；⑤更新 INDEX.md
**相关页面**: [[architecture/agent-router.md]]、[[architecture/deerflow-agent.md]]、[[flows/]]

---

## 2026-04-22

### 收录 HiClaw 1.0.6 并汇总 Skill 管理挑战
**请求**: 处理新资料 HiClaw 1.0.6，并总结 Skill 管理哪些挑战点形成新文档
**回答概要**: 新建 `architecture/mcp-management.md`（HiClaw Manager-Worker + AI Gateway 凭证代理模式）；新建 `skills/skill-challenges.md`（8 项挑战域：上下文工程/多租户/Skill 演进/版本管理/运行时安全/配额降级/分布式一致性/MCP 协同），每项含本质-难点-方案-残余风险；更新 INDEX.md 和 log.md
**相关页面**: [[architecture/mcp-management.md]]、[[skills/skill-challenges.md]]、[[INDEX.md]]

### 收录新资料工作流补充 M 状态处理
**请求**: 源文件更改了，收录新资料时需要处理 M（修改）状态，不需要提到暂存区
**回答概要**: 更新 CLAUDE.md 收录工作流，新增 `M` 状态处理逻辑（只更新 wiki 受影响部分，不全量重写），说明 git status 不需要先暂存即可检测
**相关页面**: [[CLAUDE.md]]

### 新增"怎么做"问题的严格依据规则
**请求**: 当用户问"怎么做"时，严格依据项目内容回答，如果没有则回答未找到；优先看 wiki/，wiki 没有但 raw 有则及时补充到 wiki
**回答概要**: 在 CLAUDE.md 新增三级查找顺序：①优先 wiki 直接回答；②wiki 无但 raw 有则先补 wiki 再回答；③都没有则明确说未找到并告知查过的页面
**相关页面**: [[CLAUDE.md]]

### Hermes Session FTS5 设计思路
**请求**: Hermes Agent 记忆系统中的 Session FTS5，优点是什么，为什么这么做；以及追问：想了解的是为什么这么设计
**回答概要**: Session FTS5 解决 MEMORY.md 2200 字符硬限与"无限历史档案"之间的矛盾——MEMORY.md 是"总是可见的提炼事实"，FTS5 是"按需回忆的历史全文档案"，两层职责分离；选 FTS5 而非向量搜索是因为 SQLite 内置零依赖，语义质量不足由后置 LLM 摘要补偿；MEMORY.md 的字符硬限是有意设计，强迫 Agent 做价值判断
**相关页面**: [[memory/hermes-memory.md]]

### 合并 Skill 文档为中台方案页
**请求**: 把 hermes-skills.md 改名为 skill-manage.md，融合 skill-iteration.md，存一个 AI 中台 Skill 管理方案，ToB 使用 PostgreSQL 和 OSS 对象存储
**回答概要**: 新建 skill-manage.md（Hermes 14 项功能 + ToB 树形继承 + PostgreSQL schema + OSS 路径规范 + 版本灰度 + 6 级信任 Hub），删除原两个文件，更新 INDEX.md
**相关页面**: [[skills/skill-manage.md]]、[[INDEX.md]]

### 新增技术问题回答规范到 CLAUDE.md
**请求**: 当提出技术问题时，要分析想了解的是设计思路和实现目的和场景，基于此扩展 CLAUDE.md
**回答概要**: 在 CLAUDE.md 新增"技术问题回答规范"章节，定义 4 种问法特征与对应的真实意图和回答重心，默认优先解释"为什么这么设计"而非功能描述
**相关页面**: [[CLAUDE.md]]

### 记忆衰减、睡眠巩固、置信度文档补全
**请求**: 记忆的衰减、睡眠增强、可信度没看到
**回答概要**: 扩写 memory-management.md（新增置信度体系、衰减机制、睡眠巩固三大章节）；扩写 deerflow-memory.md（补入 5 类 category、confidence 写入/注入/截断三阶段、LLM 输出 JSON diff 格式、无时间衰减说明）
**相关页面**: [[memory/memory-management.md]]、[[memory/deerflow-memory.md]]

---

## 历史技术参考（来源：raw/hermes-QA.md）

> 以下条目来自 Hermes 源码探索记录，非用户提问，仅作技术参考。

### Hermes MCP server 工具注册和调用链路
Skill 工具与 MCP 工具共用 `ToolRegistry` 单例；MCP server 长驻（asyncio Task，指数退避重试）；调用链：`registry.dispatch()` → `_run_on_mcp_loop()` → `run_coroutine_threadsafe()`，主线程 0.1s 轮询支持中断。

### Hermes Skill 评估、加载、降级机制
`skill_view()` 做十步评估返回 `SkillReadinessStatus`；`build_skills_system_prompt()` 走三层缓存后三重过滤注入提示；`fallback_for_tools` 是纯 Python 过滤，LLM 看不到被隐藏的 Skill；三层缓存内容不同（LRU 存 prompt 字符串，磁盘快照存原始元数据，L3 直接读文件）。

### Hermes 凭证管理机制
静态 API key 存 `~/.hermes/.env`（原子写入，`chmod 0o600`）；OAuth token 存 `~/.hermes/auth.json`，调用前检查是否即将过期（提前 120s 刷新）；凭证文件注册用 `ContextVar` 防多会话串漏，拒绝绝对路径+路径穿越。

### Hermes Agent 完整生命周期与主循环设计
四阶段：工具注册（进程内一次）→ tools schema 固定（每实例一次）→ 主循环 `run_conversation()`（最多 90 次 API 调用）→ `finish_reason=stop` 退出；MCP 工具格式 `mcp_{server}_{tool}`，跨线程桥接。
