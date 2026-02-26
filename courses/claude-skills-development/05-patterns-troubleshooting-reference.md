# 模块五：Patterns, Troubleshooting & Reference

> 对应 PDF 第 21-33 页

---

## 概念讲解

### 1. 方法论选择——Problem-first vs Tool-first

在开始构建 Skill 之前，PDF 用了一个 Home Depot（美国家居建材超市）的类比来区分两种思路：

| 思路 | 类比 | 用户说 | Skill 做什么 |
|------|------|--------|-------------|
| **Problem-first** | 带着问题进店："我需要修厨房柜子"，店员指引你找到合适的工具 | "I need to set up a project workspace" | 编排正确的 MCP 调用序列；用户描述结果，Skill 处理工具 |
| **Tool-first** | 拿起一把电钻，问怎么用它干活 | "I have Notion MCP connected" | 教 Claude 最优工作流和最佳实践；用户有工具访问权，Skill 提供专业知识 |

大多数 Skill 会偏向其中一个方向。知道你的用例更接近哪个，能帮你选择下面正确的 Pattern。

---

### 2. 五大设计模式（Patterns）

#### Pattern 1：顺序工作流编排（Sequential Workflow Orchestration）

**适用场景**：用户需要按特定顺序执行多步骤流程。

**示例——客户入职流程**：

```markdown
## Workflow: Onboard New Customer

### Step 1: Create Account
Call MCP tool: `create_customer`
Parameters: name, email, company

### Step 2: Setup Payment
Call MCP tool: `setup_payment_method`
Wait for: payment method verification

### Step 3: Create Subscription
Call MCP tool: `create_subscription`
Parameters: plan_id, customer_id (from Step 1)

### Step 4: Send Welcome Email
Call MCP tool: `send_email`
Template: welcome_email_template
```

**关键技巧**：
- **明确的步骤排序**——不能乱序
- **步骤间的依赖关系**——Step 3 需要 Step 1 的 customer_id
- **每步验证**——Step 2 要等支付验证完成
- **失败回滚指令**——出错了怎么办

---

#### Pattern 2：多 MCP 协调（Multi-MCP Coordination）

**适用场景**：工作流跨越多个服务。

**示例——设计到开发交接**：

| 阶段 | MCP | 操作 |
|------|-----|------|
| Phase 1：设计导出 | Figma MCP | 导出设计资产、生成设计规格、创建资产清单 |
| Phase 2：资产存储 | Drive MCP | 创建项目文件夹、上传所有资产、生成分享链接 |
| Phase 3：任务创建 | Linear MCP | 创建开发任务、附加资产链接、分配给工程团队 |
| Phase 4：通知 | Slack MCP | 在 #engineering 频道发交接摘要、包含资产链接和任务引用 |

**关键技巧**：
- **清晰的阶段分隔**
- **MCP 之间的数据传递**（Phase 2 的链接传给 Phase 3）
- **进入下一阶段前做验证**
- **集中式错误处理**

---

#### Pattern 3：迭代精炼（Iterative Refinement）

**适用场景**：输出质量通过迭代可以提升。

**示例——报告生成**：

```markdown
## Iterative Report Creation

### Initial Draft
1. Fetch data via MCP
2. Generate first draft report
3. Save to temporary file

### Quality Check
1. Run validation script: `scripts/check_report.py`
2. Identify issues:
   - Missing sections
   - Inconsistent formatting
   - Data validation errors

### Refinement Loop
1. Address each identified issue
2. Regenerate affected sections
3. Re-validate
4. Repeat until quality threshold met

### Finalization
1. Apply final formatting
2. Generate summary
3. Save final version
```

**关键技巧**：
- **明确的质量标准**——什么算"够好"
- **迭代改进**——不是一次搞定，而是循环提升
- **验证脚本**——用代码而非语言判断来检查质量
- **知道何时停止迭代**——设定终止条件，避免无限循环

---

#### Pattern 4：上下文感知工具选择（Context-Aware Tool Selection）

**适用场景**：同样的目标，根据上下文使用不同的工具。

**示例——智能文件存储**：

