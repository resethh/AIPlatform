# hermes-agent 探索 QA 记录

---

## 关注点总结

本次探索集中在三个核心主题：

1. **运行时机制**：MCP server 怎么启动、存活、被 Agent 调用，工具调度的完整链路
2. **Skill 系统**：技能如何创建、加载、评估、自我修补，以及降级/条件激活的设计逻辑
3. **凭证管理**：API key / OAuth token 从哪来、存哪里、怎么过期刷新，以及 skill 内联收集 key 的机制

---

## Q&A 列表

---

### Q1：`skill_manager_tool.py` 如何加载 MCP，被智能体使用的流程？

**A：** 两者共用同一个 `ToolRegistry` 单例，但彼此独立。

- `skill_manager_tool.py` 在模块底部调用 `registry.register(name="skill_manage", toolset="skills", ...)` 注册为内置工具
- MCP 工具由 `mcp_tool.py` 的 `discover_mcp_tools()` 动态注册，格式为 `mcp_{server}_{tool}`
- 两类工具都在 `import model_tools` 时完成注册（进程生命周期内一次），之后统一通过 `registry.dispatch(name, args)` 调用

---

### Q2：MCP server 是启动后一直存在吗？

**A：** 是的，长驻存活。

每个 MCP server 对应一个 `MCPServerTask`（asyncio Task），在后台守护线程 `mcp-event-loop` 中永久运行，阻塞在 `await self._shutdown_event.wait()`。Agent 启动到关闭全程活着，工具调用复用同一个 session，断线后指数退避重试（最多 5 次）。

---

### Q3：MCP server 会启动几个服务？

**A：** `config.yaml` 里 `mcp_servers` 中 `enabled != false` 的条目数，每条对应一个独立的 `MCPServerTask`，并行连接，某个失败不影响其他。

---

### Q4：MCP server 的端口是相同的吗？

**A：** 不冲突，两种连接方式：

| 类型 | 方式 | 端口 |
|------|------|------|
| stdio | 启动子进程，通过 stdin/stdout 管道通信 | 无端口 |
| HTTP | 建立 httpx 长连接 | 无本地端口（是客户端，连远端 URL） |

stdio 类型完全无端口，HTTP 类型各自连不同 URL。

---

### Q5：还会有不同的 URL 吗？

**A：** 会。每个 HTTP 类型的 MCP server 配置各自的 `url` 字段，指向不同远端服务。多个 server 之间完全独立，互不影响。

---

### Q6：`registry.dispatch` 调用 MCP 工具的详细步骤？

**A：** 跨线程桥接流程：

```
Agent 主线程
    registry.dispatch("mcp_github_search_code", args)
        └─► entry.handler(args)
                └─► _run_on_mcp_loop(session.call_tool(...))
                        ├─ run_coroutine_threadsafe(coro, mcp_loop) → 提交给后台 asyncio 线程
                        └─ 主线程每 0.1s 轮询 future.done() + is_interrupted()
                            ├─ 用户中断 → future.cancel()，抛 InterruptedError
                            └─ 完成 → future.result()，返回工具结果
```

0.1s 轮询设计是为了在等待 MCP 响应时支持用户中断（Ctrl+C）。

---

### Q7：Agent 从启动到任务完成的完整生命周期？

**A：** 分四个阶段：

| 阶段 | 时机 | 频率 |
|------|------|------|
| 工具注册（内置+MCP+插件） | `import model_tools` | 进程内一次 |
| tools schema 固定 | `AIAgent.__init__()` | 每个 Agent 实例一次 |
| 主循环（LLM ↔ 工具） | `run_conversation()` | 每次对话 |
| 任务完成退出 | `finish_reason == "stop"` | 按需 |

主循环默认最多 90 次 API 调用，每轮带上全部工具 schema，LLM 返回 `tool_calls` 就执行工具，返回 `stop` 就退出。

---

### Q8：这个项目怎么做 Skill 的创建、优化、评估？

**A：** 三个机制：

