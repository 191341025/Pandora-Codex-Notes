# 模块三：Hooks 运维 — 故障排除、速查与集成

> 对应 PDF 第 16-24 页（Ch 7-10）

---

## 概念讲解

### 1. 故障排除 4 大常见问题

#### 问题一：Hook 没有执行

**症状**：配置了 Hook 但没有被触发。

**排查清单**：

| 步骤 | 检查内容 | 解决办法 |
|------|----------|----------|
| 1 | 文件权限 | `chmod +x .claude/hooks/*.py` |
| 2 | JSON 语法 | `jq '.' .claude/settings.json` 验证 |
| 3 | 路径问题 | 使用**绝对路径**（官方推荐） |
| 4 | 确认触发 | 加一行日志 `echo 'Hook fired at $(date)' >> /tmp/claude-hook-debug.log` |

> **官方推荐用绝对路径**：相对路径在不同工作目录下可能出问题。

#### 问题二：Settings 修改不生效

**原因**：Settings 在 Claude Code **启动时加载**。

**解决**：
1. 重启 Claude Code
2. 检查 JSON 语法错误
3. 确认编辑的是正确的文件：
   - 项目级：`.claude/settings.json`
   - 用户级：`~/.claude/settings.json`

#### 问题三：Hook 意外阻止操作

**原因**：Hook 脚本以非零 exit code 退出。

**解决**：所有逻辑包在 try/except 中，异常时 exit 0：

```python
try:
    # Your hook logic
    pass
except Exception as e:
    with open('/tmp/hook-errors.log', 'a') as f:
        f.write(f"Error: {e}\n")
    sys.exit(0)  # Exit successfully - don't block Claude
```

#### 问题四：性能问题 — Hook 拖慢 Claude

**原因**：Hooks 是**同步执行**的，重操作会阻塞。

**解决方案**：

```json
// 方案 1：后台执行（非关键 Hook）
{
  "type": "command",
  "command": "your_command &"
}

// 方案 2：设置超时
{
  "type": "command",
  "command": "your_command",
  "timeout": 5
}
```

> **核心原则**：Hook 跑得越快越好。需要长时间运行的操作用 `&` 放到后台。

---

### 2. 速查表（Quick Reference Cheat Sheet）

#### 2.1 Hook 事件总表

| 事件 | 用途 | 能否阻止 | 关键数据 |
|------|------|----------|----------|
| UserPromptSubmit | 修改用户 Prompt | ❌ | `prompt` |
| SessionStart | 加载上下文 | ❌ | `session_id`, `source` |
| PreToolUse | 验证/阻止工具 | ✅ (exit 2) | `tool_name`, `tool_input` |
| PostToolUse | 处理结果 | ❌ | `tool_response` |
| Notification | 自定义提醒 | ❌ | `message` |
| Stop | 任务完成 | ❌ | `transcript_path` |
| SubagentStop | 子任务完成 | ✅ (exit 2) | `stop_hook_active` |
| PreCompact | 上下文压缩前 | ❌ | `trigger` |

#### 2.2 Exit Code 约定

| Exit Code | 含义 | 效果 |
|-----------|------|------|
| 0 | 成功 | stdout 在 transcript 模式显示 |
| 2 | 阻止操作 | stderr 发送给 Claude |
| 其他 | 非阻止错误 | stderr 显示给用户 |

#### 2.3 环境变量

```bash
$CLAUDE_PROJECT_DIR   # 项目根目录的绝对路径
$CLAUDE_FILE_PATHS    # 空格分隔的文件路径列表
$CLAUDE_TOOL_NAME     # 当前使用的工具名
$CLAUDE_NOTIFICATION  # 通知消息内容
```

#### 2.4 Matcher 常用模式

```json
""                     // 匹配所有工具
"Bash"                 // 匹配特定工具
"Edit|Write|MultiEdit" // 匹配多个工具
"mcp__github__*"       // 匹配 MCP 工具
```

#### 2.5 会话管理命令

| 命令 | 作用 |
|------|------|
| `claude --resume` | 恢复之前的会话上下文 |
| `claude -p "prompt" --output-format stream-json` | Headless 模式（自动化用） |
| `claude -p "prompt" --no-interactive` | 非交互模式（CI/CD 用） |

#### 2.6 Agent 命令

