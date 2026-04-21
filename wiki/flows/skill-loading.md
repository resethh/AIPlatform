# Skill 渐进式加载时序

> 涉及组件: [[skills/skill-iteration.md]] / [[skills/hermes-skills.md]] / [[architecture/deerflow-agent.md]]
> 更新日期: 2026-04-21

## 概述

Hermes 和 DeerFlow 都实现了渐进式 Skill 加载，但机制不同：Hermes 通过 `load_skill` meta-tool 按需注入 tool_schema；DeerFlow 通过 `read_file` 读取 SKILL.md 文件内容。两者的共同目标都是让系统提示只含索引（~1500 token），用的时候才加载全量内容（节省 80%+ token）。

---

## 一、Hermes Skill 加载时序

```mermaid
sequenceDiagram
    autonumber
    participant Worker as AgentWorker
    participant Cache as Skill 三层缓存<br/>(LRU → 磁盘 → 文件)
    participant Agent as Agent Loop
    participant LLM as LLM API

    Worker->>Cache: build_skills_system_prompt()
    Cache-->>Worker: Level 0 索引（名称+描述，~1500 token）
    note over Worker,Cache: 三层缓存命中路径：LRU miss → 磁盘快照 → 冷扫描

    Worker->>Agent: 启动（System Prompt 含 Level 0 + load_skill meta-tool）

    Agent->>LLM: 第 1 轮请求（active_tools 只有 load_skill）
    LLM-->>Agent: tool_call: load_skill("crm-query")

    Agent->>Cache: load_skill("crm-query")
    Cache-->>Agent: Level 1 — 完整 SKILL.md + tool_schema
    note over Agent: 把 tool_schema 追加到 active_tools

    Agent->>LLM: 第 2 轮请求（active_tools 增加了 crm-query）
    LLM-->>Agent: tool_call: crm_query(params)

    alt Skill 需要参考文档
        Agent->>Cache: skill_view("crm-query", path="references/api.md")
        Cache-->>Agent: Level 2 — 子文件内容
    end

    Agent->>LLM: 继续执行...
    LLM-->>Agent: stop（任务完成）
```

### 关键设计点

| 层级 | 内容 | Token 消耗 |
|---|---|---|
| Level 0（索引） | 所有 Skill 名称 + 描述 | ~1500 token |
| Level 1（schema） | 单个 Skill 的完整 SKILL.md + tool_schema | ~300-800 token/个 |
| Level 2（子文件） | references/ templates/ scripts/ 内的文件 | 按需 |

- 已加载的 Skill 在后续所有轮次继续可用（追加到 `active_tools`，不重复加载）
- 三层缓存失效：`skill_manage` 操作后主动 `clear()`，下次请求走冷扫描重建

---

## 二、DeerFlow Skill 加载时序

```mermaid
sequenceDiagram
    autonumber
    participant Lead as Lead Agent
    participant Prompt as Prompt Builder
    participant FS as 文件系统<br/>(/mnt/skills/)
    participant LLM as LLM API

    Lead->>Prompt: make_lead_agent(config)
    Prompt->>FS: load_skills(enabled_only=True)
    FS-->>Prompt: Skill 元数据列表（name + description + location）
    Prompt-->>Lead: 生成 <skill_system> XML 注入 System Prompt

    Lead->>LLM: 第 1 轮请求（System Prompt 含 Skill 索引）
    LLM-->>Lead: tool_call: read_file("/mnt/skills/public/data-analysis/SKILL.md")

    Lead->>FS: read_file(path)
    note over Lead,FS: 路径转换：/mnt/skills/ → 宿主机真实路径（只读）
    FS-->>Lead: SKILL.md 全文内容（工作流说明）

    LLM-->>Lead: 解析工作流，执行 Step 1
    Lead->>FS: bash("python /mnt/skills/.../analyze.py --action inspect")

    alt Skill 引用子文件
        LLM-->>Lead: tool_call: read_file("references/guide.md")
        Lead->>FS: read_file(sub_path)
        FS-->>Lead: 子文件内容
    end

    Lead->>LLM: 继续执行后续步骤...
    LLM-->>Lead: stop（任务完成）
```

---

## 三、Hermes vs DeerFlow 对比

| 维度 | Hermes | DeerFlow |
|---|---|---|
| 索引注入 | 系统提示文本（`<available_skills>`） | 系统提示 XML（含文件路径） |
| 触发工具 | `load_skill(name)` meta-tool | `read_file(path)` 通用工具 |
| 加载内容 | tool_schema（注册为可调用工具） | SKILL.md 文本（LLM 按文本执行） |
| 缓存 | 三层缓存（LRU + 磁盘快照 + 冷扫描） | 无缓存（每次 make_lead_agent 重加载） |
| 安全 | 创建/安装/view 三次安全扫描 | 安装时验证路径/名称；/mnt/skills/ 只读 |
| token 节省 | Level 0 ~1500 vs 全量 ~9000（节省 80%+） | 类似（只注入描述+路径，不含正文） |