```markdown
## Smart File Storage

### Decision Tree
1. Check file type and size
2. Determine best storage location:
   - Large files (>10MB): Use cloud storage MCP
   - Collaborative docs: Use Notion/Docs MCP
   - Code files: Use GitHub MCP
   - Temporary files: Use local storage

### Execute Storage
Based on decision:
- Call appropriate MCP tool
- Apply service-specific metadata
- Generate access link

### Provide Context to User
Explain why that storage was chosen
```

**关键技巧**：
- **清晰的决策标准**——不含糊
- **备选方案**（fallback options）
- **透明度**——向用户解释为什么选了这个方案

---

#### Pattern 5：领域专业知识嵌入（Domain-Specific Intelligence）

**适用场景**：Skill 需要添加超越工具访问的**专业领域知识**。

**示例——支付合规处理**：

```markdown
## Payment Processing with Compliance

### Before Processing (Compliance Check)
1. Fetch transaction details via MCP
2. Apply compliance rules:
   - Check sanctions lists
   - Verify jurisdiction allowances
   - Assess risk level
3. Document compliance decision

### Processing
IF compliance passed:
  - Call payment processing MCP tool
  - Apply appropriate fraud checks
  - Process transaction
ELSE:
  - Flag for review
  - Create compliance case

### Audit Trail
- Log all compliance checks
- Record processing decisions
- Generate audit report
```

**关键技巧**：
- **领域专业知识嵌入逻辑中**——合规规则不是工具提供的，是 Skill 知道的
- **先合规再操作**（Compliance before action）
- **全面的文档记录**
- **清晰的治理结构**

---

#### 五大模式对比总结

| Pattern | 核心场景 | 关键特征 | 典型示例 |
|---------|---------|---------|---------|
| 1. 顺序工作流 | 固定步骤、有先后依赖 | 步骤排序、依赖传递、每步验证 | 客户入职 |
| 2. 多 MCP 协调 | 跨多个服务 | 阶段分隔、数据传递、集中错误处理 | 设计→开发交接 |
| 3. 迭代精炼 | 质量可迭代提升 | 质量标准、验证脚本、终止条件 | 报告生成 |
| 4. 上下文感知 | 同目标不同手段 | 决策树、fallback、透明度 | 智能文件存储 |
| 5. 领域知识 | 超越工具的专业判断 | 嵌入规则、先验证再执行、审计追踪 | 支付合规 |

---

### 3. 故障排查手册（Troubleshooting）

#### 3.1 Skill 上传失败

**问题一**：`"Could not find SKILL.md in uploaded folder"`
- **原因**：文件名不是精确的 `SKILL.md`
- **解法**：重命名为 `SKILL.md`（大小写敏感）

**问题二**：`"Invalid frontmatter"`
- **原因**：YAML 格式错误
- **常见错误**：

```yaml
# ❌ 错——缺少分隔符
name: my-skill
description: Does things

# ❌ 错——引号未闭合
name: my-skill
description: "Does things

# ✅ 对
---
name: my-skill
description: Does things
---
```

**问题三**：`"Invalid skill name"`
- **原因**：名称包含空格或大写
- **解法**：

```yaml
# ❌ 错
name: My Cool Skill

# ✅ 对
name: my-cool-skill
```

---

#### 3.2 Skill 不触发（Undertriggering）

**症状**：Skill 从来不自动加载。

**排查方法**：
1. 检查 description 字段——是不是太笼统了？
2. 是否包含了用户会实际说的触发短语？
3. 是否提到了相关的文件类型？

**调试技巧**：

> 直接问 Claude："When would you use the [skill name] skill?" Claude 会引用 description 回答你。根据它的回答来调整缺失的部分。

---

#### 3.3 Skill 过度触发（Overtriggering）

**症状**：Skill 在不相关的查询上也加载了。

**三个解法**：

**方法一：添加负触发条件**
```yaml
description: Advanced data analysis for CSV files. Use for statistical
  modeling, regression, clustering. Do NOT use for simple data exploration
  (use data-viz skill instead).
```