| 命令 | 作用 |
|------|------|
| `/agents` | 打开 Agent 管理界面 |
| `/agents list` | 列出所有 Agent |
| `/agents edit [name]` | 编辑 Agent |
| `/agents delete [name]` | 删除 Agent |
| `"Create a [type] specialist"` | 自然语言创建 Agent |
| `"Use the [agent-name] to..."` | 显式调用特定 Agent |

---

### 3. 开发工作流集成

#### 3.1 Git 工作流集成

**Pre-commit 集成**：Claude 编辑文件后自动执行 pre-commit 检查：

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "git add $CLAUDE_FILE_PATHS && ./.git/hooks/pre-commit --files $CLAUDE_FILE_PATHS || true"
      }]
    }]
  }
}
```

#### 3.2 CI/CD 集成（Headless Mode）

在 CI 流水线中无人值守地运行 Claude Code：

```bash
# 在 CI 中修复 lint 错误
claude -p "Fix all linting errors in src/" --output-format stream-json

# Push 前验证
claude -p "Run all tests and fix any failures" --no-interactive
```

> **Headless Mode 的意义**：Claude Code 不只是交互式工具，它可以作为 CI/CD 流水线的一个环节来使用。

#### 3.3 IDE 集成（VSCode）

编辑后自动在 VSCode 中打开被修改的文件：

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit",
      "hooks": [{
        "type": "command",
        "command": "code --goto $CLAUDE_FILE_PATHS:1"
      }]
    }]
  }
}
```

#### 3.4 Docker 开发集成

阻止没有 `--rm` 标志的 Docker 容器运行（防止容器残留）：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "python3 -c \"import json,sys; d=json.load(sys.stdin); cmd=d.get('tool_input',{}).get('command',''); sys.exit(2 if 'docker' in cmd and '--rm' not in cmd else 0)\""
      }]
    }]
  }
}
```

---

### 4. 安全原则与安全 Hook 模板

#### 5 大安全原则

1. **Hooks 以你的凭据运行** — 它们拥有你系统的完整访问权限
2. **启用前审查所有 Hook 代码** — 不要盲信
3. **永远不运行来源不明的 Hooks** — 和运行不明脚本一样危险
4. **对 Hook 脚本使用限制性权限** — 最小权限原则
5. **验证脚本中的所有输入** — 防御性编程

#### 安全 Hook 模板（白名单方式）

```python
#!/usr/bin/env python3
import json, sys, os, re

# 白名单：只允许这些命令模式
ALLOWED_COMMANDS = [
    r'^ls\s', r'^cat\s', r'^grep\s', r'^find\s.*-name',
]

# 黑名单：禁止访问这些路径
FORBIDDEN_PATHS = [
    '.env', '.ssh/', 'credentials', 'secrets', '.git/config',
]

def validate_command(command):
    if not any(re.match(p, command) for p in ALLOWED_COMMANDS):
        return False, "Command not in allowlist"
    for path in FORBIDDEN_PATHS:
        if path in command:
            return False, f"Access to {path} is forbidden"
    return True, ""

def main():
    try:
        data = json.load(sys.stdin)
        if data.get("tool_name") == "Bash":
            command = data.get("tool_input", {}).get("command", "")
            valid, reason = validate_command(command)
            if not valid:
                print(f"Security block: {reason}", file=sys.stderr)
                sys.exit(2)
    except Exception:
        pass  # Fail open - don't block on errors
```

> **白名单 vs 黑名单**：这个模板同时用了两者。白名单（ALLOWED_COMMANDS）控制"能做什么"，黑名单（FORBIDDEN_PATHS）控制"不能碰什么"。双重防护。

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **Hook 不触发？四步排查**：权限 → JSON 语法 → 绝对路径 → 日志确认
2. **Settings 修改需重启 Claude Code** — 不会热加载
3. **Hooks 是同步的**：重操作用 `&` 放后台或设置 `timeout`
4. **8 个事件覆盖完整生命周期**：UserPromptSubmit 到 PreCompact
5. **4 个环境变量是 Hook 脚本的信息来源**：`$CLAUDE_PROJECT_DIR` 等
6. **Headless Mode 让 Claude Code 进入 CI/CD**：`--output-format stream-json` + `--no-interactive`
7. **安全 Hook 模板用白名单+黑名单双重防护**：值得直接拿去用
