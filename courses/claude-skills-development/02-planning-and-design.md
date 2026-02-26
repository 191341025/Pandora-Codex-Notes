# 模块二：Planning and Design

> 对应 PDF 第 7-13 页

---

## 概念讲解

### 1. 从用例出发——先想清楚再动手

**核心思想**：在写任何代码之前，先明确 2-3 个你的 Skill 应该支持的**具体用例**。

**好的用例定义长什么样**：

```
Use Case: Project Sprint Planning
Trigger: User says "help me plan this sprint" or "create sprint tasks"
Steps:
  1. Fetch current project status from Linear (via MCP)
  2. Analyze team velocity and capacity
  3. Suggest task prioritization
  4. Create tasks in Linear with proper labels and estimates
Result: Fully planned sprint with tasks created
```

**定义用例时要问自己的四个问题**：

| 问题 | 为什么重要 |
|------|-----------|
| 用户想**完成什么**？ | 明确目标——不是"用工具"，而是"达成结果" |
| 需要**哪些多步骤工作流**？ | 决定 Skill 的复杂度和步骤编排 |
| 需要**哪些工具**（内置 or MCP）？ | 确定依赖关系 |
| 应该嵌入**哪些领域知识或最佳实践**？ | 这才是 Skill 的核心价值——不只是调用工具，还要知道怎么调用 |

---

### 2. 三类 Skill 用例分类

Anthropic 观察到三类最常见的 Skill 用途：

#### Category 1：文档与资产创作（Document & Asset Creation）

**用途**：创建一致的、高质量的输出——文档、演示文稿、应用、设计、代码等。

**真实示例**：`frontend-design` Skill（另外还有 docx、pptx、xlsx、ppt 等 Skill）

> "Create distinctive, production-grade frontend interfaces with high design quality."

**关键技巧**：
- 嵌入**风格指南和品牌标准**
- 使用**模板结构**保持输出一致性
- 在交付前运行**质量检查清单**
- **不需要外部工具**——只用 Claude 的内置能力

#### Category 2：工作流自动化（Workflow Automation）

**用途**：多步骤流程，受益于一致的方法论，可能需要跨多个 MCP server 协调。

**真实示例**：`skill-creator` Skill

> "Interactive guide for creating new skills. Walks the user through use case definition, frontmatter generation, instruction writing, and validation."

**关键技巧**：
- 每步之间设**验证门**（validation gates）
- 提供**通用结构的模板**
- 内建**审查和改进建议**
- 支持**迭代精炼循环**

#### Category 3：MCP 增强（MCP Enhancement）

**用途**：在 MCP server 提供的工具访问之上，叠加工作流指导。

**真实示例**：`sentry-code-review` Skill（来自 Sentry）

> "Automatically analyzes and fixes detected bugs in GitHub Pull Requests using Sentry's error monitoring data via their MCP server."

**关键技巧**：
- 按顺序编排多个 MCP 调用
- **嵌入领域专业知识**
- 提供用户原本需要手动指定的上下文
- **错误处理**覆盖常见 MCP 问题

#### 三类对比

| 维度 | Category 1 | Category 2 | Category 3 |
|------|-----------|-----------|-----------|
| 核心目的 | 高质量内容输出 | 多步流程标准化 | 增强已有 MCP |
| 是否需要 MCP | 不需要 | 可能需要 | 必须 |
| 主要价值 | 一致性 + 品质 | 流程自动化 | 工具 + 知识结合 |
| 典型示例 | frontend-design | skill-creator | sentry-code-review |

---

### 3. 成功标准定义（Define Success Criteria）

**为什么需要**：你得知道 Skill 什么时候算"好使了"。

> **重要提示**：这些指标是**大致基准**（rough benchmarks），不是精确阈值。Anthropic 自己也承认有"vibes-based assessment"的成分，正在开发更严格的测量工具。

#### 3.1 定量指标

| 指标 | 目标 | 怎么测 |
|------|------|--------|
| Skill 触发率 | 90% 的相关查询能触发 | 跑 10-20 个测试查询，统计自动触发 vs 手动激活的比例 |
| 工作流完成效率 | 在 X 次工具调用内完成 | 对比有/无 Skill 的同一任务，统计 tool calls 和总 token |
| API 调用失败率 | 每个工作流 0 次失败 | 监控 MCP server 日志，追踪重试率和错误码 |

#### 3.2 定性指标

| 指标 | 怎么评估 |
|------|----------|
| 用户不需要提示 Claude 下一步 | 测试中记录你需要重新引导或澄清的次数；向 Beta 用户收集反馈 |
| 工作流无需用户纠正即可完成 | 同一请求跑 3-5 次，比较输出的结构一致性和质量 |
| 跨会话结果一致 | 让新用户尝试——首次尝试能否在最少指导下完成任务？ |

---

### 4. 技术要求（Technical Requirements）

#### 4.1 文件结构

```
your-skill-name/
├── SKILL.md          # 必须——主 Skill 文件
├── scripts/          # 可选——可执行代码
│   ├── process_data.py
│   └── validate.sh
├── references/       # 可选——参考文档
│   ├── api-guide.md
│   └── examples/
└── assets/           # 可选——模板等
    └── report-template.md
```

#### 4.2 关键命名规则（Critical Rules）

| 规则 | 正确 ✅ | 错误 ❌ | 原因 |
|------|---------|---------|------|
| SKILL.md 文件名 | `SKILL.md` | `SKILL.MD`、`skill.md` | 大小写敏感，必须精确匹配 |
| Skill 文件夹名 | `notion-project-setup` | `Notion Project Setup`、`notion_project_setup`、`NotionProjectSetup` | 必须 kebab-case |
| 不放 README.md | 无 README | 在 Skill 文件夹内放 README.md | 所有文档写在 SKILL.md 或 references/ 里 |

