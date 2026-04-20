# Hermes Agent 项目研究记录

> 分析日期：2026-04-16  
> 项目版本：0.9.0  
> 分析目标：Skills 优化机制 & Agent RL 训练机制

---

## 一、项目概述

**Hermes Agent** 是 Nous Research 开发的自我改进 AI Agent 框架，核心特性：

- **闭环学习**：从经验中创建 Skills（程序性记忆），持续自我改进
- **RL 训练集成**：内置 Tinker-Atropos 框架，支持 GRPO 强化学习训练
- **多平台支持**：CLI、Telegram、Discord、Slack、WhatsApp、Signal
- **40+ 工具**：灵活的 Toolset 组合系统
- **Python >= 3.11**

### 项目目录结构

```
hermes-agent/
├── agent/                      # 核心 Agent 实现
│   ├── skill_utils.py          # Skill 元数据工具（frontmatter 解析、平台匹配）
│   ├── skill_commands.py       # Slash 命令处理、Skill 调用
│   ├── prompt_builder.py       # LLM Prompt 构建（注入 Skills、记忆、上下文）
│   ├── trajectory.py           # Trajectory 数据结构与保存
│   ├── context_compressor.py   # 上下文压缩策略
│   └── memory_manager.py       # 记忆生命周期管理
├── tools/                      # 40+ 工具实现
│   ├── skill_manager_tool.py   # Agent 管理 Skill 的创建/编辑/Patch
│   ├── rl_training_tool.py     # RL 训练工具接口（58KB）
│   ├── skills_hub.py           # Skills Hub API 客户端
│   └── skills_guard.py         # Skill 安全扫描
├── skills/                     # 内置 Skills 库（~25 个分类）
├── optional-skills/            # 扩展 Skills 库（需额外安装）
├── tinker-atropos/             # RL 子模块（git submodule）
├── environments/               # RL 训练环境
├── run_agent.py                # 核心 Agent 循环（597KB）
├── cli.py                      # 主 CLI 入口（457KB）
├── rl_cli.py                   # RL 专用 CLI Runner
├── batch_runner.py             # 批量 Trajectory 生成
├── trajectory_compressor.py    # Trajectory 压缩优化
└── toolsets.py                 # Toolset 定义与组合
```

---

## 二、Skills 系统与优化机制

### 2.1 什么是 Skills

Skills 是 Agent 的**程序性记忆**（Procedural Memory）——从实际经验中提炼出的、可复用的任务操作流程，以 Markdown 文件格式存储。

- 内置 Skills：`skills/`（~25 个分类，如 devops、github、mlops、research 等）
- 用户创建 Skills：`~/.hermes/skills/`
- 扩展 Skills：`optional-skills/`（需通过 Skills Hub 安装）

### 2.2 Skill 文件格式

```markdown
---
name: skill-name
description: 简短描述
version: 1.0.0
author: 创建者
metadata:
  hermes:
    tags: [tag1, tag2]
    config:                      # 配置变量声明
      - key: setting.path
        description: 配置说明
        default: "~/default"
        prompt: "安装时的引导提示"
    requires_toolsets: [terminal]   # 依赖的工具集
    fallback_for_tools: [tool-name] # 作为某工具的降级替代
    related_skills: [other-skill]
    platforms: [macos, linux]       # 平台限制
---

# Skill 内容
（基于 Markdown 的操作流程与指令）

# 支持文件目录结构
references/    模板/参考资料
scripts/       辅助脚本
assets/        静态资源
```

### 2.3 Skill 调用流程

用户输入 `/claude-code "Fix the auth bug"` 后的完整调用链：

