# 模块三：Testing and Iteration

> 对应 PDF 第 14-17 页

---

## 概念讲解

### 1. 三层测试方法

Skill 可以在不同严格程度下测试，取决于你的需求：

| 方法 | 适用场景 | 特点 |
|------|----------|------|
| **手动测试（Claude.ai）** | 快速迭代、小规模 Skill | 直接在对话中运行查询，观察行为。无需配置，最快上手 |
| **脚本测试（Claude Code）** | 需要可重复验证 | 自动化测试用例，跨版本一致验证 |
| **编程测试（Skills API）** | 企业级、大规模部署 | 构建评估套件，系统性地跑预定义测试集 |

**选择依据**：用于小团队内部的 Skill 和部署给成千上万企业用户的 Skill，测试需求完全不同。按你的质量要求和 Skill 的可见度来选。

> **Pro Tip**：先在**单个任务上迭代**直到成功，然后再抽象为 Skill。这利用了 Claude 的 in-context learning，比广撒网测试更快获得反馈信号。

---

### 2. 推荐测试方案——三个维度

#### 2.1 触发测试（Triggering Tests）

**目标**：确保 Skill 在正确的时机加载。

**测试矩阵**：

| 类型 | 目的 | 示例 |
|------|------|------|
| ✅ 应该触发——直接任务 | 检查明显的任务请求 | "Help me set up a new ProjectHub workspace" |
| ✅ 应该触发——改述请求 | 检查换种说法的请求 | "I need to create a project in ProjectHub" |
| ✅ 应该触发——变体请求 | 检查不同表达方式 | "Initialize a ProjectHub project for Q4 planning" |
| ❌ 不应触发——无关话题 | 确保不误触发 | "What's the weather in San Francisco?" |
| ❌ 不应触发——相似但不相关 | 精确边界 | "Help me write Python code" |
| ❌ 不应触发——相邻功能 | 避免越界 | "Create a spreadsheet"（除非 Skill 确实处理表格） |

#### 2.2 功能测试（Functional Tests）

**目标**：验证 Skill 产出正确的输出。

**测试清单**：
- 有效输出是否正确生成
- API 调用是否成功
- 错误处理是否工作
- 边缘情况是否覆盖

**示例——具体写法**：

```
Test: Create project with 5 tasks
Given: Project name "Q4 Planning", 5 task descriptions
When: Skill executes workflow
Then:
  - Project created in ProjectHub
  - 5 tasks created with correct properties
  - All tasks linked to project
  - No API errors
```

#### 2.3 性能对比（Performance Comparison）

**目标**：证明 Skill 确实比没有 Skill 时表现更好。

用模块二中定义的成功标准来对比。以下是一个对比示例：

| 指标 | 无 Skill | 有 Skill |
|------|----------|----------|
| 用户交互 | 每次都要提供指令 | 自动执行工作流 |
| 来回对话次数 | 15 条消息 | 2 个澄清问题 |
| API 调用失败次数 | 3 次（需重试） | 0 次 |
| Token 消耗 | 12,000 | 6,000 |

---

### 3. skill-creator 工具

**定义**：`skill-creator` 是 Anthropic 提供的一个内置 Skill，帮你**构建和迭代**其他 Skill。在 Claude.ai 上通过插件目录可用，也可下载到 Claude Code。

**能做什么**：

| 功能 | 说明 |
|------|------|
| **创建 Skill** | 从自然语言描述生成 Skill；生成格式正确的 SKILL.md（含 frontmatter）；建议触发短语和结构 |
| **审查 Skill** | 标记常见问题（模糊 description、缺少触发条件、结构问题）；识别过触发/欠触发风险；根据 Skill 的声明用途建议测试用例 |
| **迭代改进** | 使用中遇到边缘案例或失败时，带着这些例子回来让 skill-creator 改进 |

**使用方式**：

```
"Use the skill-creator skill to help me build a skill for [your use case]"
```

**迭代改进的示例用法**：

```
"Use the issues & solution identified in this chat to improve how the
skill handles [specific edge case]"
```

> **重要限制**：skill-creator 帮你**设计和精炼** Skill，但它**不会执行自动化测试套件**，也不产出定量评估结果。测试还是得你自己做。

**效率承诺**：如果你已经有 MCP server 并且知道你的 2-3 个核心工作流，使用 skill-creator 可以在 **15-30 分钟内**构建并测试一个功能性 Skill。

---

### 4. 基于反馈的迭代

**核心理念**：Skill 是**活文档**（living documents）。你需要根据实际使用反馈持续迭代。

#### 4.1 欠触发（Undertriggering）

**信号**：
- Skill 在应该加载时没有加载
- 用户需要手动启用
- 客服收到"什么时候用这个"的问题

**解法**：给 description 字段添加**更多细节和关键词**——特别是技术术语。Claude 是根据 description 来判断相关性的，信息越丰富越好。

#### 4.2 过触发（Overtriggering）

**信号**：
- Skill 在不相关的查询上也加载了
- 用户开始主动禁用它
- 大家搞不清楚这个 Skill 到底是干嘛的

**三种解法**：

**方法一：添加负触发条件**
```yaml
description: Advanced data analysis for CSV files. Use for statistical
  modeling, regression, clustering. Do NOT use for simple data exploration
  (use data-viz skill instead).
```

**方法二：更具体**
```yaml
# 太宽泛
description: Processes documents

# 更具体
description: Processes PDF legal documents for contract review
```

**方法三：明确作用域**
```yaml
description: PayFlow payment processing for e-commerce. Use specifically
  for online payment workflows, not for general financial queries.
```

#### 4.3 执行问题（Execution Issues）

**信号**：
- 结果不一致
- API 调用失败
- 需要用户纠正

**解法**：改进指令，添加错误处理。参见模块二的指令编写最佳实践。

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **先在单个任务上迭代成功，再抽象为 Skill**——比广撒网测试效率高得多
2. **触发测试三方向**：应该触发（含改述）、不应触发（含相邻功能）
3. **功能测试用 Given-When-Then 格式**：具体、可重复、有预期结果
4. **性能对比是核心论证**：有 Skill vs 无 Skill 的 token 消耗、对话轮次、失败率对比
5. **skill-creator ≠ 自动化测试工具**：它帮你设计和审查，但不跑测试
6. **Skill 是活文档**：必须根据欠触发/过触发/执行问题持续迭代
7. **负触发条件**：明确说"不要在 X 场景使用"，是解决过触发的有效手段
