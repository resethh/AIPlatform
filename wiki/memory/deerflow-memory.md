# DeerFlow 记忆系统

## 一句话定位
DeerFlow 的长期记忆实现：对话结束后用 LLM 提取关键信息写入 JSON，下次对话开始时注入系统提示，实现跨 Session 个性化。

## 数据结构

```json
{
  "version": "1.0",
  "user": {
    "workContext":     {"summary": "...", "updatedAt": "..."},
    "personalContext": {"summary": "...", "updatedAt": "..."},
    "topOfMind":       {"summary": "...", "updatedAt": "..."}
  },
  "history": {
    "recentMonths":       {"summary": "...", "updatedAt": "..."},
    "earlierContext":     {"summary": "...", "updatedAt": "..."},
    "longTermBackground": {"summary": "...", "updatedAt": "..."}
  },
  "facts": [
    {"id": "fact_abc123", "content": "...", "category": "preference", "confidence": 0.9}
  ]
}
```

三大类型：`user`（用户画像）、`history`（历史摘要）、`facts`（结构化事实）。

## 关键模块

| 文件 | 职责 |
|---|---|
| `memory/updater.py` | 记忆读写 + LLM 更新逻辑 |
| `memory/queue.py` | 防抖异步更新队列（`MemoryUpdateQueue`） |
| `memory/prompt.py` | 更新 Prompt 模板 + 格式化注入工具 |

## 防抖异步队列设计

- `ConversationContext` 数据类：封装待处理对话的上下文
- `MemoryUpdateQueue`：收集多条对话后批量触发 LLM 提取，避免每条消息都调一次 LLM
- 注入函数 `format_memory_for_injection()`：把记忆格式化为 System Prompt 片段

## 上下游依赖

- 上游: [[architecture/deerflow.md]]（Agent Loop 在会话结束后调用队列）
- 下游: [[context/deerflow-context.md]]（Memory Injection 注入系统提示）
- 对比: [[memory/hermes-memory.md]]（冻结快照 vs DeerFlow 的动态更新）
