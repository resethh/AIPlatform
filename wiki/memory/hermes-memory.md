# Hermes 记忆系统

## 一句话定位
插件化多层记忆架构，BuiltinProvider 管理本地 MEMORY.md/USER.md，可接入 Honcho/Mem0 等外部 Provider；核心设计是"冻结快照"保持前缀缓存稳定。

## 架构分层

```
MemoryManager（中央协调器：路由工具调用、聚合注入内容、调度预取）
    ├── BuiltinMemoryProvider  → MEMORY.md / USER.md（本地文件）
    └── ExternalMemoryProvider → Honcho / Hindsight / Mem0（可选插件）
              ↑
    MemoryProvider (ABC)  统一插件接口
```

## 内置 Provider 关键设计

### 数据结构

| 文件 | 内容 | 字符上限 |
|---|---|---|
| `~/.hermes/memories/<profile>/MEMORY.md` | Agent 笔记、事实、偏好 | 2200 字符 |
| `~/.hermes/memories/<profile>/USER.md` | 用户画像：角色、习惯 | 1375 字符 |

- 条目分隔符：`§`（段落符），支持多行
- 基于字符计数而非 Token 计数（与模型无关）

### 冻结快照机制

```python
_system_prompt_snapshot: Dict   # Session 开始时冻结
```

- Session 启动时固定快照，中途即使 Agent 写入了新记忆，当前 Session 也不会生效。
- **目的**：保持系统提示前缀稳定，最大化 LLM 的 Prompt Cache 命中率，降低 Token 成本。
- **代价**：新记忆需等到下次 Session 才能被利用。

### 非阻塞 I/O

记忆写入操作全部异步执行，不阻塞对话响应链路。

## 外部 Provider 接入（Honcho 示例）

Honcho 实现三层记忆结构，通过 `MemoryProvider` ABC 接入后，`MemoryManager` 统一聚合其返回内容注入系统提示。

## 上下游依赖

- 上游: [[architecture/hermes.md]]（MemoryManager 由 prompt_builder 调用，在组装系统提示时注入）
- 对比: [[memory/deerflow-memory.md]]（DeerFlow 动态 JSON vs Hermes 冻结快照的不同权衡）
