# Skill 迭代设计 — Hermes Agent 参考对标

> 版本: v1.0 | 日期: 2026-04-14 | 作者: 大迈
> 隶属于: 企业级多智能体网关架构方案 → Skill 继承体系与版本迭代设计（补充篇）

---

## 一、Hermes Agent Skill 系统核心特性拆解

读完 Hermes Agent 的完整 Skills 文档，提炼出 **对我们 ToB 方案有参考价值的 7 个核心设计**：

### 1.1 Progressive Disclosure（渐进式披露）

```
Hermes 的做法:

Level 0: skills_list()  →  只返回 {name, description, category}  (~3k tokens)
Level 1: skill_view(name)  →  完整的 SKILL.md + 元数据
Level 2: skill_view(name, path)  →  SKILL.md 中引用的具体参考文件

Agent 按需加载，不用的 Skill 不消耗 Token
```

**对 ToB 的启示：**

我们的方案里，Worker 在 `run_agent_loop()` 时一次性把所有可用 Skill 的 `tool_schema` 注入到 `tools` 参数中。当可用 Skill 达到 30+ 时，tool 定义本身就消耗大量 token。

**需要借鉴：** 在 LLM 调用链中实现渐进式加载。

### 1.2 Conditional Activation（条件激活）

```yaml
# Hermes 的做法:
metadata:
  hermes:
    requires_toolsets: [web]          # 仅当 web 工具集可用时加载
    requires_tools: [web_search]      # 仅当特定工具可用时加载
    fallback_for_toolsets: [browser]  # 当 browser 不可用时才作为备选加载
    fallback_for_tools: [browser_navigate]
```

**对 ToB 的启示：**

我们有 Skill 继承的 `grant/deny/override`，但缺少**运行时上下文条件**——某些 Skill 应该根据当前 Session 的环境动态显隐（比如：有浏览器插件就不需要手动爬虫 Skill，有付费 API 就不需要免费备选）。

### 1.3 Agent-Managed Skills（Agent 自管理 Skill）

```
Hermes 的做法:
- Agent 完成复杂任务（5+ tool calls）后，自动创建 Skill 保存解决方案
- 支持 create / patch / edit / delete / write_file / remove_file
- 这是 Agent 的"程序性记忆"——学到的工作流可复用
```

**对 ToB 的启示：**

这在 ToC 场景很强大，但 ToB 需要**严格管控**——不能让 Agent 自己创建/修改 Skill 并影响其他用户。需要引入**草稿+审批流程**。

### 1.4 Skills Hub 多源集成

```
Hermes 支持的 Skill 来源:
- builtin    — 内置随装（最高信任）
- official   — 官方可选（内置信任）
- trusted    — 知名第三方（openai/skills, anthropics/skills）
- community  — 社区贡献（需安全扫描，--force 才能安装）
- well-known — URL 发现（/.well-known/skills/index.json）
- skills-sh  — Vercel 的公共目录
- GitHub     — 直接从 repo 安装
```

**对 ToB 的启示：**

我们需要类似的**多源 + 信任等级**模型，但落到企业场景：
- 平台内置 → 集团级 → BU级 → 第三方市场 → 自建
- 每一级都有不同的信任等级和审核流程

### 1.5 Security Scanning（安全扫描）

```
Hermes 的做法:
- 所有 hub-installed skills 经过安全扫描
- 检查: 数据泄露模式、Prompt 注入、破坏性命令、Shell 注入
- 信任分级: builtin > official > trusted > community
- community 有危险发现 → 直接 blocked；非危险 → --force 可覆盖
```

### 1.6 Skill 目录结构规范

```
skill-name/
├── SKILL.md           # 必需：主指令文件
├── references/        # 可选：参考文档
├── templates/         # 可选：输出格式模板
├── scripts/           # 可选：辅助脚本
└── assets/            # 可选：补充文件
```

### 1.7 Secure Setup on Load（安全配置加载）

```yaml
# Hermes 的做法:
required_environment_variables:
  - name: TENOR_API_KEY
    prompt: "Tenor API key"
    help: "Get a key from https://developers.google.com/tenor"
    required_for: "full functionality"

required_credential_files:
  - path: google_token.json
    description: "Google OAuth2 token (created by setup script)"
```