**方法二：更具体**
```yaml
# ❌ 太宽泛
description: Processes documents

# ✅ 更具体
description: Processes PDF legal documents for contract review
```

**方法三：明确作用域**
```yaml
description: PayFlow payment processing for e-commerce. Use specifically
  for online payment workflows, not for general financial queries.
```

---

#### 3.4 指令不被遵循（Instructions Not Followed）

**症状**：Skill 加载了，但 Claude 不按指令行事。

**常见原因及解法**：

| 原因 | 解法 |
|------|------|
| **指令太长** | 保持简洁；用列表和编号；把详细内容移到 `references/` |
| **关键指令被埋没** | 把重要指令放在**最前面**；用 `## Important` 或 `## Critical` 标题；必要时重复关键点 |
| **语言含糊** | 用精确语言替代模糊表达（见下方对比） |
| **模型懒惰（Model "laziness"）** | 这是官方认可的已知问题——Claude 有时会走捷径、跳过步骤。解法：添加明确鼓励语（见下方），但注意放在 user prompt 中比放在 SKILL.md 中更有效 |

```markdown
# ❌ 含糊
Make sure to validate things properly

# ✅ 精确
CRITICAL: Before calling create_project, verify:
- Project name is non-empty
- At least one team member assigned
- Start date is not in the past
```

> **高级技巧**：对于关键验证，考虑**捆绑一个脚本**来程序化执行检查，而不是依赖语言指令。代码是确定性的；语言理解不是。参见 Office skills 中的此模式示例。

**添加明确鼓励（但注意放置位置）**：

```markdown
## Performance Notes
- Take your time to do this thoroughly
- Quality is more important than speed
- Do not skip validation steps
```

> **注意**：这种鼓励语添加在**用户 prompt 中**比放在 SKILL.md 中更有效。

---

#### 3.5 MCP 连接问题

**症状**：Skill 加载了但 MCP 调用失败。

**排查清单**：

1. **验证 MCP server 连接状态**
   - Claude.ai：Settings > Extensions > [Your Service] → 应显示 "Connected"

2. **检查认证**
   - API key 是否有效且未过期
   - 权限/scope 是否正确
   - OAuth token 是否已刷新

3. **独立测试 MCP**
   - 直接让 Claude 调用 MCP（不通过 Skill）
   - 例："Use [Service] MCP to fetch my projects"
   - 如果这也失败，问题在 MCP 不在 Skill

4. **验证工具名称**
   - Skill 中引用的 MCP tool name 是否正确
   - 查阅 MCP server 文档
   - tool name 是**大小写敏感**的

---

#### 3.6 大上下文问题（Large Context Issues）

**症状**：Skill 运行变慢或回复质量下降。

**可能原因**：
- Skill 内容太大
- 同时启用了太多 Skill
- 没有使用渐进式披露，所有内容一次性加载

**解法**：

| 策略 | 具体做法 |
|------|----------|
| **优化 SKILL.md 大小** | 把详细文档移到 `references/`；用链接代替内联；**SKILL.md 控制在 5000 词以内** |
| **减少同时启用的 Skill** | 评估是否同时启用了超过 20-50 个 Skill；推荐选择性启用；考虑为相关能力建"Skill 包" |

---

### 4. Reference A：上传前后快速清单

#### 开始前

- [ ] 已确定 2-3 个具体用例
- [ ] 已确认所需工具（内置或 MCP）
- [ ] 已阅读本指南和示例 Skill
- [ ] 已规划文件夹结构

#### 开发中

- [ ] 文件夹名用 kebab-case
- [ ] SKILL.md 文件存在（精确拼写）
- [ ] YAML frontmatter 有 `---` 分隔符
- [ ] name 字段：kebab-case，无空格，无大写
- [ ] description 包含 WHAT 和 WHEN
- [ ] 没有 XML 标签（`<` `>`）
- [ ] 指令清晰可操作
- [ ] 包含错误处理
- [ ] 提供示例
- [ ] 引用的参考文件路径正确

#### 上传前

