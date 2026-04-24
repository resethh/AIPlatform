# 用户提问时间线

> 只记录用户发出的问题与日期，按时间顺序追加在末尾。
> 完整的问答内容（含回答、相关页面）请看 [[用户问题.md]]。

---

## 2026-04-20
- 优化 CLAUDE.md 的工作流指令模块，新文件判断用 git 工具，提问记录到 QA 文档并判断是否有类似问题进行合并
- 处理新资料（首次全量收录 15 份 raw 文件）
- wiki 文档不够全面，先列 hermes skill 管理的功能清单再补齐，并增加不同 Agent 框架的功能对比文档
- 基于以上工作优化 CLAUDE.md，增加文档补齐、框架对比工作流，并增加判断是否需要修改 CLAUDE.md 的机制
- 提交 git

## 2026-04-21
- 提交成功了吗？可是 GitHub 没看到提交，问题出在哪里
- INDEX 文件的内容不够丰富具体，帮我优化下
- 作为系统架构，我还想看到时序图，应该放在哪里，帮我规划下
- 实现用户发的 query 模型自动判断是否要修改 CLAUDE.md，如需要则自动修改，并执行画图和修改 INDEX.md
- 再看下 raw/ 下的文件，判断 wiki 下的目录是否有类似时序图这样缺的内容，以及是否超出 CLAUDE.md 里的关注领域
- 从缺口 3 开始，全部补齐

## 2026-04-22
- 如何理解 MCP 采用 Manager-Worker 分离架构
- 从 Swagger/OpenAPI 导入，如果是要动态获取的 key 呢
- 处理新资料 HiClaw 1.0.6，并总结 Skill 管理哪些挑战点形成新文档
- 源文件更改了，收录新资料时需要处理 M（修改）状态，不需要提到暂存区
- 当用户问"怎么做"时，严格依据项目内容回答，如果没有则回答未找到；优先看 wiki/，wiki 没有但 raw 有则及时补充到 wiki
- Hermes Agent 记忆系统中的 Session FTS5，优点是什么，为什么这么做
- 把 hermes-skills.md 改名为 skill-manage.md，融合 skill-iteration.md，存一个 AI 中台 Skill 管理方案，ToB 使用 PostgreSQL 和 OSS 对象存储
- 当提出技术问题时，要分析想了解的是设计思路和实现目的和场景，基于此扩展 CLAUDE.md
- 记忆的衰减、睡眠增强、可信度没看到

## 2026-04-24
- 收录新资料，Building agents that reach production systems with MCP 的内容与 HiClaw 1.0.6 做整合，一块写入 mcp-management
- skill、mcp、agent 之间的区别是什么
- 存在综合问题里吧，以后不确定分类的都存在这个文档里
- 我想以后把用户问题和回答都放在用户问题.md，QA 里只存用户的问题就行，按顺序往里追加并记录时间
- mcp-manage 架构链路增加个时序图
- 在 claude.md 中添加自行判断是否需要时序图
