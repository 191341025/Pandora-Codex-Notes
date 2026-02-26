# 模块五：Sub-Agent 系统 — 创建、实践与链式编排

> 对应 PDF 第 32-52 页（Ch 15）

---

## 概念讲解

### 1. Sub-Agent 概念与核心洞察

**定义**：Sub-Agent 是 Claude Code 中的**专业化 AI 助手**，独立运行在自己的上下文中，拥有聚焦的能力。你可以把它们想象成你能随时召唤的"专家顾问"。

**核心洞察（Key Insight）**：

> Agent 本质上是**"预配置的小 Prompt"**，存在于你的项目中。Claude 会在识别到匹配的专业领域时自动委派任务给它们——你实际上是在构建一个定制的 AI 专家团队。

**范式转变**：从"告诉 AI 怎么做"变为"描述你想要什么"。你的自定义 AI 团队自己搞定其余的事情。

---

### 2. 创建 Sub-Agent

#### 方式一：命令行界面（7 步）

```
1. 输入 /agents 打开 Agent 管理界面
2. 选择 "Create New Agent"
3. 选择作用域：
   - 项目级：.claude/agents/ → 可通过 Git 共享
   - 个人级：~/.claude/agents/ → 所有项目通用
4. 描述你想要什么（如 "An agent that rolls dice"）
5. 选择工具/权限
6. 选择颜色：帮助在终端中识别哪个 Agent 在运行
7. 保存并使用
```

#### 方式二：自然语言创建

直接用自然语言告诉 Claude：
- "Create a debugging specialist agent"
- "I need an agent that only reviews Python code"
- "Set up a data analysis assistant"
- "Make a security auditor agent"

> **零配置**：不需要写 YAML 或 JSON，用自然语言就能创建。

---

### 3. Main Claude vs Agent 对比

| 维度 | Main Claude Code | Agent/Sub-Agent |
|------|-----------------|-----------------|
| **上下文** | 完整对话历史 | 全新开始，无先前上下文 |
| **定位** | 通用型助手 | 专业化单任务聚焦 |
| **记忆** | 跨消息保持状态 | 调用间无记忆 |
| **最适合** | 迭代开发、需要上下文的工作 | 隔离任务、专业领域 |
| **工具权限** | 按配置完全访问 | 按用途限制 |

> **选择策略**：需要延续上下文 → 用 Main Claude；需要全新视角 → 用 Agent。

---

### 4. 四大关键特性

| 特性 | 说明 |
|------|------|
| **上下文隔离** | 每个 Agent 在独立上下文中运行，不会污染主对话 |
| **专业化** | 可以用详细指令针对特定领域微调，成功率更高 |
| **复用性** | 创建一次，跨项目使用，可与团队共享 |
| **灵活权限** | 每个 Agent 可以有不同的工具访问级别 |

---

### 5. Sub-Agent 最佳实践

#### 5.1 System Prompt 是关键

**与普通使用的区别**：你写的是 **System Prompt**（系统提示词），不是 User Prompt。这给你更大的行为控制力。

**示例**：

```markdown
# Code Review Agent System Prompt
You are a specialized code review agent.
Your primary agent will send you code changes to review.

Your responsibilities:
1. Check for bugs and logic errors
2. Identify security vulnerabilities
3. Suggest performance improvements
4. Ensure code follows project conventions

Always respond with structured feedback that the primary agent can act upon.
```

#### 5.2 Sub-Agent 是和 Primary Agent 对话，不是和你

**重要**：Agent 的输出是回复给你的主 Claude，而不是直接回复给你。写 Prompt 时要据此调整。

#### 5.3 指令要极度明确

```
❌ 模糊："Review this code"
✅ 明确："Review this Python code for SQL injection vulnerabilities,
         checking all database queries for parameterization"
```

#### 5.4 善用 Meta-Agent

Meta-Agent 是用来**创建其他 Agent 的 Agent**。用它来快速引导（bootstrap）你的多 Agent 系统。

#### 5.5 链式编排

```
Primary Agent → Code Generator → Code Review → Test Writer
```

---

### 6. Agentic Coding 的 "Big 3"

当你扩展多 Agent 系统时，三个要素变得至关重要：

| 要素 | 说明 | 决定什么 |
|------|------|----------|
| **Context** | 每个 Agent 能访问什么信息 | Agent 的知识边界 |
| **Model** | 哪个 AI 模型驱动每个 Agent | Agent 的能力上限 |
| **Prompt** | 每个 Agent 如何被指导 | Agent 的行为模式 |

> **核心论点**：这三个要素的流动和管理决定了你的多 Agent 系统的成败。

---

### 7. Hooks + Sub-Agents 高级集成

#### 7.1 自动创建 Agent

**场景**：Claude 检测到你在使用新框架时，自动创建对应的专业 Agent。