- 缺失不会隐藏 Skill（只降级）
- 本地 CLI 安全提示，不在聊天中暴露
- 自动传递到沙箱环境

---

## 二、对标差异分析：Hermes (ToC) vs 我们 (ToB)

| Hermes 特性 | ToC 实现 | 我们 ToB 需要的改造 |
|---|---|---|
| **Progressive Disclosure** | Agent 按需 `skill_view()` | 需要：Level 0 索引 + Level 1 按需加载 + tool_schema 延迟注入 |
| **Conditional Activation** | 本地检测 toolsets/tools | 需要：运行时上下文条件 + 组织级条件 + 渠道条件 |
| **Agent-Managed Skills** | Agent 自由创建/修改 | 需要：Agent 创建 → 草稿 → 审核 → 发布流程 |
| **Skills Hub 多源** | 文件系统 + GitHub + URL | 需要：企业内部 Registry + 信任分级 + 审批链 |
| **Security Scanning** | 安装时静态扫描 | 需要：安装扫描 + 运行时沙箱 + 定期审计 |
| **目录结构** | SKILL.md + references + scripts | 可沿用，加上版本和元数据 |
| **Secure Setup** | 本地 .env + config.yaml | 需要：Vault 集成 + 租户级凭证隔离 |
| **单一目录 (~/.hermes/skills/)** | 文件系统，读写自由 | 需要：数据库存储 + 组织树绑定 + 权限控制 |
| **外部目录 (external_dirs)** | 只读扫描，本地优先 | 可借鉴：平台级 Skill 只读继承，团队级可覆盖 |

---

## 三、借鉴改造设计

### 3.1 Progressive Disclosure — ToB 版

解决问题：**30+ Skill 的 tool_schema 一次性注入浪费 token**

```python
# worker/progressive_skill_loader.py

class ProgressiveSkillLoader:
    """
    渐进式 Skill 加载 — 参考 Hermes 的三级加载
    
    Level 0: 系统提示中只注入 Skill 索引（名称+描述）
             ~50 token/skill × 30 skills = ~1500 tokens
             相比全量 tool_schema（~300 token/skill × 30 = ~9000 tokens）节省 80%+
    
    Level 1: Agent 决定要用某个 Skill → 动态注入 tool_schema
    Level 2: Skill 需要参考文档 → 按需加载 references/
    """
    
    def build_level0_index(self, resolved_skills: List[ResolvedSkill]) -> str:
        """
        Level 0: 生成轻量 Skill 索引，嵌入到 system prompt
        
        格式:
        ## Available Skills
        When you need to use a skill, call `load_skill(name)` first.
        
        | Skill | Description | Category |
        |-------|-------------|----------|
        | crm_query | 查询 CRM 客户和商机数据 | 数据 |
        | erp_lookup | 查询 ERP 库存和订单 | 数据 |
        | approval_flow | 发起审批流程 | 办公 |
        ...
        """
        lines = ["## Available Skills",
                 "When you need to use a skill, call `load_skill(name)` first.\n",
                 "| Skill | Description | Category |",
                 "|-------|-------------|----------|"]
        
        for skill in resolved_skills:
            lines.append(f"| {skill.skill_name} | {skill.description[:60]} | {skill.category} |")
        
        return "\n".join(lines)
    
    
    def build_load_skill_tool(self) -> dict:
        """
        注册一个 meta-tool: load_skill
        Agent 调用此工具来加载 Skill 的完整 tool_schema
        """
        return {
            "type": "function",
            "function": {
                "name": "load_skill",
                "description": "加载一个 Skill 的完整工具定义。调用后该 Skill 的工具会变为可用。",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "name": {
                            "type": "string",
                            "description": "Skill 名称（从 Available Skills 列表中选择）"
                        }
                    },
                    "required": ["name"]
                }
            }
        }
    
    
    async def handle_load_skill(self, skill_name: str, 
                                 resolved_skills: Dict[str, ResolvedSkill]) -> LoadResult:
        """
        Level 1: Agent 请求加载某个 Skill
        
        返回:
        1. tool_schema（注入到后续 LLM 调用的 tools 中）
        2. system_prompt 片段（Skill 专用指令）
        3. config 信息（合并后的配置值）
        """
        skill = resolved_skills.get(skill_name)
        if not skill:
            return LoadResult(error=f"Skill '{skill_name}' 不存在或无权访问")
        
        return LoadResult(
            success=True,
            tool_schema=skill.tool_schema,
            system_prompt_fragment=skill.system_prompt,
            config_info=f"[Skill config: {json.dumps(skill.config, ensure_ascii=False)}]",
            references=skill.list_references(),  # 列出可加载的参考文件
        )
    
    
    async def handle_load_reference(self, skill_name: str, 
                                     ref_path: str) -> str:
        """
        Level 2: Agent 请求加载某个 Skill 的参考文档
        """
        content = await self.storage.read_skill_reference(skill_name, ref_path)
        return content
```

