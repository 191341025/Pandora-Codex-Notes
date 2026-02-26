# 模块四：扩展能力 — 可观测性、Slash Commands 与 MCP

> 对应 PDF 第 25-31 页（Ch 11-14）

---

## 概念讲解

### 1. Multi-Agent 可观测性仪表板

**目标**：实时监控多个 Claude Code Agent 的活动。

**架构**：

```
Claude Agents → Hooks → HTTP Server → SQLite → WebSocket → Dashboard
```

**工作原理**：每个 Agent 的 PreToolUse 和 PostToolUse 事件都通过 Hook 发送到一个本地 HTTP Server，数据存入 SQLite，通过 WebSocket 推送到仪表板。

**Hook 配置**：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "python3 .claude/hooks/send_event.py --event PreToolUse --agent $USER"
      }]
    }],
    "PostToolUse": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "python3 .claude/hooks/send_event.py --event PostToolUse --agent $USER --include-output"
      }]
    }]
  }
}
```

**事件发送器脚本核心逻辑**（`.claude/hooks/send_event.py`）：

```python
def send_event(event_type, agent_name, include_output=False):
    data = json.load(sys.stdin)
    event = {
        "timestamp": datetime.utcnow().isoformat(),
        "agent": agent_name,
        "event_type": event_type,
        "tool_name": data.get("tool_name"),
        "tool_input": data.get("tool_input"),
    }
    if include_output:
        event["tool_output"] = data.get("tool_response", {}).get("output", "")[:500]
    try:
        requests.post("http://localhost:8080/events", json=event, timeout=1)
    except:
        pass  # Don't block Claude if observability is down
```

**关键设计**：
- `timeout=1` — 如果可观测性服务挂了，1 秒内放弃，不拖慢 Claude
- `pass` — 可观测性是**非关键功能**，挂了不应该影响工作
- `[:500]` — 截断输出，避免大量数据传输
- `--include-output` 只在 PostToolUse 上用（才有结果可看）

> **适用场景**：团队多人同时使用 Claude Code，需要统一监控各个 Agent 的活动和性能。

---

### 2. Custom Slash Commands

**定义**：Slash Commands 是通过自然语言模板扩展 Claude Code 功能的方式。简单说就是——把你经常复制粘贴的 Prompt 封装成一个可复用的命令。

**核心原则**：**如果你发现自己在重复粘贴同一个 Prompt，那就是时候把它做成 Slash Command 了。**

#### 存放位置

| 位置 | 作用域 | 路径 |
|------|--------|------|
| 项目级 | 仅当前项目 | `.claude/commands/` |
| 全局级 | 所有项目 | `~/.claude/commands/` |

#### 示例 1：PR 审查命令

文件：`.claude/commands/review-pr.md`

```markdown
Please review the changes in the current git branch:
1. First, check what branch we're on with `git branch --show-current`
2. Show the diff with `git diff main...HEAD`
3. Analyze the changes for:
   - Potential bugs or logic errors
   - Security vulnerabilities
   - Performance issues
   - Missing error handling
4. Suggest improvements with specific code examples
5. Check if tests are needed for the changes
Focus on actual issues, not style preferences.
```

**使用**：`/project:review-pr`

#### 示例 2：安全审计命令

文件：`.claude/commands/security-check.md`

```markdown
Perform a comprehensive security audit on the codebase:
1. Check for hardcoded credentials or API keys
2. Review authentication and authorization logic
3. Look for SQL injection vulnerabilities
4. Check for XSS vulnerabilities in frontend code
5. Review file upload handling for security issues
6. Check dependency versions for known vulnerabilities
7. Review error handling to ensure no sensitive data leaks
Provide specific recommendations for each issue found.
```

#### 示例 3：带参数的命令

文件：`.claude/commands/fix-issue.md`

```markdown
Please fix GitHub issue: $ARGUMENTS
Steps:
1. Use `gh issue view $ARGUMENTS` to read the issue
2. Identify the root cause
3. Find relevant files with grep/find
4. Implement the fix
5. Write tests for the fix
6. Verify all tests pass
7. Create a commit with message "Fix #$ARGUMENTS: [description]"
```

**使用**：`/project:fix-issue 123`

> **`$ARGUMENTS` 是 Slash Command 的参数占位符**：用户在命令后面跟的任何内容都会替换 `$ARGUMENTS`。

---

### 3. MCP 工具（Model Context Protocol）

**定义**：MCP（Model Context Protocol）工具是扩展 Claude Code 能力的标准化接口。每个 MCP Server 提供一组专业化工具。

#### 7 个核心 MCP Server

| MCP Server | 能力 | 用途 |
|------------|------|------|
| **Sequential** | 复杂多步骤思考 | 问题分解、推理链 |
| **Context7** | 获取官方库文档 | 从线上抓取最新的库/框架文档 |
| **Magic** | 生成现代 UI 组件 | 按最佳实践生成前端组件 |
| **Playwright** | 浏览器自动化与测试 | E2E 测试、网页操作 |
| **Memory** | 跨会话持久化存储 | 记住上下文，跨会话保留信息 |
| **Filesystem** | 增强的文件操作 | 超出默认文件能力的操作 |
| **GitHub** | GitHub API 直接集成 | Issue、PR、仓库管理 |

#### 安装 MCP Server

```bash
# 安装
claude mcp add sequential
claude mcp add context7
claude mcp add magic
claude mcp add playwright