```python
patterns = {
    "react": ["useState", "useEffect", "jsx"],
    "django": ["models.Model", "views", "urls.py"],
    "fastapi": ["FastAPI", "@app.", "BaseModel"],
}

for framework, keywords in patterns.items():
    agent_path = f".claude/agents/{framework}-specialist.md"
    if not os.path.exists(agent_path) and any(kw in str(tool_input) for kw in keywords):
        create_specialist_agent(framework, agent_path)
        print(f"🤖 Created {framework} specialist agent!")
```

> **效果**：第一次在项目中写 React 代码时，自动创建一个 react-specialist Agent。

#### 7.2 主动触发（Proactive Triggering）

配置 Agent 在满足条件时自动执行：

```markdown
# Auto-Summary Sub-Agent
When called, check if:
- More than 10 files have been modified
- More than 500 lines of code changed
- A major refactoring occurred
- Session duration exceeds 30 minutes

If any condition is true, automatically generate:
1. Executive summary of changes
2. Technical details
3. Next steps
4. Identified technical debt
```

配合 Stop Hook 使用：

```json
{
  "hooks": {
    "Stop": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "if [ $(git diff --stat | tail -1 | awk '{print $4}') -gt 500 ]; then echo 'Major changes detected. Consider running /agent:summarizer'; fi"
      }]
    }]
  }
}
```

#### 7.3 信息密集关键词

**环境变量**（在 Hook 脚本中可用）：

| 变量 | 说明 |
|------|------|
| `$CLAUDE_PROJECT_DIR` | 项目根目录 |
| `$CLAUDE_FILE_PATHS` | 当前相关文件 |
| `$CLAUDE_TOOL_NAME` | 正在使用的工具 |
| `$CLAUDE_SESSION_ID` | 当前会话 ID |

**Sub-Agent Prompt 中的模板变量**：

| 变量 | 说明 |
|------|------|
| `{{current_files}}` | 当前文件 |
| `{{recent_changes}}` | 最近修改 |
| `{{project_context}}` | 项目上下文 |

---

### 8. 五个实战 Agent 示例

#### 8.1 Strict Code Reviewer

```yaml
name: strict-reviewer
description: |
  Enforces coding standards and security practices.
  Call after implementing features or before PRs.
  Triggers: "review this code", "check for issues", "security review"
tools:
  - read_file
  - search_files
```

**输出格式**：
- 🔴 Critical：安全/破坏性问题
- 🟡 Warning：性能/质量问题
- 🟢 Good：良好的模式

#### 8.2 Performance Optimizer

```yaml
name: perf-optimizer
description: |
  Identifies and fixes performance bottlenecks.
  Call when: users report slowness, before scaling up, after adding new features
tools:
  - read_file
  - write_file
  - run_command  # For profiling tools
```

**六大优化方向**：
1. 先测量再优化（no premature optimization）
2. 算法改进优先于微优化
3. 考虑缓存策略
4. 分析数据库查询
5. 检查 N+1 问题
6. 审查内存使用

#### 8.3 Test Generator

```yaml
name: test-gen
description: |
  Creates comprehensive test suites. Call after implementing new features.
  Targets 90%+ coverage including happy path, edge cases, error scenarios, integration tests
tools:
  - read_file
  - write_file
  - search_files
```

**测试命名规范**：`test_[function]_[scenario]_[expected_result]`

#### 8.4 Dice Roller（非代码示例）

```yaml
name: dice-roller
description: |
  Rolls dice for games or random decisions.
  Triggers: "roll a d6", "roll some dice", "roll 2d20"
tools: []  # No tools needed!
```

> **重要示例**：Agent 不一定需要工具权限。`tools: []` 也是合法的。

#### 8.5 Debugging Specialist

```yaml
name: debugger
description: |
  Systematic debugging using root cause analysis.
  Uses Five Whys and systematic elimination.
tools:
  - read_file
  - search_files
  - run_command  # For running tests/logs
```

---

### 9. Agent Frontmatter 与 Description 最佳实践

**Description 是最关键的字段**：Claude 通过 Description 来自动发现和决定何时使用哪个 Agent。

**Description 编写规则**：

| 规则 | 说明 |
|------|------|
| **具体化** | ❌ "reads files" → ✅ 详细列出触发条件和能力 |
| **加触发短语** | 包含用户可能说的话，如 "Use when user says 'roll a d6'" |
| **指定输出格式** | 告诉 Claude 这个 Agent 返回什么格式的结果 |
| **列出显式条件** | 什么时候用 AND 什么时候不用 |
| **唯一目的** | 每个 Agent 有一个清晰、独特的角色 |

---

### 10. 工具权限策略表（最小权限原则）

| Agent 类型 | 推荐工具 | 原因 |
|------------|----------|------|
| **Reviewers** | read_file, search_files | 只读分析 |
| **Writers** | read_file, write_file | 创建/修改代码 |
| **Testers** | 所有文件工具 + run_command | 需要执行测试 |
| **Analyzers** | read_file only | 最小访问 |
| **Builders** | 所有工具 | 完整开发权限 |