**在 Agent Loop 中的集成：**

```python
async def run_agent_loop_progressive(self, agent, messages, task, trace_id,
                                      resolved_skills):
    """改进后的 Agent Loop — 支持渐进式加载"""
    
    # 初始状态：只注入 Level 0 索引 + load_skill meta-tool
    active_tools = [self.skill_loader.build_load_skill_tool()]
    loaded_skills = {}
    
    # 将 Level 0 索引附加到系统消息
    index_text = self.skill_loader.build_level0_index(resolved_skills)
    messages[0]["content"] += f"\n\n{index_text}"
    
    for iteration in range(max_iterations):
        response = await self.llm_pool.chat_completion(
            model=agent.model_config.primary,
            messages=messages,
            tools=active_tools,
            ...
        )
        
        choice = response.choices[0]
        
        if choice.finish_reason == "stop":
            return AgentResult(final_text=choice.message.content, ...)
        
        if choice.finish_reason == "tool_calls":
            messages.append(choice.message)
            
            for tool_call in choice.message.tool_calls:
                tool_name = tool_call.function.name
                tool_args = json.loads(tool_call.function.arguments)
                
                if tool_name == "load_skill":
                    # ── Level 1: 动态加载 Skill ──
                    result = await self.skill_loader.handle_load_skill(
                        tool_args["name"], resolved_skills
                    )
                    if result.success:
                        # 将 Skill 的 tool_schema 加入 active_tools
                        active_tools.append(result.tool_schema)
                        loaded_skills[tool_args["name"]] = result
                        
                        # 注入 Skill 专用指令
                        if result.system_prompt_fragment:
                            messages.append({
                                "role": "system",
                                "content": result.system_prompt_fragment,
                            })
                    
                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "content": json.dumps({"loaded": result.success, 
                                               "config": result.config_info}),
                    })
                
                else:
                    # ── 正常工具调用 ──
                    tool_result = await self.tool_executor.execute(
                        tool_name, tool_args, ...
                    )
                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "content": json.dumps(tool_result),
                    })
```

### 3.2 Conditional Activation — ToB 版

在 Hermes 的基础上扩展更多条件维度：

