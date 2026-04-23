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

## 动态凭证支持

静态 API key 只是最简单的情形。生产环境大量 API 使用**动态凭证**——需要运行时获取、按 TTL 刷新、按请求签名。这类场景下 Gateway 的角色从"代理静态凭证"升级为 **凭证获取与生命周期管理器**，Worker 依然完全无感。

### 典型动态凭证类型

| 场景 | 管理员配置（根凭证） | Gateway 运行时行为 |
|---|---|---|
| **OAuth2 client_credentials**（服务间） | `client_id` + `client_secret` + token endpoint | 首次调用换 access_token，按 `expires_in` 缓存，过期前 60s 主动刷新 |
| **OAuth2 authorization_code**（用户级，如 GitHub / Google） | `client_id` + `client_secret` + redirect_uri | 用户首次使用走浏览器授权流，refresh_token 按 user_id 存 Vault；调用前换 access_token |
| **AWS STS / AssumeRole** | base IAM 凭证 + `role_arn` + `external_id` | 每次调用（或 15 分钟缓存）前 AssumeRole，用临时凭证签名转发 |
| **AWS SigV4** | `access_key` + `secret_key` | 每次转发按请求路径 / headers / body 计算签名，加到 Authorization 头 |
| **JWT 服务账号**（GCP / 企业内部） | signing key（RSA 私钥）+ 声明模板 | 按 TTL 缓存签发的 JWT，过期前重签 |
| **Session-based login** | 用户名/密码 或 登录凭证 | 维护 session pool，cookie 过期或 401 时重登录 |
| **mTLS**（客户端证书） | 客户端证书 + 私钥 | 转发时用证书建立 TLS 通道，Worker 无感知 |

### 统一抽象：credential_provider 插件

MCP Server YAML 里声明认证类型，Gateway 加载对应的 provider 插件：

```yaml
mcp_server: billing
endpoint: https://billing.internal.company.com
auth:
  type: oauth2_client_credentials
  provider_config:
    token_url: https://auth.internal.company.com/oauth/token
    client_id_ref: vault://oauth/billing/client_id
    client_secret_ref: vault://oauth/billing/client_secret
    scope: "billing:read billing:write"
  cache:
    refresh_before_expiry_sec: 60
```

Provider 统一接口：

```python
class CredentialProvider:
    def get_credential(self, request_context) -> dict:
        """返回当前该请求的认证 headers / params"""

    def on_auth_failure(self, response) -> bool:
        """401/403 时是否重试（刷新凭证后重试一次）"""
```

已知插件库：`static_api_key` / `oauth2_cc` / `oauth2_ac` / `aws_sts` / `aws_sigv4` / `jwt_service_account` / `session_login` / `mtls`。

### Swagger 导入的扩展流程

```
管理员：Swagger URL + 认证配置
Manager：
  ① 解析 Swagger 的 securitySchemes 块（OpenAPI 标准字段）
     - securitySchemes.oauth2   → auth_type=oauth2
     - securitySchemes.http     → bearer → auth_type=static_bearer
     - securitySchemes.apiKey   → auth_type=static_api_key
  ② 根据 auth_type 选对应 credential_provider 插件
  ③ 动态凭证：让管理员补充根凭证（OAuth client_secret / STS base key / JWT signing key 等）
  ④ 部署到 Gateway，provider 插件启动
```

### 关键设计要点

1. **根凭证 vs 运行时凭证**：OAuth 的 `client_secret`、STS 的 IAM base key 是"根凭证"——配置一次、长期有效、存 Vault；换出来的短期 token 在 Gateway 内存里，不落盘
2. **缓存粒度**：服务级凭证（client_credentials）按 `mcp_server` 缓存；用户级凭证（authorization_code）按 `mcp_server + user_id` 缓存
3. **失败重试**：遇 401/403 时 provider 自动 refresh 一次再重试，第二次仍失败才返回错误给 Worker
4. **审计**：每次凭证获取 / 刷新 / 失败都写审计日志，追踪"谁代表谁调用了什么"
5. **对 Worker 完全透明**：Worker 始终只写 `mcporter.call("billing.get_customer", ...)`，不关心下面是静态 key、OAuth token、还是每次现算的 STS 临时凭证

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