> **安全原则**：从最小权限开始，按需添加。Review Agent 不能修改文件 = 更安全。

---

### 11. Agent 链式模式

#### 顺序链（Sequential Chain）

```
debugger → finds issue → test-writer → writes failing test → fixer → implements solution → reviewer → validates fix
```

#### 并行分析（Parallel Analysis）

```
              ┌→ security-auditor  ─┐
code-change   ├→ performance-opt   ├→ combined report
              └→ test-coverage     ─┘
```

#### 层级委托（Hierarchical Delegation）

```
architect → creates design
├→ frontend-dev → implements UI
├→ backend-dev → implements API
└→ test-writer → creates integration tests
```

**自动链式编排**：你不需要手动指定链式顺序。用自然语言描述就行：

```
"Roll some dice and summarize and finally save the output to a file"
```

Claude 自动：dice-roller → summarizer → file-writer

---

### 12. "预测性问题是特性"

Agent 的匹配是**概率性的（probabilistic）**，不是确定性的。这可能让人觉得不可控，但实际上是它的**超能力**。

**拥抱它**：
- 不需要记住精确的 Agent 名字
- 用自然语言描述你想要什么
- "just works" 式的交互

**控制它**：
- Description 写得越具体，匹配越精确
- 加反面示例："Do NOT use for..."
- 使用明确的触发短语
- 用各种 Prompt 测试 Agent 激活情况

---

### 13. Hooks vs Sub-Agents vs Main Claude 决策矩阵

| 场景 | 使用 Main Claude | 使用 Agent | 使用 Hook |
|------|-----------------|------------|-----------|
| 需要对话上下文 | ✅ | | |
| 迭代开发同一段代码 | ✅ | | |
| 需要全新视角 | | ✅ | |
| 需要专业领域知识 | | ✅ | |
| 需要确定性自动化 | | | ✅ |
| 强制策略/标准 | | | ✅ |
| 速度关键 | | | ✅ |
| 需要 AI 推理 | | ✅ | |
| 集成外部工具 | | | ✅ |
| 可复用组件 | | ✅ | |

**三者组合示例**：

```
1. Main Claude: "Plan the user authentication feature"
2. Hook: Auto-creates architecture document template
3. Agent: architect reviews and completes design
4. Main Claude: Implements based on architecture
5. Hook: TDD reminder before coding
6. Agent: test-writer creates test suite
7. Main Claude: Implements to pass tests
8. Hook: Auto-formats and lints code
9. Agent: security-auditor reviews implementation
10. Hook: Generates session summary
```

---

### 14. 重要警告

| 警告 | 说明 |
|------|------|
| **YOLO Mode 极度谨慎** | 让 Agent 不需确认就执行——只在安全环境中使用 |
| **保持 Agent 分离** | Agent 多了之后保持清晰的边界，避免混乱 |
| **使用隔离工作流** | 不同 Agent 用不同目录、配置、日志流、测试环境 |
| **链不要太长** | 3-5 个 Agent 最多，太长难以管理和调试 |
| **不要完全卸载认知工作** | 你仍然需要理解代码、做架构决策、审查输出 |

**性能考量**：

| 指标 | 说明 |
|------|------|
| **延迟** | Agent 每次从零开始，有 1-2 秒额外开销 |
| **Token 效率** | 干净的上下文对聚焦任务可能更高效 |
| **上下文丢失** | 无跨调用记忆——提前规划 |
| **并行执行** | 多个 Agent 可同时工作以加速 |

---

### 15. 超越代码：通用自动化

Agent 不只是编程工具。你可以创建用于：
- **文档大纲**：基于你的模板
- **会议纪要**：转录 → 行动项
- **研究组织**：分类和标签笔记
- **内容创作**：博客、邮件、报告
- **数据分析**：非技术性的数据洞察

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **Agent = 预配置的小 Prompt**：本质上就是存储在项目中的角色提示词
2. **写 System Prompt 而非 User Prompt**：这给你更强的行为控制力
3. **Agent 是和主 Claude 对话**：输出回给主 Agent，不是直接给你
4. **"Big 3"**：Context + Model + Prompt 决定多 Agent 系统的成败
5. **Description 是自动发现的关键**：写得越具体，自动匹配越准确
6. **最小权限原则**：Reviewer 只需 read_file，Tester 需要 run_command
7. **链长度 3-5 个 Agent 为宜**：太长难以调试和管理
8. **概率性匹配是特性而非 Bug**：自然语言 "just works"
9. **三者互补**：Hook 管自动化，Agent 管智能，Main Claude 管上下文
10. **不要完全卸载认知**：Agent 增强你的能力，不替代你的判断