```python
# resolver/conditional_activation.py

class ConditionalActivation:
    """
    Skill 条件激活 — 基于 Hermes 的 requires/fallback 扩展到 ToB 场景
    
    Hermes 的维度:
    - requires_toolsets / requires_tools
    - fallback_for_toolsets / fallback_for_tools
    - platforms (os)
    
    ToB 扩展维度:
    - requires_channel    → 仅在特定渠道可用
    - requires_chat_type  → 仅群聊/仅私聊
    - requires_agent_cap  → 仅当 Agent 有特定能力时
    - time_window         → 仅在指定时间段可用
    - feature_flag        → 结合功能开关
    - fallback_for_skill  → 某 Skill 不可用时的替代
    """
    
    @dataclass
    class ActivationCondition:
        # 继承 Hermes 的条件
        requires_tools: Optional[List[str]] = None
        fallback_for_tools: Optional[List[str]] = None
        platforms: Optional[List[str]] = None
        
        # ToB 扩展条件
        requires_channel: Optional[List[str]] = None       # ["feishu", "wecom"]
        requires_chat_type: Optional[List[str]] = None     # ["direct", "group"]
        requires_agent_capabilities: Optional[List[str]] = None  # ["code_execution", "web_browse"]
        time_window: Optional[dict] = None                 # {"from": "09:00", "to": "18:00", "weekdays": [1-5]}
        feature_flags: Optional[List[str]] = None          # ["beta_feature_x"]
        fallback_for_skills: Optional[List[str]] = None    # ["premium_crm"] → 当这个 Skill 不可用时激活
        min_user_tier: Optional[str] = None                # "premium" → 仅付费用户
    
    
    def evaluate(self, condition: ActivationCondition, 
                  context: ActivationContext) -> ActivationResult:
        """
        评估条件（所有 requires 必须满足，任一 fallback 条件触发则显示）
        """
        
        # requires 条件（AND）
        if condition.requires_tools:
            for tool in condition.requires_tools:
                if tool not in context.available_tools:
                    return ActivationResult(active=False, reason=f"缺少工具: {tool}")
        
        if condition.requires_channel:
            if context.channel not in condition.requires_channel:
                return ActivationResult(active=False, reason=f"渠道不匹配: {context.channel}")
        
        if condition.requires_chat_type:
            if context.chat_type not in condition.requires_chat_type:
                return ActivationResult(active=False, reason=f"聊天类型不匹配")
        
        if condition.platforms:
            if context.platform not in condition.platforms:
                return ActivationResult(active=False, reason=f"平台不匹配")
        
        if condition.time_window:
            if not self._check_time_window(condition.time_window):
                return ActivationResult(active=False, reason="不在生效时间窗口")
        
        if condition.feature_flags:
            for flag in condition.feature_flags:
                if not context.feature_flags.get(flag, False):
                    return ActivationResult(active=False, reason=f"功能开关未启用: {flag}")
        
        if condition.min_user_tier:
            if not self._tier_meets(context.user_tier, condition.min_user_tier):
                return ActivationResult(active=False, reason="用户等级不足")
        
        # fallback 条件（如果主 Skill 可用，则 fallback 隐藏）
        if condition.fallback_for_tools:
            for tool in condition.fallback_for_tools:
                if tool in context.available_tools:
                    return ActivationResult(active=False, reason=f"主工具 {tool} 可用，备选隐藏")
        
        if condition.fallback_for_skills:
            for skill in condition.fallback_for_skills:
                if skill in context.available_skills:
                    return ActivationResult(active=False, reason=f"主 Skill {skill} 可用，备选隐藏")
        
        return ActivationResult(active=True)
```

### 3.3 Agent-Managed Skills — ToB 版（受控的自学习）

Hermes 让 Agent 自由创建 Skill。ToB 需要约束：

