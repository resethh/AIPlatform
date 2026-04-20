# 知识库 QA 记录

---

## Q: Hermes MCP server 的工具注册和调用链路 (2026-04-16)
**问题**: `skill_manager_tool.py` 如何加载 MCP，被智能体使用的流程？MCP server 是否长驻？如何调用？
**答案**:
- Skill 工具和 MCP 工具共用同一 `ToolRegistry` 单例，但注册方式不同：skill_manage 在模块底部手动注册，MCP 工具由 `discover_mcp_tools()` 动态注册，格式为 `mcp_{server}_{tool}`
- MCP server 长驻：每个 server 对应一个后台 asyncio Task，Agent 生命周期内永久运行，断线后指数退避重试（最多 5 次）
- 调用链路：`registry.dispatch()` → `_run_on_mcp_loop()` → `run_coroutine_threadsafe()` 跨线程桥接，主线程每 0.1s 轮询以支持用户中断（Ctrl+C）
- MCP server 间互不影响：stdio 类型走子进程管道（无端口），HTTP 类型各自连不同远端 URL

**相关页面**: [[skills/hermes-skills.md]]、[[architecture/hermes.md]]

---

## Q: Hermes Skill 评估、加载、降级机制 (2026-04-16)
**问题**: Skill 如何评估、加载到 Agent？`fallback_for_tools` 降级是提示词实现的吗？三层缓存内容一样吗？
**答案**:
- **评估**：`skill_view()` 依次做路径查找 → 安全检查 → 平台检查 → 禁用检查 → 凭证状态评估，返回 `SkillReadinessStatus`（AVAILABLE/SETUP_NEEDED/UNSUPPORTED）
- **加载**：`build_skills_system_prompt()` 走三层缓存（LRU内存 → 磁盘快照 → 冷扫描），三重过滤（平台/禁用/条件激活）后注入系统提示
- **降级**：`fallback_for_tools` 是纯 Python 代码过滤（`prompt_builder.py` `_skill_should_show()`），在生成 `<available_skills>` 文本前就过滤，LLM 完全看不到被隐藏的 Skill
- **三层缓存内容不同**：L1 LRU 存最终 prompt 字符串；L2 磁盘快照存原始元数据（含 mtime 指纹）；L3 直接读文件

**相关页面**: [[skills/hermes-skills.md]]

---

## Q: Hermes 凭证管理机制 (2026-04-16)
**问题**: API key / OAuth token 从哪来、存哪里、怎么过期刷新？凭证文件注册怎么实现？
**答案**:
- **静态 API key**：存 `~/.hermes/.env`（明文，`chmod 0o600`），调用时读文件，`save_env_value()` 写入时原子替换 + fsync + 非 ASCII 字符 strip
- **OAuth token**：存 `~/.hermes/auth.json`，每次 API 调用前检查是否即将过期（提前 120 秒），需要则刷新；支持 ISO-8601 / JWT base64 / 毫秒时间戳三种格式
- **凭证池失效**：API 返回 429/402 → `mark_exhausted` → 冷却 1 小时
- **凭证文件注册**：`ContextVar` 防多会话串漏；拒绝绝对路径 + 路径穿越检测；沙箱挂载时检测并消除 symlink 防密钥泄露

**相关页面**: [[skills/hermes-skills.md]]、[[security/agent-security.md]]
