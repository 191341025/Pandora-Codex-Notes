# 模块二：Hooks 实战 — 从基础示例到高级用法

> 对应 PDF 第 5-15 页（Ch 4-6）

---

## 概念讲解

### 1. 示例 1：声音通知（Sound Notifications）

**目标**：Claude 需要你输入时播放提示音。

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "afplay /System/Library/Sounds/Glass.aiff"
          }
        ]
      }
    ]
  }
}
```

> **平台差异**：macOS 用 `afplay`，Linux 用 `aplay`。

**这是最简单的 Hook**——一行命令，零脚本。适合作为你的第一个 Hook 练手。

---

### 2. 示例 2：智能语音播报（Voice Alerts）

**目标**：用文字转语音（TTS）让 Claude 在不同事件时"说话"。

**依赖**：`pip install pyttsx3`

**脚本**（`.claude/hooks/announcer.py`）：

```python
#!/usr/bin/env python3
import sys, json, pyttsx3

def main():
    try:
        input_data = json.load(sys.stdin)
        event_name = input_data.get("hook_event_name", "Unknown event")
        message = ""
        if event_name == "Stop":
            message = "Main task complete. Ready for the next step."
        elif event_name == "SubagentStop":
            message = "Sub-task finished."
        elif event_name == "Notification":
            message = "Your agent needs your input."
        if message:
            engine = pyttsx3.init()
            engine.say(message)
            engine.runAndWait()
    except Exception:
        pass  # Fail silently to avoid interrupting Claude

if __name__ == "__main__":
    main()
```

**配置**：同一个脚本挂在 3 个不同事件上（Stop、SubagentStop、Notification）。

**关键设计**：
- 通过 `hook_event_name` 字段区分不同事件
- `except Exception: pass` — 静默失败，不干扰 Claude 工作
- 同一个脚本可以复用到多个事件

---

### 3. 示例 3：安全护栏 — 阻止 rm -rf

**目标**：阻止 Claude 执行 `rm -rf` 等危险删除命令。

**脚本**（`.claude/hooks/safety_check.py`）：

```python
#!/usr/bin/env python3
import sys, json, re

def is_dangerous_rm(command):
    patterns = [
        r'\brm\s+-[^\s]*r[^\s]*f',
        r'\brm\s+-[^\s]*f[^\s]*r',
    ]
    return any(re.search(p, command) for p in patterns)

def main():
    input_data = json.load(sys.stdin)
    tool_name = input_data.get("tool_name")
    if tool_name == "Bash":
        command = input_data.get("tool_input", {}).get("command", "")
        if is_dangerous_rm(command):
            print("BLOCKED: Dangerous rm command detected and prevented.",
                  file=sys.stderr)
            sys.exit(2)  # Exit code 2 = 阻止操作

if __name__ == "__main__":
    main()
```

**配置**：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 .claude/hooks/safety_check.py"
          }
        ]
      }
    ]
  }
}
```

**关键机制**：
- 挂在 **PreToolUse** 事件 + **Bash** matcher = 只检查 Bash 命令
- `sys.exit(2)` 阻止执行，stderr 消息反馈给 Claude
- 正则匹配 `rm -rf` 和 `rm -fr` 两种写法变体

> **这是 Hooks 最核心的用法**：用确定性的正则规则阻止 AI 的非确定性行为。

---

### 4. 示例 4：TDD 强制执行

**目标**：在 Claude 修改源代码文件前，检查是否有对应的测试文件。如果没有，发出警告。

**核心逻辑**：检测修改的文件是否是源代码文件（`.py`、`.ts`、`.js` 等），如果是，查找是否存在对应的测试文件。

```python
def should_have_test(file_path):
    patterns = [r'\.py', r'\.ts', r'\.js', r'\.java', r'\.go']
    return any(re.search(pattern, file_path) for pattern in patterns)
```

> **注意**：这个 Hook 不阻止操作（exit 0），只是警告。你可以改为 exit 2 来强制阻止无测试的代码修改。

---

### 5. 最佳实践

#### 5.1 Debug First — 先搞清楚数据长什么样