```python
# skills/agent_skill_management.py

class AgentSkillManager:
    """
    Agent 自管理 Skill — ToB 受控版
    
    Hermes: Agent 自由创建/修改/删除
    ToB: Agent 创建 → 进入草稿 → 需要人工审核 → 才能发布
    
    场景:
    - Agent 处理了一个复杂工单（10+ tool calls），总结出一套流程
    - Agent 想保存这个流程给未来复用
    - 但 ToB 环境下，不能让 Agent 自己的"想法"直接变成全组织可用的 Skill
    """
    
    # ── 触发条件（参考 Hermes）──
    CREATION_TRIGGERS = {
        "complex_task_completed": {
            "condition": lambda run: run.tool_call_count >= 5 and run.status == "success",
            "description": "完成了 5+ 步的复杂任务",
        },
        "error_recovery": {
            "condition": lambda run: run.error_count > 0 and run.status == "success",
            "description": "经历错误后找到了正确路径",
        },
        "user_correction": {
            "condition": lambda run: run.user_corrections > 0,
            "description": "用户纠正后掌握了正确方法",
        },
        "repeated_workflow": {
            "condition": lambda run: run.similar_past_runs >= 3,
            "description": "相似工作流已执行 3+ 次",
        },
    }
    
    
    async def propose_skill_creation(self, run: AgentRunRecord) -> Optional[SkillProposal]:
        """
        Agent Run 结束后，检查是否应该提议创建 Skill
        注意：只是提议，不直接创建
        """
        for trigger_name, trigger in self.CREATION_TRIGGERS.items():
            if trigger["condition"](run):
                return SkillProposal(
                    trigger=trigger_name,
                    trigger_description=trigger["description"],
                    proposed_by_agent=run.agent_id,
                    proposed_at=datetime.now(),
                    source_trace_id=run.trace_id,
                    source_session_id=run.session_id,
                    
                    # Agent 生成的 Skill 草稿
                    draft=SkillDraft(
                        name=None,          # 待 Agent 或人工命名
                        description=None,   # 待生成
                        procedure=None,     # 待从 run 中提炼
                        tool_calls=run.tool_calls,  # 原始调用链
                    ),
                )
        return None
    
    
    async def create_draft_from_run(self, proposal: SkillProposal, 
                                     llm_client) -> SkillDraft:
        """
        用 LLM 从执行记录中提炼 Skill 草稿
        """
        prompt = f"""你是一个 Skill 设计专家。根据以下 Agent 执行记录，提炼出一个可复用的 Skill。

执行记录:
{json.dumps([{
    'step': i+1,
    'tool': tc.tool_name,
    'params_summary': self._summarize_params(tc.params),
    'result_summary': self._summarize_result(tc.result),
    'success': tc.status == 'success',
} for i, tc in enumerate(proposal.draft.tool_calls)], ensure_ascii=False, indent=2)}

触发原因: {proposal.trigger_description}

请生成 Skill 定义:
1. name: 简短英文名
2. description: 一行中文描述
3. procedure: 分步骤指令（Markdown）
4. pitfalls: 已知问题和处理方法
5. tool_dependencies: 需要哪些工具

返回 JSON。"""
        
        response = await llm_client.chat(
            model="qwen-max",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3,
        )
        
        draft_data = json.loads(response.content)
        
        return SkillDraft(
            name=draft_data["name"],
            description=draft_data["description"],
            procedure=draft_data["procedure"],
            pitfalls=draft_data.get("pitfalls", ""),
            tool_dependencies=draft_data.get("tool_dependencies", []),
            status="draft",  # ← 关键：只是草稿
            created_by="agent",
            source_run=proposal.source_trace_id,
        )
    
    
    # ── ToB 管控流程 ──
    
    SKILL_LIFECYCLE = """
    Agent 完成复杂任务
           │
           ▼
    提议创建 Skill（SkillProposal）
           │
           ▼
    LLM 提炼 Skill 草稿（SkillDraft, status=draft）
           │
           ▼
    通知团队管理员/Skill Owner 审核
           │
       ┌───┴───┐
       │       │
     批准     拒绝（附理由）
       │       │
       ▼       ▼
    status=   归档/丢弃
    canary
       │
       ▼
    灰度验证 → stable（走正常版本迭代流程）
    """
    
    
    async def submit_for_review(self, draft: SkillDraft, 
                                 context: SecurityContext):
        """
        提交 Agent 创建的 Skill 草稿给管理员审核
        
        审核要点:
        1. Skill 逻辑是否正确（不是 Agent 幻觉）
        2. 是否包含敏感操作
        3. 是否与现有 Skill 重复
        4. 命名和描述是否规范
        """
        # 写入 pending_reviews 表
        review = SkillReview(
            draft=draft,
            submitted_by=context.agent_id,
            submitted_for_tenant=context.tenant_id,
            submitted_for_node=context.department_id,
            review_status="pending",
        )
        
        await self.db.insert("skill_reviews", review)
        
        # 通知审核人
        await self.notify_reviewers(review, context)
        
        return review
```

### 3.4 Skills Hub — ToB 企业版