```
1. 命令解析
   skill_key = resolve_skill_command_key("claude-code")  → "/claude-code"

2. 加载 Skill
   loaded_skill = skill_view(skill_dir)
   frontmatter, body = parse_frontmatter(raw_content)

3. 解析配置变量
   config_vars = extract_skill_config_vars(frontmatter)
   resolved_config = resolve_skill_config_values(config_vars)

4. 构建 Skill 消息
   activation_note = "[SYSTEM: User invoked the 'claude-code' skill...]"
   skill_message = _build_skill_message(
       loaded_skill, skill_dir, activation_note, user_instruction
   )

5. 注入 LLM 上下文
   最终消息 = activation_note + frontmatter + body
              + [config值] + [支持文件] + user_instruction
```

关键文件：
- `agent/skill_commands.py:14KB` — 命令解析与 Skill 调用
- `agent/skill_utils.py:16KB` — frontmatter 解析、平台匹配、配置提取
- `agent/prompt_builder.py:50KB` — Skill 注入 Prompt 的完整逻辑

### 2.4 Skills 优化机制（核心）

#### 机制一：模糊匹配 Patch 系统（Fuzzy Patching）

**文件：** `tools/skill_manager_tool.py`

Agent 在优化 Skill 时使用**模糊匹配**代替精确字符串替换：

- 容忍缩进、空白、换行的变化
- 允许 Agent 在运行时迭代改进 Skill 内容
- 避免因格式差异导致的 Patch 失败
- 测试：`tests/tools/test_skill_improvements.py`

```python
# 核心思路：使用模糊算法定位要替换的代码块
# 而不是精确字符串匹配，使 Skill 编辑更健壮
skill_manager_tool.patch(skill_path, old_content, new_content, fuzzy=True)
```

#### 机制二：Agent 自主创建与编辑 Skills

**文件：** `tools/skill_manager_tool.py`

提供给 Agent 的操作：

| 操作 | 说明 |
|------|------|
| `create` | 从成功经验创建新 Skill |
| `edit` | 整体修改 Skill 内容 |
| `patch` | 局部修改（模糊匹配） |
| `delete` | 删除不再有用的 Skill |
| `write_file` | 向 Skill 添加支持文件 |
| `remove_file` | 从 Skill 移除文件 |

这构成了**闭合学习循环**：Agent 成功完成任务 → 提炼操作流程 → 保存为 Skill → 下次遇到类似任务直接调用。

#### 机制三：安全扫描门控

**文件：** `tools/skills_guard.py:37KB`

所有 Agent 创建或从 Skills Hub 安装的 Skill 都经过安全扫描：

- 检测 Shell 注入模式
- 检测凭证泄露
- 拦截危险代码模式
- 保护用户系统安全

#### 机制四：配置变量系统

Skills 在 frontmatter 中声明配置变量：

- 首次安装时通过 `prompt` 字段引导用户配置
- 配置存储于 `~/.hermes/config.yaml`（`skills.config.*` 路径下）
- 运行时动态注入，无需修改 Skill 本身
- 实现 Skill 的**参数化**与**可移植性**

#### 机制五：平台感知过滤

- Skills 通过 `platforms: [macos, linux]` 声明支持平台
- 加载时自动过滤不兼容平台的 Skill
- 防止 Windows 上加载 macOS 专用 Skill

#### 机制六：Skills Hub 社区生态

**文件：** `tools/skills_hub.py:115KB`、`hermes_cli/skills_hub.py`

- 社区技能市场（agentskills.io）
- Agent 可搜索、下载、安装社区 Skills
- 安装前经过 `skills_guard` 安全审查

---

## 三、Agent RL 训练机制

### 3.1 RL 架构概览

RL 系统基于 **Tinker-Atropos** 框架，使用 **GRPO**（Group Relative Policy Optimization）算法，通过 **LoRA** 适配器训练模型。

三大组件：

| 组件 | 职责 |
|------|------|
| **Atropos** | Trajectory API 服务器，协调环境与训练的交互 |
| **Tinker** | 训练服务，管理模型权重、LoRA 适配器、推理 |
| **Environments** | Python 类，定义任务、评分函数、奖励信号 |

