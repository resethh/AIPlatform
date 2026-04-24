
## 凭证安全的困境

在生产环境中运行 AI Agent 时，常面临以下安全挑战：

- **想用 GitHub，但不想暴露 PAT** —— 一个泄露的 token 可能摧毁整个仓库
- **Worker 需要调用内部 API，但密钥过于敏感** —— 计费系统、数据库、支付网关的 API key 风险极大
- **不同 Worker 需要不同权限，管理复杂** —— 前端 Worker 不应访问生产数据库，但如何强制执行？

HiClaw 1.0.6 提供了完整解决方案：**Higress AI Gateway + mcporter** 企业级 MCP Server 管理。

---

## 什么是 MCP？为什么重要？

**MCP (Model Context Protocol)** 是一个开放标准，用于将 API 暴露为 AI Agent 可发现和调用的“工具”。可理解为“给 AI Agent 用的 OpenAPI”。

MCP 的核心优势：**工具定义与凭证管理分离**。工具的 schema 说明功能和参数，但不包含 API key。这种分离是企业级安全部署的基础。

---

## 介绍 mcporter：通用 MCP CLI

由 Peter Steinberger（OpenClaw 作者）开发的强大 MCP 工具包。

### 核心能力

- **零配置发现**：自动发现 Cursor、Claude Code、Codex 等环境中配置的 MCP 服务器
- **友好 CLI**：`mcporter call server.tool key=value`
- **类型安全**：生成带完整类型推断的 TypeScript 客户端
- **一键 CLI 生成**：将任意 MCP 服务器转换为独立 CLI 工具

```bash
# 列出所有已配置的 MCP 服务器
mcporter list

# 查看某个服务器的工具和完整参数 schema
mcporter list github --schema

# 调用工具
mcporter call github.search_repositories query="hiclaw" limit=5
```

在 HiClaw 1.0.6 中，Manager 和 Worker 均通过 mcporter 与 MCP 服务器交互，但通过 Higress AI Gateway 实现了关键的安全增强。

---

## MCP 与 SKILLS：互补而非替代

### HiClaw 的开放技能生态

- 对接 [skills.sh](http://skills.sh) 开放 SKILLS 市场
- 支持 Nacos 企业自建 SKILLS 市场
- SKILLS 是面向场景、可迭代的能力包装

### MCP 的定位

- **明确的约束和规范**
- **权限治理体系**（MCP Server 粒度，企业版支持工具粒度）
- **批量转换能力**：基于 Higress 网关无痛将存量 API 批量转换为 MCP 工具

### mcporter 的桥接作用

```
MCP 工具（原子能力）
    ↓ mcporter 编排
SKILL（场景化能力包）
    ↓ 实战使用
SKILL 迭代优化
```

### 最佳实践

- **SKILL** 负责场景演进
- **MCP** 负责细粒度权限管控

两者结合实现 **1+1 > 2** 的效果。

---

## 架构：一切如何运作

```text
你（人类） → Manager Claw → Higress AI Gateway → Worker Claw
```

### 详细流程

1. **你** 向 Manager 描述 API 并给出凭证
2. **Manager** 生成 MCP Server YAML 配置，部署到 Higress，用 mcporter 验证
3. **Higress AI Gateway** 安全存储真实凭证，签发临时 consumer token 给 Worker
4. **Worker** 拉取配置，发现并测试工具，生成 SKILL.md，后续任务中使用

### 核心安全原则

**Worker 永远看不到真实凭证**。即使 Worker 被完全攻破，攻击者也只能获得一个可随时吊销、不包含任何复用凭证的 consumer token。

---

## 端到端示例：添加自定义 API

### 步骤 1：向 Manager 描述 API

```text
你：我想添加计费 API 作为 MCP 工具。
    端点：GET https://billing.internal.company.com/api/v1/customers/{customer_id}
    认证：Authorization header 里的 Bearer token
    这是 token：Bearer eyJhbGciOiJSUzI1NiIs...
```

### 步骤 2：Manager 完成繁重工作

- 生成 YAML 配置
- 部署到 Higress MCP Gateway
- 用 mcporter 验证
- 通知相关 Worker

### 步骤 3：Worker 自动配置

```bash
# 从 MinIO 拉取更新的配置
hiclaw-sync

# 发现新工具
mcporter list billing --schema

# 测试工具
mcporter call billing.get_customer customer_id=CUST-12345

# 生成 SKILL.md
```

### 步骤 4：Worker 在任务中使用工具

Worker 调用工具并返回结果，全程未接触真实 API token。

---

## 从 Swagger/OpenAPI 到 MCP 工具

直接提供 Swagger 文档 URL：

```text
你：这是我们的产品目录 API Swagger 文档：
    https://docs.internal.company.com/swagger.json
    通过 X-API-Key header 认证。Key：prod_cat_xxx

Manager：我会将 Swagger 文档转换为 MCP 工具...
         发现 12 个端点，创建 12 个 MCP 工具...
         部署到 Higress 为 `product-catalog` MCP 服务器。
```

---

## 从 curl 到 MCP 工具

直接粘贴 curl 命令：

```text
你：添加这个 API 调用作为工具：
    curl -X GET "https://api.shipping.com/v1/track?tracking_id=ABC123" \
         -H "X-API-Key: ship_xxx"

Manager：创建 MCP 服务器 `shipping`，包含工具 `track_package`...
```

---

## Worker 生成的 Skills：自我完善的文档

当 Worker 首次遇到新 MCP 服务器时，它会：

1. 发现所有工具
2. 测试代表性工具
3. 生成 `SKILL.md`，包含自然语言说明、示例命令、参数说明、注意事项

随着使用增加，Worker 可基于实战经验不断改进 SKILL。

---

## 安全模型：深度防御

### Worker 可以做的

- 调用任何已授权的 MCP 服务器
- 通过 Higress AI Gateway 使用工具
- 生成和改进 SKILL 文档
- 在授权范围内自主工作

### Worker 不能做的

- 看到真实的 API key、token 或凭证
- 调用未授权的 MCP 服务器
- 从网关提取凭证
- 与其他 Worker 共享凭证

### 如果 Worker 被攻破

- 攻击者获得一个仅在 HiClaw 内有效的 consumer token
- Manager 立即吊销该 token，无需轮换凭证
- 真实 API key 保持安全
- 创建新 Worker，几分钟内恢复工作

---

## 其他改进

### Slash Commands（跨场景可用）

- `/reset` — 修复卡住或配置错误的 Claw
- `/stop` — 中断长时间运行的任务
- `/model` — 快速切换当前会话模型

### 优化的文件同步

统一“写入者推送并通知，接收者按需拉取”原则，5 分钟定时拉取仅作兜底。

---

## 快速开始

### 升级现有 HiClaw

```bash
bash <(curl -sSL https://higress.ai/hiclaw/install.sh)
```

### 新用户安装

```bash
# macOS / Linux
bash <(curl -sSL https://higress.ai/hiclaw/install.sh)

# Windows (PowerShell 7+)
Set-ExecutionPolicy Bypass -Scope Process -Force; $wc=New-Object Net.WebClient; iex $wc.DownloadString('https://higress.ai/hiclaw/install.ps1')
```

安装完成后，访问 `http://127.0.0.1:18088` 打开 Element Web，开始添加 MCP 工具。