```python
# hub/enterprise_skill_hub.py

class EnterpriseSkillHub:
    """
    企业级 Skill Hub — 参考 Hermes 的多源集成，适配 ToB
    
    来源层级（信任从高到低）:
    
    ┌──────────────────────────────────────────────────┐
    │  Level 5: platform_builtin  ← 平台内置，最高信任   │
    │           类似 Hermes 的 builtin                   │
    ├──────────────────────────────────────────────────┤
    │  Level 4: enterprise_official  ← 企业 IT 部发布    │
    │           类似 Hermes 的 official                  │
    ├──────────────────────────────────────────────────┤
    │  Level 3: division_shared  ← BU/事业部级共享       │
    │           自研 Skill，经过 BU 审核                  │
    ├──────────────────────────────────────────────────┤
    │  Level 2: department_custom  ← 部门定制            │
    │           部门自建，经部门管理员审核                  │
    ├──────────────────────────────────────────────────┤
    │  Level 1: marketplace_verified  ← 第三方市场（已审核）│
    │           类似 Hermes 的 trusted                   │
    ├──────────────────────────────────────────────────┤
    │  Level 0: marketplace_community  ← 社区贡献（需审核）│
    │           类似 Hermes 的 community                 │
    └──────────────────────────────────────────────────┘
    """
    
    TRUST_LEVELS = {
        "platform_builtin":      {"auto_approve": True,  "scan": False, "sandbox_test": False},
        "enterprise_official":   {"auto_approve": True,  "scan": True,  "sandbox_test": False},
        "division_shared":       {"auto_approve": False, "scan": True,  "sandbox_test": True},
        "department_custom":     {"auto_approve": False, "scan": True,  "sandbox_test": True},
        "marketplace_verified":  {"auto_approve": False, "scan": True,  "sandbox_test": True},
        "marketplace_community": {"auto_approve": False, "scan": True,  "sandbox_test": True, "quarantine": True},
    }
    
    
    async def install_skill(self, skill_ref: str, target_node: str, 
                             context: SecurityContext) -> InstallResult:
        """
        安装 Skill 到指定组织节点
        
        流程:
        1. 解析来源和信任等级
        2. 安全扫描（参考 Hermes 的检查项）
        3. 沙箱测试（如需要）
        4. 根据信任等级决定：自动批准 / 进入审核队列 / 进入隔离区
        5. 创建 Skill 版本 + 组织节点绑定
        """
        # 1. 解析
        source = self.resolve_source(skill_ref)
        trust = self.TRUST_LEVELS[source.trust_level]
        
        # 2. 安全扫描
        if trust["scan"]:
            scan_result = await self.security_scanner.scan(source.package)
            if scan_result.has_dangerous_findings:
                return InstallResult(
                    blocked=True, 
                    reason=f"安全扫描发现危险问题: {scan_result.findings}"
                )
            if scan_result.has_warnings and not context.force:
                return InstallResult(
                    needs_confirmation=True,
                    warnings=scan_result.findings,
                    message="发现非致命安全警告，使用 --force 确认安装"
                )
        
        # 3. 沙箱测试
        if trust["sandbox_test"]:
            test_result = await self.sandbox_tester.test(source.package)
            if not test_result.passed:
                return InstallResult(blocked=True, reason=f"沙箱测试失败: {test_result.errors}")
        
        # 4. 审批
        if trust.get("quarantine"):
            # 社区 Skill → 先进隔离区，等管理员审核
            await self.quarantine(source.package, target_node, context)
            return InstallResult(quarantined=True, message="已进入隔离区，等待管理员审核")
        
        if not trust["auto_approve"]:
            await self.submit_install_review(source.package, target_node, context)
            return InstallResult(pending_review=True, message="已提交审核")
        
        # 5. 安装
        return await self._do_install(source.package, target_node, context)
```

### 3.5 Skill 定义格式 — ToB 增强版

在 Hermes 的 SKILL.md 格式上，增加 ToB 必需的字段：