### 3.2 关键 RL 文件

| 文件 | 大小 | 说明 |
|------|------|------|
| `rl_cli.py` | 16KB | RL 专用 CLI Runner（超时设置更长） |
| `tools/rl_training_tool.py` | 58KB | RL 训练工具实现，Agent 可调用 |
| `batch_runner.py` | 56KB | 批量 Trajectory 生成，用于数据收集 |
| `trajectory_compressor.py` | 64KB | Trajectory 优化/压缩，用于训练数据集 |
| `agent/trajectory.py` | — | Trajectory 数据结构与保存工具 |
| `tinker-atropos/` | 子模块 | RL 核心框架（git submodule） |

### 3.3 RL 工具函数（Agent 可调用）

```python
# 环境发现
rl_list_environments()            # 扫描 tinker-atropos/environments/ 目录
rl_select_environment(env_name)   # 加载指定训练环境

# 配置管理
rl_get_current_config()           # 查看所有参数（可配置 + 锁定）
rl_edit_config(field, value)      # 修改可配置参数

# 训练生命周期
rl_start_training()               # 启动训练（同时启动 3 个进程）
rl_check_status()                 # 监控进度 + WandB 指标
rl_stop_training()                # 终止训练
rl_get_results()                  # 获取最终指标 + 模型权重路径
rl_list_runs()                    # 列出所有训练运行记录

# 辅助
rl_test_inference()               # 快速推理测试（通过 OpenRouter）
```

### 3.4 RL 配置系统：锁定 vs 可配置

**锁定参数**（基础设施级，Agent 无法修改）：

```yaml
env:
  tokenizer_name: "Qwen/Qwen3-8B"
  rollout_server_url: "http://localhost:8000"
  max_token_length: 8192
  max_num_workers: 2048
  total_steps: 2500
  steps_per_eval: 25

tinker:
  lora_rank: 32
  learning_rate: 0.00004
  max_token_trainer_length: 9000
```

**可配置参数**（Agent 可通过 `rl_edit_config` 修改）：

- `group_size`：每个样本的补全数量（默认 16，GRPO 核心参数）
- `batch_size`：训练批量大小（默认 128）
- `wandb_name`：WandB 运行名称
- 环境特定参数

### 3.5 训练环境（Environments）

**位置：** `tinker-atropos/tinker_atropos/environments/`

使用 **AST 解析**自动发现继承自 `BaseEnv` 的环境类：

```python
def _discover_environments() -> Dict[str, str]:
    """使用 AST 解析扫描 BaseEnv 子类。"""
    # 解析 environments/ 目录下的每个 .py 文件
    # 查找所有继承自 BaseEnv 的类定义
    # 返回映射：环境名称 → 文件路径
```

内置环境：
- `gsm8k_tinker`：数学推理（GSM8K 数据集）
- `arc_tinker`：常识推理（ARC 数据集）

环境实现需要定义：
1. 数据集加载逻辑
2. Prompt 格式化
3. 评分/奖励函数

### 3.6 RL 完整训练工作流

```
1. 环境发现
   └─ 扫描 tinker-atropos/environments/，列出可用 BaseEnv 子类

2. 环境选择与检查
   └─ 加载环境类，查看：数据集加载、Prompt 格式化、评分逻辑

3. 配置训练参数
   ├─ 查看当前配置（锁定 + 可配置字段）
   └─ 修改可配置参数（batch_size、group_size 等）

4. 推理测试（训练前验证）
   └─ 通过 OpenRouter 快速推理测试，确认环境就绪

5. 启动训练（3 个进程并行）
   ├─ Atropos：Trajectory API 服务
   ├─ Tinker：模型训练服务
   └─ 监控进程
   → 初始化 LoRA 适配器
   → 开始收集 Trajectories，计算 GRPO 优势值

6. 监控进度
   ├─ 定期检查训练状态
   ├─ 监控 WandB 指标
   └─ 周期性推理验证

7. 评估结果
   └─ 获取最终指标 + 模型权重路径
```