**在写复杂逻辑之前**，先用一个简单的日志 Hook 记录 JSON payload：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "jq '.' >> .claude/hooks/log.jsonl"
      }]
    }]
  }
}
```

> **这是调试 Hooks 的第一步**：不知道 payload 长什么样，就别急着写逻辑。

#### 5.2 项目组织规范

| 规范 | 说明 |
|------|------|
| 配置文件 | `.claude/settings.json` 放在项目仓库中 |
| 脚本目录 | `.claude/hooks/` 统一存放 Hook 脚本 |
| 版本控制 | 让 Hooks 可共享、可追溯 |

#### 5.3 错误处理原则

| 原则 | 做法 |
|------|------|
| 静默失败 | 除非设计为阻止操作，否则 Hook 应该静默失败 |
| exit 0 | 正常结束 |
| exit 2 | 仅用于**有意**阻止操作 |
| 异常捕获 | 所有逻辑包在 try/except 中 |

#### 5.4 依赖管理

推荐用 Astral 的 **uv** 来管理 Python 脚本依赖：

```bash
uv run my_script.py
```

自动处理环境配置，不需要手动 pip install。

---

### 6. 社区 Pro Tips

来自社区实战经验的 4 条建议：

| # | 建议 | 说明 |
|---|------|------|
| 1 | **新建会话而不是清除** | 上下文混乱时别用 clear，开个新会话。用 `claude --resume` 可以恢复之前的会话 |
| 2 | **用 Slash Commands 代替复制粘贴** | 重复使用的 Prompt（如安全检查模板）应该做成 Slash Command |
| 3 | **多工具协同学习** | Claude Code + Cursor IDE 并行使用：Claude 写代码，Cursor 帮你理解代码 |
| 4 | **学习循环** | 用另一个 AI 工具分析 Claude Code 的输出，加深对生成代码的理解 |

> **第 1 条特别重要**：`claude --resume` 是一个很多人不知道的功能。上下文乱了不要 clear，开新会话然后 resume 旧的。

---

### 7. 高级示例：多工具模式匹配

**目标**：根据文件类型，对不同工具执行不同的后处理操作。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "if [[ \"$CLAUDE_FILE_PATHS\" =~ \\.(py)$ ]]; then black $CLAUDE_FILE_PATHS && ruff check --fix $CLAUDE_FILE_PATHS; fi"
          },
          {
            "type": "command",
            "command": "if [[ \"$CLAUDE_FILE_PATHS\" =~ \\.(ts|tsx)$ ]]; then prettier --write $CLAUDE_FILE_PATHS && npx tsc --noEmit $CLAUDE_FILE_PATHS; fi"
          }
        ]
      }
    ]
  }
}
```

**工作原理**：
- `matcher: "Edit|MultiEdit|Write"` — 匹配所有代码编辑工具
- 通过 `$CLAUDE_FILE_PATHS` 环境变量获取被修改的文件路径
- Python 文件 → `black` 格式化 + `ruff` 检查
- TypeScript 文件 → `prettier` 格式化 + `tsc` 类型检查

> **实用价值极高**：每次 Claude 编辑文件后自动格式化和 lint，保证代码质量一致。

---

### 8. 高级示例：会话上下文加载

**目标**：Claude Code 启动时自动加载项目上下文（Git 状态、最近提交、TODO 项）。

**脚本**（`.claude/hooks/session_start.py`）核心逻辑：

```python
def load_context():
    context = []
    # Git status
    git_status = subprocess.check_output(['git', 'status', '--short'], text=True)
    if git_status:
        context.append(f"Current git changes:\n{git_status}")
    # Recent commits
    commits = subprocess.check_output(['git', 'log', '--oneline', '-5'], text=True)
    context.append(f"Recent commits:\n{commits}")
    # TODO items
    todos = subprocess.check_output(
        ['grep', '-r', 'TODO:', '.', '--include=*.py'], text=True)
    if todos:
        context.append(f"TODO items found:\n{todos[:500]}...")
```

**配置事件**：`SessionStart`

> **效果**：每次打开 Claude Code 就自动知道项目当前状态——改了什么、最近做了什么、还有什么 TODO。不用每次手动说"先看看 git status"。

---

### 9. 高级示例：AI 驱动的命令验证

**目标**：用模式匹配分析可能危险的命令，在执行前阻止。

**比示例 3 更完整的危险模式列表**：

```python
DANGEROUS_PATTERNS = [
    (r'rm\s+-[^\s]*[rf]', "Recursive deletion detected"),
    (r'chmod\s+777', "Overly permissive file permissions"),
    (r'curl.*\|\s*sh', "Piping curl to shell - potential security risk"),
    (r'npm\s+install.*--global', "Global npm install detected"),
    (r'docker.*--privileged', "Privileged Docker container"),
]
```

**阻止机制**：

```python
if risks:
    print(json.dumps({
        "error": "Command blocked due to security concerns",
        "risks": risks,
        "command": command,
        "suggestion": "Please use a safer alternative or confirm this is intentional"
    }), file=sys.stderr)
    sys.exit(2)
```

> **比简单的 rm -rf 检查更全面**：覆盖了 chmod 777、curl | sh、全局 npm install、privileged Docker 等多种安全风险。

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **从简单到复杂的学习路径**：声音通知（1 行）→ 语音播报（脚本）→ 安全护栏（正则 + exit 2）→ 多工具匹配（条件分支）
2. **Debug First**：先用 `jq '.' >> log.jsonl` 记录 payload，搞清楚数据结构再写逻辑
3. **静默失败是默认原则**：Hook 出错不应该干扰 Claude 工作，除非你有意要阻止
4. **`claude --resume` 是隐藏技巧**：上下文混乱时开新会话 + resume，而非 clear
5. **多工具模式匹配是最实用的高级 Hook**：编辑后自动 lint/format，省掉手动操作
6. **SessionStart 上下文加载**：让 Claude 每次启动就知道项目状态
7. **5 种危险模式**：rm -rf、chmod 777、curl|sh、global npm、privileged docker