```yaml
---
# === 基础信息（兼容 Hermes/agentskills.io 标准）===
name: crm-query
description: 查询 CRM 系统中的客户、商机和合同数据
version: 2.0.0
author: 数据平台团队
license: proprietary

# === 平台/条件（沿用 Hermes）===
platforms: [linux]
metadata:
  hermes:  # 兼容 Hermes 格式
    tags: [数据, CRM, 销售]
    category: business-data
    requires_tools: [database_query]

# === ToB 扩展字段 ===
enterprise:
  # 信任等级
  trust_level: enterprise_official
  
  # 组织归属
  owner_node: tenant_acme
  owner_team: data-platform
  
  # 安全等级
  security:
    risk_level: medium              # low / medium / high / critical
    data_classification: internal   # public / internal / confidential / restricted
    pii_involved: true              # 是否涉及 PII 数据
    requires_approval: false        # 是否需要人工审批才能调用
    audit_level: full               # none / basic / full
  
  # 条件激活（扩展 Hermes 的 requires/fallback）
  activation:
    requires_channel: [feishu, wecom]
    requires_chat_type: [direct, group]
    time_window:
      from: "08:00"
      to: "22:00"
      weekdays: [1, 2, 3, 4, 5]
    feature_flags: []
    fallback_for_skills: [premium_crm_analytics]
    min_user_tier: standard
  
  # 配额
  quotas:
    per_user_daily: 200
    per_tenant_daily: 10000
  
  # 版本约束
  version_policy:
    min_compatible: "1.5.0"         # 最低兼容版本
    breaking_from: "1.0.0"          # 与此版本不兼容

# === 凭证要求（沿用 Hermes 格式 + 扩展）===
required_environment_variables:
  - name: CRM_SERVICE_TOKEN
    prompt: "CRM 服务访问 Token"
    help: "从内部凭证管理系统获取"
    required_for: "CRM API 访问"
    vault_ref: "secret/crm/service_token"  # ToB 扩展：Vault 引用

# === 配置（沿用 Hermes config 格式 + 扩展）===
config:
  - key: crm.default_region
    description: "默认查询区域"
    default: "all"
    prompt: "CRM 默认查询区域"
    inheritable: true               # ToB 扩展：此配置可被组织继承覆盖
  - key: crm.max_results
    description: "单次查询最大返回数"
    default: 50
    inheritable: true
---

# CRM 数据查询

## When to Use
- 用户询问客户信息、商机状态、合同详情时
- 用户需要销售数据分析时
- 用户提到 CRM、客户管理、销售管线时

## Quick Reference
| 操作 | 说明 |
|------|------|
| 查询客户 | query_type=customer, filters={name, region, ...} |
| 查询商机 | query_type=opportunity, filters={stage, owner, ...} |
| 查询合同 | query_type=contract, filters={status, amount_range, ...} |

## Procedure
1. 确认用户要查询的数据类型（客户/商机/合同）
2. 提取过滤条件
3. 调用 crm_query 工具
4. 格式化结果返回给用户

## Pitfalls
- 大数据量查询必须加 limit，默认 50
- 跨区域查询需要 region 参数，否则只查默认区域
- 合同金额字段单位为分（不是元）

## Verification
- 确认返回的数据条数合理
- 检查 region 是否符合用户意图
```

---

## 四、需要反向更新的文档

Hermes 参考带来的设计改进需要回写到以下文档：

| 文档 | 需要更新的内容 |
|---|---|
| `skill-inheritance-and-versioning.md` | 增加 Conditional Activation 条件、Progressive Disclosure 索引、Agent-Managed Skill 审核流程 |
| `agent-security-design.md` | 增加 Skill 安装安全扫描（4 项检查）、信任分级、隔离区机制 |
| `multi-agent-gateway-architecture.md` | Worker 执行层改为渐进式加载、Agent Loop 增加 `load_skill` meta-tool |
| `agent-routing-design.md` | 路由层可考虑 Skill 可用性作为路由参考（某 Agent 没有合适 Skill 则不路由） |

---

## 五、总结：从 Hermes 学到什么

```
Hermes 的设计哲学（ToC）:
  → Agent 是主人，Skill 是它的工具箱
  → 自由创建/修改/删除，文件系统即真理
  → 信任开发者，安全扫描是辅助

我们的 ToB 改造哲学:
  → Agent 是执行者，Skill 是组织授权的能力
  → 创建需审核，修改需版本，删除需权限
  → 信任组织流程，安全是硬约束

保留 Hermes 的精华:
  ✅ Progressive Disclosure（省 Token，按需加载）
  ✅ Conditional Activation（条件激活，减少噪声）
  ✅ SKILL.md 标准格式（兼容 agentskills.io）
  ✅ Skills Hub 多源发现（统一安装体验）
  ✅ 安全扫描分级信任
  ✅ Agent 可提议创建 Skill（自学习能力）

ToB 必须增强的:
  🔒 Agent 创建的 Skill 不能自动生效（需审核）
  🔒 组织树绑定 + 权限继承（不是文件系统层叠）
  🔒 版本管理 + 灰度发布（不是直接覆盖文件）
  🔒 多租户隔离（Skill A 对租户 X 可见，对 Y 不可见）
  🔒 配额计量 + 审计追溯
  🔒 凭证不走文件，走 Vault
```