### 3.7 Trajectory 系统

Trajectory（轨迹）是 RL 训练的原始数据，记录 Agent 的完整交互过程。

**数据格式**（ShareGPT 兼容）：

```json
{
  "conversations": [
    {"from": "user", "value": "..."},
    {"from": "assistant", "value": "..."},
    {"from": "tool_result", "value": "..."}
  ],
  "timestamp": "2026-04-16T14:30:00",
  "model": "model-name",
  "completed": true
}
```

**数据管道：**

```
Agent 执行任务
    ↓
batch_runner.py           批量生成 Trajectories
    ↓
trajectory_compressor.py  压缩优化（去冗余、截断超长序列）
    ↓
tinker-atropos            GRPO 训练
    ↓
优化后的 LoRA 适配器
```

### 3.8 RL 依赖配置

```toml
# pyproject.toml
[project.optional-dependencies]
rl = [
    "atroposlib",   # Atropos API 框架
    "tinker",       # Tinker 训练服务
    "fastapi",      # API 服务
    "uvicorn",      # ASGI 服务器
    "wandb",        # 实验追踪
]
```

必要环境变量：
- `TINKER_API_KEY`：Tinker 服务 API 密钥
- `WANDB_API_KEY`：Weights & Biases 指标追踪密钥

安装 rl 依赖后，`rl` toolset 自动启用。

---

## 四、两个系统的协同关系

```
用户使用 Agent 完成任务
        ↓
  成功经验被提炼
        ↓
Agent 创建/优化 Skill ──── [技能闭环：即时、明确]
  （skill_manager_tool）
        ↓
Skill 存入 ~/.hermes/skills/
        ↓
下次任务直接调用 Skill
        ↓
收集任务 Trajectories ──── [学习闭环：长期、隐式]
  （batch_runner）
        ↓
压缩优化 Trajectories
  （trajectory_compressor）
        ↓
RL 训练（Tinker-Atropos）
  GRPO + LoRA
        ↓
模型能力提升
        ↓
更高质量的 Skill 创建
```

**双层自我改进机制：**
1. **Skills 层**（即时、明确）：成功经验立即转化为可复用的操作流程，下次直接调用
2. **RL 层**（长期、隐式）：通过 GRPO 训练优化模型本身，提升基础推理与执行能力

---

## 五、核心设计亮点汇总

| 特性 | 实现方式 | 关键文件 |
|------|----------|----------|
| 自主 Skill 创建 | Agent 从成功经验主动提炼并保存技能 | `tools/skill_manager_tool.py` |
| 模糊 Patch | 容忍格式差异的字符串替换，使迭代改进更健壮 | `tools/skill_manager_tool.py` |
| 集成 RL 训练 | 直接调用 Tinker-Atropos 进行 GRPO + LoRA 训练 | `tools/rl_training_tool.py` |
| Trajectory 管道 | 批量生成 → 压缩优化 → RL 训练的完整数据流 | `batch_runner.py`, `trajectory_compressor.py` |
| 安全门控 | 所有 Skill 经安全扫描，防止恶意代码注入 | `tools/skills_guard.py` |
| AST 环境发现 | 用 AST 解析自动发现训练环境，无需手动注册 | `tools/rl_training_tool.py` |
| 配置锁定分离 | 锁定基础设施参数，仅开放安全可配置参数给 Agent | `tools/rl_training_tool.py` |
| Skills Hub 生态 | 社区技能市场，支持搜索/下载/安装 | `tools/skills_hub.py` |
| 平台感知过滤 | Skills 声明平台依赖，加载时自动过滤 | `agent/skill_utils.py` |
| 跨会话记忆 | 用户 Profile、会话搜索、记忆提示的持久化 | `agent/memory_manager.py` |
