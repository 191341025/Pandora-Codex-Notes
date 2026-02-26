# 模块四：Distribution and Sharing

> 对应 PDF 第 18-20 页

---

## 概念讲解

### 1. 当前分发模型（2026年1月）

#### 1.1 个人用户安装方式

1. 下载 Skill 文件夹
2. 打包成 .zip（如需要）
3. 上传到 Claude.ai：**Settings > Capabilities > Skills**
4. 或者直接放到 Claude Code 的 skills 目录中

#### 1.2 组织级部署

- 管理员可以**在整个工作区范围内部署** Skill（此功能于 2025 年 12 月 18 日上线）
- 支持**自动更新**
- 提供**集中管理**能力

> **对 MCP 开发者的价值**：Skill 让你的 MCP 集成更完整。当用户在比较不同的 connector 时，配有 Skill 的方案能更快地交付价值，这是一个竞争优势。

---

### 2. 开放标准（Open Standard）

**核心声明**：Anthropic 已将 Agent Skills 发布为**开放标准**。

**设计理念**：和 MCP 一样，Anthropic 认为 Skill 应该**跨工具和平台可移植**——同一个 Skill 无论你用 Claude 还是其他 AI 平台都应该能工作。

**现实限定**：
- 有些 Skill 是为了充分利用**特定平台的能力**而设计的
- 作者可以在 Skill 的 `compatibility` 字段中注明这一点
- Anthropic 正与生态系统成员合作推进标准，早期采用情况良好

---

### 3. 通过 API 使用 Skills

对于需要编程方式使用 Skill 的场景——比如构建应用、Agent 或自动化工作流——API 提供了直接控制能力。

**API 关键能力**：

| 能力 | 说明 |
|------|------|
| `/v1/skills` 端点 | 列出和管理 Skill |
| `container.skills` 参数 | 在 Messages API 请求中添加 Skill |
| 版本控制和管理 | 通过 Claude Console 操作 |
| Agent SDK 集成 | 与 Claude Agent SDK 配合构建自定义 Agent |

**什么场景用 API vs Claude.ai**：

| 使用场景 | 最佳平台 |
|----------|----------|
| 终端用户直接与 Skill 交互 | Claude.ai / Claude Code |
| 开发过程中手动测试和迭代 | Claude.ai / Claude Code |
| 个人临时工作流 | Claude.ai / Claude Code |
| **应用中编程使用 Skill** | **API** |
| **大规模生产部署** | **API** |
| **自动化管道和 Agent 系统** | **API** |

> **技术前提**：API 中使用 Skills 需要 **Code Execution Tool beta**，它提供了 Skill 运行所需的安全环境。

**相关文档**（PDF 中提到的）：
- Skills API Quickstart
- Create Custom Skills
- Skills in the Agent SDK

---

### 4. 推荐分发策略

PDF 给出了一个今天就能用的实操路径：

#### 第一步：托管到 GitHub

- 公开仓库用于开源 Skill
- 清晰的 README（给人类看的，注意这是**仓库级别**的 README，不是放在 Skill 文件夹里面的）
- 包含使用示例和截图

#### 第二步：在 MCP 文档中关联

- 从 MCP 文档链接到 Skill
- 解释为什么两者一起使用更有价值
- 提供快速入门指南

#### 第三步：创建安装指南

PDF 中给出的模板：

```markdown
# Installing the [Your Service] skill

1. Download the skill:
   - Clone repo: `git clone https://github.com/yourcompany/skills`
   - Or download ZIP from Releases

2. Install in Claude:
   - Open Claude.ai > Settings > Skills
   - Click "Upload skill"
   - Select the skill folder (zipped)

3. Enable the skill:
   - Toggle on the [Your Service] skill
   - Ensure your MCP server is connected

4. Test:
   - Ask Claude: "Set up a new project in [Your Service]"
```

---

### 5. 如何定位你的 Skill（Positioning）

**原则**：你怎么描述 Skill 决定了用户是否理解其价值、是否愿意尝试。

#### 5.1 聚焦结果，而非功能

```markdown
# ✅ 好——强调结果
"The ProjectHub skill enables teams to set up complete project
workspaces in seconds — including pages, databases, and
templates — instead of spending 30 minutes on manual setup."

# ❌ 坏——描述技术细节
"The ProjectHub skill is a folder containing YAML frontmatter
and Markdown instructions that calls our MCP server tools."
```

#### 5.2 讲好 MCP + Skills 的故事

```markdown
"Our MCP server gives Claude access to your Linear projects.
Our skills teach Claude your team's sprint planning workflow.
Together, they enable AI-powered project management."
```

> **关键洞察**：用户不关心 Skill 的技术实现（文件夹、YAML、Markdown），他们关心的是**结果**（节省了多少时间、减少了多少手动操作）。定位时永远从用户视角出发。

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **个人用户**：下载 → 打包 → 上传到 Settings > Capabilities > Skills
2. **组织级部署**：管理员可全工作区推送、自动更新、集中管理（2025.12.18 上线）
3. **开放标准**：Skill 被设计为跨平台可移植，类似 MCP 的定位
4. **API 使用需要 Code Execution Tool beta**：不是所有 API 用户都能直接用
5. **API 适合场景**：大规模生产部署、自动化管道、Agent 系统
6. **推荐分发路径**：GitHub 公开仓库 → MCP 文档关联 → 安装指南
7. **定位法则**：聚焦结果而非功能——用户不关心 YAML，关心的是节省 30 分钟