- **创建**：`skill_manage(action="create")`，验证名称/frontmatter，原子写入，安全扫描（注入模式/Unicode/路径穿越）
- **优化**：发现 skill 有误时立即 `skill_manage(action="patch")` 修补，不等用户要求；修补后自动清除系统提示缓存
- **评估**：隐式评估，通过 `SkillReadinessStatus`（AVAILABLE / SETUP_NEEDED / UNSUPPORTED）和条件激活（`fallback_for_tools` / `requires_toolsets`）动态决定 skill 是否可用/可见

---

### Q9：为什么有降级处理（`fallback_for_tools`）这种设计？

**A：** 同一件事往往有两种做法：原生工具（快、精准，但需要 API key）和 Skill（零依赖，靠 Agent 手动执行）。若两者同时出现在 system prompt，Agent 会困惑该用哪个。降级机制让 Skill 按需显现：

- 有原生工具 → Skill 隐藏
- 没有原生工具 → Skill 出现顶上

---

### Q10：`fallback_for_tools` 是提示词实现的吗？三层缓存内容一样吗？

**A：**

**不是提示词**，是纯 Python 代码过滤（`prompt_builder.py:552` 的 `_skill_should_show()`），在 `<available_skills>` 文本生成**之前**就过滤掉，LLM 完全看不到被过滤的 skill。

**三层缓存内容不同：**

| 层 | 存储内容 | 生命周期 |
|----|---------|---------|
| 第1层 LRU（内存） | 最终 prompt 字符串（已过滤） | 进程内，skill_manage 后主动清除 |
| 第2层 磁盘快照 | 原始元数据（未过滤），含 mtime/size 指纹 | 跨进程，文件变化后自动失效 |
| 第3层 文件系统扫描 | 不存储，直接读 SKILL.md，写入第2层 | 冷路径，前两层都 miss 时执行 |

---

### Q11：Skill 评估和加载到 Agent 的详细实现？

**A：** 见 `skill管理.md` 第九、十节，核心要点：

**评估（`skill_view()`）**：路径查找（三种策略）→ 安全检查 → 平台检查 → 禁用检查 → 环境变量捕获（CLI 交互/Gateway hint/远端说明）→ 凭证文件注册 → 环境透传注册 → 返回 `SkillReadinessStatus`

**加载到 Agent（`build_skills_system_prompt()`）**：LRU → 磁盘快照 → 冷扫描；三重过滤（平台/禁用/条件激活）；注入强制指令 `"you MUST load it with skill_view()"`

---

### Q12：凭证文件注册（`credential_files.py`）怎么实现的？

**A：** 解决远程沙箱（Docker/Modal）启动时无宿主机文件的问题：

- `ContextVar` 存储注册表（防 Gateway 多会话串漏）
- 注册时做两层安全检查：拒绝绝对路径 + 路径穿越检测（`validate_within_dir`）
- 文件不存在 → 加入 missing 列表 → `setup_needed = True`
- 沙箱创建时调 `get_credential_file_mounts()` 合并 skill 注册 + config.yaml 两个来源
- skills 目录挂载时检测 symlink，有 symlink 则创建无 symlink 的临时副本再挂载（防密钥泄露）

---

### Q13：Key 怎么判断什么时候获取、存在哪里、有效期处理？

**A：** 两类 key 处理方式不同：

**静态 API key（`~/.hermes/.env`）**：无有效期，调用时读文件，`_is_env_var_persisted()` 判断是否存在。

**OAuth token（`~/.hermes/auth.json`）**：有有效期，每次 API 调用前 `resolve_xxx_credentials()` 检查是否即将过期（提前 120 秒），需要则刷新：

```
_is_expiring(expires_at, skew=120)
    ├─ 通用：ISO-8601 时间戳解析
    ├─ Codex：JWT base64 解包读 exp 字段
    └─ Qwen：毫秒时间戳比较
```

凭证池失效：API 返回 429/402 → `mark_exhausted` → 冷却 1 小时；provider 返回 reset_at 则用指定时间。

---

### Q14：对接外部 SaaS 的 API key 怎么存的？

