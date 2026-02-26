# 模块一：Hooks 基础 — 概念、配置与事件系统

> 对应 PDF 第 1-4 页（Ch 1-3）

---

## 概念讲解

### 1. Claude Code Hooks 是什么

**定义**：Hooks 是你自己定义的 Shell 命令或脚本，它们会在 Claude Code 运行生命周期的**特定时刻自动执行**。简单说就是——可编程触发器（Programmable Triggers），让你在 Claude 的工作流中注入自定义逻辑。

**核心思想**：Claude Code 擅长快速生成代码，但它经常跳过专业开发中的关键步骤（架构规划、安全审查、测试策略、性能规划）。Hooks 就是解决这个"专业性缺口"的机制——给 AI 加上结构、流程和专业护栏。

**类比**：就像 Git Hooks（pre-commit、post-push），但触发点不是 Git 操作，而是 Claude Code 的工具调用和生命周期事件。

---

### 2. 三大核心能力

Hooks 提供三个维度的能力，覆盖了 AI 驱动开发的核心需求：

#### （1）可观测性与日志（Observability & Logging）

| 能力 | 说明 |
|------|------|
| 审计追踪 | 记录 Claude 的每一个操作，形成详细的行动日志 |
| 长任务调试 | 通过逐步日志回顾，定位长时间任务中的问题 |
| Agent 行为分析 | 分析 Agent 的行为模式，优化你的 Prompt |
| 工具使用模式 | 了解 Claude 使用了哪些工具、以什么顺序 |

#### （2）控制与安全（Control & Safety）

| 能力 | 说明 |
|------|------|
| 阻止危险命令 | 拦截 `rm -rf *` 等破坏性操作 |
| 保护敏感文件 | 禁止访问 `.env`、配置文件等 |
| 强制工作流 | 编辑后自动运行 linter |
| 确定性护栏 | 给非确定性的 AI 加上确定性的安全围栏 |

> **关键洞察**：AI 是非确定性的（probabilistic），而 Hooks 是确定性的（deterministic）。两者结合 = 既有 AI 的灵活性，又有规则的可预测性。

#### （3）自动化与个性化（Automation & Personalization）

| 能力 | 说明 |
|------|------|
| 语音通知 | 任务完成时播放声音或语音播报 |
| 个性化体验 | 不同操作配不同提示音 |
| 自动化重复工作 | 减少手动操作 |
| 强制 TDD | 在写实现代码之前自动提醒写测试 |

---

### 3. 配置文件位置与 JSON 结构

**配置文件路径**：`.claude/settings.json`（项目根目录）

> **设计优势**：放在项目目录里意味着 Hooks 配置可以被**版本控制**（git 追踪），也可以在团队中**共享**。

**基本 JSON 结构**：

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "ToolPattern",
        "hooks": [
          {
            "type": "command",
            "command": "your_shell_command_or_script_here"
          }
        ]
      }
    ]
  }
}
```

---

### 4. 配置元素详解

| 元素 | 作用 | 示例 |
|------|------|------|
| **EventName** | 要钩入的生命周期事件 | `"PreToolUse"`, `"Stop"` |
| **matcher** | 匹配工具的模式 | `""` 匹配所有工具；`"Bash"` 或 `"Edit"` 匹配特定工具；支持正则表达式 |
| **command** | 要执行的 Shell 命令 | 可以是单行命令，也可以调用脚本 |

**matcher 匹配规则**：

```json
// 匹配所有工具
"matcher": ""

// 匹配特定工具
"matcher": "Bash"

// 匹配多个工具（正则 OR）
"matcher": "Edit|Write|MultiEdit"

// 匹配 MCP 工具（通配符）
"matcher": "mcp__github__*"
```

---

### 5. Hook 事件系统（5 个核心事件）

每个 Hook 通过 **stdin** 接收一个 JSON 载荷（payload），包含事件上下文。你的脚本可以解析这些数据来做出智能决策。

| 事件 | 触发时机 | 能否阻止操作 | 常用场景 | 关键载荷字段 |
|------|----------|-------------|----------|-------------|
| **PreToolUse** | 工具执行**之前** | ✅ 能（exit 2） | 阻止危险命令、参数验证、记录日志 | `hook_event_name`, `tool_name`, `tool_input` |
| **PostToolUse** | 工具执行**之后** | ❌ 不能 | 记录结果、运行格式化、后续操作 | `hook_event_name`, `tool_name`, `tool_input`, `tool_response` |
| **Notification** | Claude 需要用户输入时 | ❌ 不能 | 自定义提醒、桌面通知 | `hook_event_name`, `message` |
| **Stop** | 主 Agent 完成任务时 | ❌ 不能 | 任务完成提醒、会话日志 | `hook_event_name`, `transcript_path` |
| **SubagentStop** | Sub-Agent 完成任务时 | ✅ 能（exit 2） | 多步骤任务通知 | `hook_event_name`, `stop_hook_active` |

> **重要区分**：
> - **PreToolUse** 是唯一能在工具执行**前**拦截并阻止操作的事件（exit code 2 = 阻止）
> - **PostToolUse** 只能在工具执行**后**做记录或后续操作，无法回滚

**补充事件**（来自速查表 P19）：

| 事件 | 触发时机 | 能否阻止 |
|------|----------|----------|
| **UserPromptSubmit** | 用户提交 Prompt 时 | ❌ |
| **SessionStart** | 会话启动时 | ❌ |
| **PreCompact** | 上下文压缩前 | ❌ |

---

### 6. Exit Code 约定

这是 Hooks 的核心控制机制：

| Exit Code | 含义 | 效果 |
|-----------|------|------|
| **0** | 成功 | stdout 内容在 transcript 模式下显示 |
| **2** | 阻止操作 | stderr 内容发送给 Claude（Claude 能看到你的拒绝理由） |
| **其他** | 非阻止性错误 | stderr 显示给用户（不阻止操作） |

> **设计精妙之处**：exit code 2 阻止操作时，stderr 会反馈给 Claude。这意味着你不仅阻止了操作，还能告诉 Claude **为什么**被阻止，让它调整行为。

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **Hooks = AI 的确定性护栏**：给非确定性的 AI 加上确定性的规则，两者互补
2. **`.claude/settings.json` 是配置入口**：项目级，可版本控制，可团队共享
3. **matcher 支持正则**：`""` 匹配全部，`"Edit|Write"` 匹配多个，`"mcp__*"` 匹配 MCP
4. **PreToolUse 是唯一的"门卫"**：只有它能用 exit code 2 在工具执行前阻止操作
5. **exit code 2 的 stderr 反馈给 Claude**：不仅阻止，还能告诉 Claude 为什么，让它自我修正
6. **6 个以上事件覆盖完整生命周期**：从用户提交 Prompt → 工具执行前后 → 会话结束
