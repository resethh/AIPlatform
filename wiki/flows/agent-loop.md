# Agent 执行循环时序

> 涉及组件: [[architecture/hermes.md]] / [[architecture/deerflow-agent.md]] / [[memory/memory-management.md]]
> 更新日期: 2026-04-21

## 概述

Agent 执行循环（Agent Loop）是 Agent 与 LLM 之间的核心交互模式：LLM 返回工具调用则执行工具并将结果喂回，返回纯文本则退出。Hermes 和 DeerFlow 在实现方式上有显著差异。

---

## 一、Hermes Agent Loop

```mermaid
sequenceDiagram
    autonumber
    participant Worker as AgentWorker
    participant Prompt as prompt_builder.py
    participant LLM as LLM API
    participant Tools as ToolRegistry
    participant Memory as Memory<br/>(MEMORY.md)
    participant Skills as Skill Manager

    Worker->>Prompt: build_system_prompt()
    note over Prompt: 角色 + 记忆快照（冻结）+ Skill 索引(Level 0) + Tool Schema
    Prompt-->>Worker: system_prompt（Session 内不变）

    Worker->>LLM: 第 1 轮（system_prompt + 对话历史）

    loop 最多 90 次 API 调用
        LLM-->>Worker: 响应

        alt finish_reason = "tool_calls"
            Worker->>Tools: dispatch(tool_name, args)

            alt tool = load_skill
                Tools->>Skills: skill_view(name)
                Skills-->>Tools: Level 1 schema
                Tools-->>Worker: 追加 tool_schema 到 active_tools
            else tool = skill_manage(patch)
                Tools->>Skills: 修补 SKILL.md，清除缓存
                Skills-->>Tools: 成功
            else tool = 记忆相关工具
                Tools->>Memory: read/edit/write（非阻塞）
                Memory-->>Tools: 结果
            else 业务工具
                Tools->>Tools: 执行（web_search / sql_query 等）
                Tools-->>Worker: 工具结果
            end

            Worker->>LLM: 下一轮（携带 ToolMessage）

        else finish_reason = "stop"
            Worker-->>Worker: 退出循环
        end
    end

    Worker-->>Worker: 返回最终回复
```

**关键参数**：最多 90 次 API 调用；每轮携带全量工具 schema（不像 DeerFlow 有 Deferred Tool）；记忆更新是主线程内的同步工具调用（但 Agent 通常不阻塞等结果）。

---

## 二、DeerFlow Agent Loop（LangGraph 版）

```mermaid
sequenceDiagram
    autonumber
    participant MW as 中间件链<br/>(13 层)
    participant Agent as LangGraph Agent Node
    participant LLM as LLM API
    participant Tools as Tool Executor
    participant State as ThreadState<br/>(Checkpointer)

    note over MW,State: 每次用户消息触发一次 Graph 运行

    MW->>Agent: 前置中间件处理<br/>(SandboxMiddleware / UploadsMiddleware 等)
    Agent->>State: 读取当前 ThreadState（含对话历史）

    loop 直到 finish_reason=stop 或超出 max_turns
        Agent->>LLM: 请求（messages + tool_schemas）
        LLM-->>Agent: 响应

        alt tool_call
            Agent->>Tools: 执行工具
            note over Tools: bash / read_file / task(委派子Agent) 等
            Tools-->>Agent: 工具结果（ToolMessage）
            Agent->>State: 更新 State（追加 messages）
        else stop
            Agent-->>Agent: 退出循环
        end
    end

    Agent->>MW: 后置中间件处理
    note over MW: MemoryMiddleware 异步写记忆<br/>TitleMiddleware 生成标题<br/>LoopDetectionMiddleware 已过滤

    MW->>State: 持久化最终 ThreadState（Checkpointer）
    MW-->>MW: 返回最终回复
```

**关键参数**：无硬性轮次上限（由中间件 LoopDetectionMiddleware 检测重复调用退出）；SummarizationMiddleware 在接近 token 限制时自动压缩；MemoryMiddleware 异步写记忆不阻塞主流程。

---

## 三、两种 Loop 设计对比

| 维度 | Hermes | DeerFlow |
|---|---|---|
| 实现方式 | 自定义 `run_conversation()` 循环 | LangGraph StateGraph |
| 最大轮次 | 硬限制 90 次 API 调用 | 无硬限制，LoopDetection 软检测 |
| 状态管理 | 对话历史存内存，Session 结束写文件 | LangGraph Checkpointer 持久化 |
| 上下文压缩 | `context_compressor.py` 滑动窗口 | SummarizationMiddleware 按需压缩 |
| 工具调度 | `ToolRegistry.dispatch()`，同步阻塞 | LangGraph Tool Node，支持并发 |
| 记忆更新 | Agent 主动调工具写文件（inline） | MemoryMiddleware 异步队列（解耦） |
| Skill 加载 | `load_skill` meta-tool 动态追加 schema | `read_file` 读 SKILL.md 文本 |
| 断点恢复 | 不支持 | ✅ Checkpointer 支持跨 Session 恢复 |