# 查看已连接的 MCP Server
claude
# 然后使用：/mcp
```

#### MCP 工具命名模式

所有 MCP 工具遵循统一的命名格式：`mcp__<server>__<tool>`

```
mcp__memory__create_entities
mcp__filesystem__read_file
mcp__github__search_repositories
mcp__sequential__think_step_by_step
mcp__context7__fetch_documentation
mcp__magic__generate_component
```

#### MCP 工具的 Hook 集成

**审计 GitHub 操作**：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "mcp__github__*",
      "hooks": [{
        "type": "command",
        "command": "echo '[$(date)] GitHub MCP action: $CLAUDE_TOOL_NAME' >> ~/.claude/mcp-audit.log"
      }]
    }]
  }
}
```

**自动 Stage MCP 文件写入**：

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "mcp__filesystem__write_file",
      "hooks": [{
        "type": "command",
        "command": "git add $CLAUDE_FILE_PATHS && echo 'Auto-staged changes from MCP filesystem write'"
      }]
    }]
  }
}
```

> **Matcher 支持通配符**：`mcp__github__*` 匹配所有 GitHub MCP 工具调用。

---

### 4. 学习与开发工作流

#### 4.1 Multi-AI 工作流：Claude Code + Cursor IDE

**协同策略**：

```
1. Claude Code：负责实现功能或修复问题
2. Cursor AI：在 Cursor 中分析 Claude 生成的代码
   - "Explain what this function does"
   - "List everything in the .claude directory and its purpose"
   - "What design patterns are being used here?"
3. Claude Desktop：深入理解复杂概念
```

> **为什么要这样做**：Claude Code 快速生成代码，但你需要**理解**它生成的代码。用另一个 AI 工具来解释，形成学习闭环。

#### 4.2 个性化学习助手

构建一个基于你个人偏好的 AI 助手，训练在：
- 你偏好的学习风格
- 你的技术栈和领域知识
- 你的编码规范

然后把 Claude Code 的输出喂给它，获得个性化的解释和学习路径。

#### 4.3 学习日志 Hook

自动记录 Claude 的每一次代码修改：

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "echo '## Code Change\nTool: $CLAUDE_TOOL_NAME\nFiles: $CLAUDE_FILE_PATHS\n' >> .claude/learning-log.md"
      }]
    }]
  }
}
```

> **效果**：事后回顾 `learning-log.md`，就能清楚地看到 Claude 做了什么、改了哪些文件。

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **可观测性架构**：Hooks → HTTP → SQLite → WebSocket → Dashboard，适合团队监控
2. **可观测性不能拖慢 Claude**：timeout=1 + pass，挂了就忽略
3. **Slash Command = 封装重复 Prompt**：如果你复制粘贴同一个 Prompt 超过两次，做成命令
4. **`$ARGUMENTS` 实现参数化**：`/project:fix-issue 123` 中的 123 替换 `$ARGUMENTS`
5. **7 个核心 MCP Server**：Sequential、Context7、Magic、Playwright、Memory、Filesystem、GitHub
6. **MCP 命名模式统一**：`mcp__<server>__<tool>`，matcher 支持通配符
7. **Multi-AI 学习闭环**：Claude Code 写代码 → Cursor 解释代码 → 你理解代码
8. **学习日志自动化**：PostToolUse + Edit|Write → 自动记录每次代码修改