> **注意**：发布到 GitHub 时，你仍然需要一个**仓库级别的 README**（给人类看的），但它放在 Skill 文件夹**外面**，不要放在 Skill 文件夹里面。

#### 4.3 YAML Frontmatter——最重要的部分

Frontmatter 是 Claude 决定**是否加载你的 Skill** 的依据。写好它至关重要。

**最小必需格式**：

```yaml
---
name: your-skill-name
description: What it does. Use when user asks to [specific phrases].
---
```

就这么少。这就够启动了。

**字段详解**：

| 字段 | 必须？ | 规则 |
|------|--------|------|
| `name` | **是** | kebab-case，无空格无大写，应与文件夹名匹配 |
| `description` | **是** | 必须同时包含**做什么** + **什么时候用**（触发条件）；≤1024 字符；不能有 XML 标签；包含具体的用户触发短语；如果涉及文件类型也要提及 |
| `license` | 否 | 开源时使用，如 MIT、Apache-2.0 |
| `compatibility` | 否 | 1-500 字符，说明环境要求（产品平台、系统包、网络需求等） |
| `metadata` | 否 | 自定义键值对；建议包含 author、version、mcp-server |

**安全限制**：
- **禁止** XML 尖括号（`<` `>`）——因为 frontmatter 会出现在 Claude 的 system prompt 中，恶意内容可能注入指令
- **禁止**在 name 中使用 "claude" 或 "anthropic"——这些是保留词

---

### 5. 写出高效的 Skill

#### 5.1 description 字段——第一级渐进式披露的关键

根据 Anthropic 工程博客：

> "This metadata...provides just enough information for Claude to know when each skill should be used without loading all of it into context."

**结构公式**：`[做什么] + [什么时候用] + [核心能力]`

**好的 description 示例**：

```yaml
# 好——具体且可操作
description: Analyzes Figma design files and generates developer handoff
  documentation. Use when user uploads .fig files, asks for "design specs",
  "component documentation", or "design-to-code handoff".

# 好——包含触发短语
description: Manages Linear project workflows including sprint planning,
  task creation, and status tracking. Use when user mentions "sprint",
  "Linear tasks", "project planning", or asks to "create tickets".

# 好——明确价值主张
description: End-to-end customer onboarding workflow for PayFlow. Handles
  account creation, payment setup, and subscription management. Use when
  user says "onboard new customer", "set up subscription", or
  "create PayFlow account".
```

**坏的 description 示例**：

```yaml
# 太模糊
description: Helps with projects.

# 缺少触发条件
description: Creates sophisticated multi-page documentation systems.

# 太技术化，没有用户触发
description: Implements the Project entity model with hierarchical relationships.
```

> **坑**：description 字段常见的错误就是——**只说了"做什么"但忘了说"什么时候用"**。没有触发条件，Claude 就不知道什么时候该加载你的 Skill。

#### 5.2 主指令编写——Frontmatter 之后的部分

**推荐结构模板**：

```markdown
---
name: your-skill
description: [....]
---

# Your Skill Name

## Instructions

### Step 1: [First Major Step]
Clear explanation of what happens.

```bash
python scripts/fetch_data.py --project-id PROJECT_ID
Expected output: [describe what success looks like]
```

(Add more steps as needed)

## Examples

### Example 1: [common scenario]
User says: "Set up a new marketing campaign"
Actions:
1. Fetch existing campaigns via MCP
2. Create new campaign with provided parameters
Result: Campaign created with confirmation link

(Add more examples as needed)

## Troubleshooting

### Error: [Common error message]
Cause: [Why it happens]
Solution: [How to fix]

(Add more error cases as needed)
```

#### 5.3 指令编写最佳实践

**要具体可操作**：

```markdown
# ✅ 好
Run `python scripts/validate.py --input {filename}` to check data format.
If validation fails, common issues include:
- Missing required fields (add them to the CSV)
- Invalid date formats (use YYYY-MM-DD)

# ❌ 坏
Validate the data before proceeding.
```

**清晰引用捆绑资源**：

```markdown
Before writing queries, consult `references/api-patterns.md` for:
- Rate limiting guidance
- Pagination patterns
- Error codes and handling
```

**善用渐进式披露**：

把 SKILL.md 聚焦在核心指令上，详细文档移到 `references/` 目录并链接过去。

**包含错误处理**：

```markdown
## Common Issues

### MCP Connection Failed
If you see "Connection refused":
1. Verify MCP server is running: Check Settings > Extensions
2. Confirm API key is valid
3. Try reconnecting: Settings > Extensions > [Your Service] > Reconnect
```

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **先定用例再写代码**：明确 2-3 个具体用例，问"用户要什么" + "需要几步" + "用什么工具" + "嵌入什么知识"
2. **三类用例**：文档创作（不需 MCP）、工作流自动化（可能需 MCP）、MCP 增强（必须 MCP）
3. **成功标准是大致基准**：90% 触发率、X 次 tool calls 内完成、0 失败 API 调用——但承认有"凭感觉"的成分
4. **SKILL.md 大小写敏感**：必须精确拼写，文件夹用 kebab-case，不放 README.md
5. **description 是最关键的字段**：公式 = 做什么 + 什么时候用 + 核心能力；缺触发条件 = Skill 废了
6. **安全限制**：frontmatter 中禁用 XML 尖括号和 "claude"/"anthropic" 保留词
7. **指令要具体可操作**：说清楚跑什么命令、预期输出是什么、出错了怎么办