**A：** 存在 `~/.hermes/.env`，明文 `KEY=VALUE`，靠文件权限 `0o600` 保护。

写入流程（`save_env_value()`）：名称格式校验 → 非 ASCII 字符检测并 strip（防 PDF 复制混入相似字符导致 HTTP 头编码错误）→ 读现有文件并修复损坏行 → upsert → `mkstemp` 原子写入 → `fsync` 落盘 → `os.replace` 原子替换 → `chmod 0o600` → `os.environ[key] = value` 即时生效。

---

### Q15：有没有自动收集外部 key 的功能？

**A：** 有，通过 skill 的 `required_environment_variables` 或 `setup.collect_secrets` 声明：

```yaml
required_environment_variables:
  - name: STRIPE_API_KEY
    prompt: "Stripe Secret Key (https://dashboard.stripe.com/apikeys)"
    optional: false
```

用户第一次 `/skill-name` 时：
- 本地 CLI → TUI 弹输入框（`getpass`，屏幕不显示），输入后自动存入 `.env`
- Gateway/Web → 返回文字提示，引导用户手动配置
- 值不暴露给 LLM（返回给模型的只是"已存储"的结果）
- 下次调用自动通过，不再询问

---

## 文件索引

| 文件 | 内容 |
|------|------|
| `skill管理.md` | Skill 完整生命周期、三层缓存、降级设计、评估与加载详细实现、外部 key 收集机制 |
| `skill管理tool.md` | skill_manager_tool.py 与 MCP 加载流程、Agent 完整生命周期时序 |
| `QA.md` | 本文件，问答记录与关注点总结 |

---

### Q16：项目里是否对接了外部 SaaS 系统？API Key 的收集与存储怎么处理的？

**A：** 有，通过 Skill 形式对接了十余个外部服务。

**对接的主要 SaaS：**

| 类别 | SaaS | Key 变量名 |
|------|------|-----------|
| 代码托管 | GitHub | `GITHUB_TOKEN` |
| 项目管理 | Linear | `LINEAR_API_KEY` |
| 笔记数据库 | Notion | `NOTION_API_KEY` |
| 社交媒体 | X/Twitter | `X_API_KEY` 等 5 个 |
| 邮件 | AgentMail | `AGENTMAIL_API_KEY` |
| 知识库 | SiYuan | `SIYUAN_TOKEN` |
| 密钥管理 | 1Password | `OP_SERVICE_ACCOUNT_TOKEN` |
| ML 追踪 | Weights & Biases | `WANDB_API_KEY` |
| 云计算 | Modal | 由 `modal setup` 管理 |

**三种声明方式（能力逐步增强）：**

```yaml
# 方式一：仅声明（旧格式，不弹框）
prerequisites:
  env_vars: [NOTION_API_KEY]

# 方式二：带提示（新格式，CLI 弹框）
required_environment_variables:
  - name: SIYUAN_TOKEN
    prompt: "SiYuan API token"
    help: "Settings > About in SiYuan desktop app"
    optional: true

# 方式三：最完整（带跳转 URL）
setup:
  collect_secrets:
    - env_var: OP_SERVICE_ACCOUNT_TOKEN
      prompt: "1Password Service Account Token"
      provider_url: "https://developer.1password.com/docs/service-accounts/"
      secret: true
```

**Key 最终存在哪里：**

| 存储位置 | 适用场景 |
|---------|---------|
| `~/.hermes/.env` | 绝大多数 SaaS API key |
| `~/.hermes/config.yaml` mcp_servers.env | 通过 MCP 接入的服务（如 AgentMail） |
| 第三方工具自管 | 有专属 CLI 的服务（Modal、W&B） |
| `~/.config/<tool>/` | 有自己配置目录的工具（Himalaya） |

**收集流程**：用户首次调用 skill 时，若 `~/.hermes/.env` 里缺少 key，本地 CLI 弹 `getpass` 输入框；Gateway/Web 端返回文字提示引导手动配置；输入值原子写入 `.env`，不暴露给 LLM。