- [ ] 对明显任务的触发测试通过
- [ ] 对改述请求的触发测试通过
- [ ] 确认不会在无关话题上触发
- [ ] 功能测试通过
- [ ] 工具集成正常（如适用）
- [ ] 已压缩为 .zip 文件

#### 上传后

- [ ] 在真实对话中测试
- [ ] 监控欠触发/过触发
- [ ] 收集用户反馈
- [ ] 迭代 description 和指令
- [ ] 更新 metadata 中的版本号

---

### 5. Reference B：YAML Frontmatter 完整参考

**必需字段**：

```yaml
---
name: skill-name-in-kebab-case
description: What it does and when to use it. Include specific trigger phrases.
---
```

**所有可选字段**：

```yaml
name: skill-name
description: [required description]
license: MIT                              # 开源许可证
allowed-tools: "Bash(python:*) Bash(npm:*) WebFetch"  # 限制工具访问
metadata:                                 # 自定义元数据
  author: Company Name
  version: 1.0.0
  mcp-server: server-name
  category: productivity
  tags: [project-management, automation]
  documentation: https://example.com/docs
  support: support@example.com
```

**安全注意事项**：

| 规则 | 说明 |
|------|------|
| 允许 | 任何标准 YAML 类型（字符串、数字、布尔、列表、对象）；自定义 metadata 字段；description 最长 1024 字符 |
| 禁止 | XML 尖括号（`<` `>`）——安全限制；YAML 中的代码执行（使用安全 YAML 解析）；name 以 "claude" 或 "anthropic" 开头（保留） |

> **注意 `allowed-tools` 字段**：这是一个可选字段，可以限制 Skill 能使用的工具。格式示例：`"Bash(python:*) Bash(npm:*) WebFetch"`。

---

### 6. Reference C：完整 Skill 示例资源

PDF 推荐了以下资源作为学习和参考：

| 资源 | 说明 |
|------|------|
| **Document Skills** | PDF、DOCX、PPTX、XLSX 创建 |
| **Example Skills** | 公开仓库 `GitHub: anthropic/skills`，包含 Anthropic 创建的可定制 Skill，各种工作流模式示例。建议 clone 下来修改后作为自己的模板 |
| **Partner Skills Directory** | 来自 Asana、Atlassian、Canva、Figma、Sentry、Zapier 等合作伙伴的 Skill |

**其他资源**：

| 类型 | 资源 |
|------|------|
| 官方文档 | Best Practices Guide、Skills Documentation、API Reference、MCP Documentation |
| 博客文章 | Introducing Agent Skills、Engineering Blog: Equipping Agents for the Real World、Skills Explained、How to Create Skills for Claude、Building Skills for Claude Code、Improving Frontend Design through Skills |
| 工具 | skill-creator（内置 Claude.ai，可下载到 Claude Code）；可评估你的 Skill 并建议改进 |
| 技术支持 | Claude Developers Discord（通用问题）；GitHub Issues: anthropics/skills/issues（Bug 报告，需附 Skill 名称、错误信息、复现步骤） |

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **Problem-first vs Tool-first**：两种思路决定你选哪个 Pattern——前者编排调用，后者提供专业知识
2. **五大模式各有适用场景**：顺序流程（固定步骤）、多 MCP（跨服务）、迭代精炼（质量驱动）、上下文感知（同目标不同手段）、领域知识（超越工具的判断力）
3. **SKILL.md 大小写敏感是第一大坑**：必须精确拼写，YAML 必须有 `---` 分隔符
4. **指令不遵循的高级解法**：用脚本做确定性验证，而非依赖语言指令——代码是确定性的，语言理解不是
5. **"鼓励语"放 user prompt 比放 SKILL.md 更有效**：这个发现很反直觉但实际有效
6. **SKILL.md 控制在 5000 词以内**：太大会导致性能问题，详细内容移到 references/
7. **同时启用超过 20-50 个 Skill 可能出问题**：考虑选择性启用或建 Skill 包
8. **`allowed-tools` 字段**：可以限制 Skill 能使用的工具，增加安全性
9. **上传前清单是保底机制**：开发中对着清单逐项核实，能避免大部分常见错误
