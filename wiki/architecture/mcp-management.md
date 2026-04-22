# MCP Server 管理方案

> 参考实现: [[raw/HiClaw 1.0.6：企业级 MCP Server 管理 — 凭证零暴露，工具全接入.md]]
> 更新日期: 2026-04-22

## 一句话定位
企业级 MCP Server 管理体系：通过 **Manager-Worker 双角色 + AI Gateway 凭证代理**，实现"工具全接入 + 真实凭证零暴露"——Worker 只拿可吊销的 consumer token，真实 API key / Bearer token / Service Token 永不进入 Worker 进程。

---

## 核心设计决策（ADR）

- **决策**：MCP 采用 Manager-Worker 分离架构，凭证只存 Manager/Gateway 一侧，Worker 通过 Gateway 调用 MCP 工具
- **背景**：AI Agent 运行在不可信环境（容器可逃逸、Prompt 可注入、日志可被读取），任何进入 Worker 进程的真实凭证都有泄露风险。传统"把 API key 注入环境变量"做不到细粒度吊销和审计
- **后果**：新增 Gateway 层（增加一跳网络延迟 ~5ms），但获得"凭证漏一次只影响 Worker 级别""吊销 token 不需要轮换真实凭证"的安全收益

---

## MCP 与 Skill 的定位差异

MCP 和 Skill 不是替代关系，是**分层协同**：

| 层次 | 职责 | 粒度 | 演进速度 |
|---|---|---|---|
| **MCP 工具** | 原子能力：一个 API 端点 = 一个工具 | 细粒度（单一动作） | 慢（接口稳定） |
| **Skill** | 场景能力包：把多个工具编排成一个可复用的工作流 | 场景级 | 快（迭代频繁） |

```
MCP 工具（原子能力：call_api / read_db / send_message）
    ↓ Skill 编排
SKILL（场景能力包：crm-query / sales-report / ticket-routing）
    ↓ 实战使用与迭代
SKILL 自进化（用户纠正 / Agent 修补）
```

**边界**：
- **权限治理**交给 MCP 层（工具粒度 RBAC、凭证隔离、速率限制）
- **场景演进**交给 Skill 层（自由迭代、审核发布、沙箱评估）

---

## 架构链路

```
用户
  ↓
Manager Claw（管控平面）
  │ ① 管理员描述 API + 提交真实凭证
  │ ② 生成 MCP Server YAML 配置
  │ ③ 部署到 Gateway
  │ ④ 用 mcporter 验证连通性
  ↓
Higress AI Gateway（凭证代理层）
  │ 存真实凭证（Vault）
  │ 签发 consumer token 给 Worker
  │ 代理所有 MCP 调用，按 token 做权限校验
  │ 记录审计日志
  ↓
Worker Claw（执行平面）
  │ 拉取 MCP 配置（只含 consumer token + 工具 schema）
  │ mcporter 发现工具 → 自动生成 SKILL.md
  │ 调用工具时走 Gateway，看不到真实凭证
```

**关键数据流**：
- 真实凭证：User → Manager → Gateway（**Worker 永远看不到**）
- Consumer token：Gateway → Worker（可随时吊销，作用域限定）
- 工具调用：Worker → Gateway → 真实 API（Gateway 替换为真实凭证转发）

---

## 凭证零暴露模型

### Worker 可以做的
- 调用所有已授权的 MCP 工具
- 通过 Gateway 使用工具（consumer token 鉴权）
- 基于工具调用经验生成 / 修改 SKILL.md
- 在授权范围内自主工作

### Worker 不能做的
- 读取任何真实凭证（环境变量 / 文件 / 内存里都没有）
- 调用未授权的 MCP Server
- 从 Gateway 反向提取凭证
- 跨 Worker 共享凭证

### Worker 被攻破的影响范围
1. 攻击者拿到的只是 consumer token（仅在本系统内有效）
2. Manager 立即吊销该 token，无需轮换任何真实凭证
3. 真实 API key 安全不受影响
4. 创建新 Worker，几分钟内恢复工作

---

## 工具批量接入能力

### 从 Swagger/OpenAPI 导入

```
管理员：这是产品目录 API 的 Swagger：https://docs.internal/swagger.json
       通过 X-API-Key 认证，Key: prod_cat_xxx

Manager：
  - 解析 Swagger → 发现 12 个端点
  - 自动生成 12 个 MCP 工具定义
  - 部署到 Gateway，命名空间 product-catalog
  - 通知相关 Worker 拉取配置
```

### 从 curl 命令导入

```
管理员：curl -X GET "https://api.shipping.com/v1/track?tracking_id=ABC" -H "X-API-Key: ship_xxx"

Manager：
  - 解析 curl → 推断工具 schema（路径、参数、认证头）
  - 创建 MCP Server shipping，工具 track_package
  - Gateway 部署，凭证存 Vault
```

### 价值

传统方案需要工程师为每个 API 写 MCP 适配代码，批量接入成本高。Swagger/curl 直接导入让**运维人员自助接入 API**，Agent 几分钟内就能用上新工具。

---

## Worker 自生成 SKILL.md

首次调用新 MCP Server 时，Worker 自动执行：

```
1. mcporter list {server} --schema    → 发现所有工具及参数
2. mcporter call {server}.{tool}      → 测试代表性工具
3. 基于调用结果生成 SKILL.md：
   - 自然语言说明（LLM 总结工具用途）
   - 典型调用示例
   - 参数说明 + 陷阱
   - 错误处理注意事项
4. 写入个人 Skill 空间（skills/{tenant_id}/personal/{user_id}/）
```

后续调用该工具时，走 Skill 路径而非裸调 MCP，享受 Skill 的所有治理能力（条件激活、配额、降级等）。

---

## 与中台 Skill 管理的协同

- **凭证管理**：[[skills/skill-manage.md]] #10 API Key / 凭证管理 的企业级实现，替代 `~/.hermes/.env` 的本地文件模式
- **自生成 Skill**：Worker 首次探索新 MCP → 生成 Skill 草稿 → 走 #15 Agent 提议审核流程
- **工具条件激活**：MCP 工具的 `requires_toolsets` 可直接对应 Gateway 配置的 MCP Server 列表

---

## 上下游依赖

- 上游: [[architecture/gateway.md]] Worker 由网关路由触发
- 下游: [[skills/skill-manage.md]] Skill 编排 MCP 工具为场景能力
- 安全: [[security/agent-security.md]] 凭证隔离 + Worker 攻破兜底
