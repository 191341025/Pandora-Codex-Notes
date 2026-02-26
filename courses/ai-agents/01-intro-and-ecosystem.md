# 模块一：Introduction & Agent 生态

> 对应 PDF 第 1-10 页

---

## 概念讲解

### 1. AI Agent 是什么？

**定义**：AI Agent（AI 代理）是一种结合了高级 AI 模型智能 + 工具调用能力的系统，能够代替你执行操作，且在你的控制之下。

**通俗理解**：以前的 AI 只能"回答问题"，Agent 能"替你干活"。

**范式转变**：
```
传统 AI：用户提问 → AI 回答 → 完了
AI Agent：用户给目标 → Agent 规划步骤 → 调用工具执行 → 观察结果 → 继续 → 交付成果
```

**白皮书原话（Google Cloud CEO Thomas Kurian）**：
> "Agentic workflow 不只是问问题得到答案。而是给 AI 一个复杂目标——比如'规划这次产品发布'或'解决这个供应链中断'——让它自己编排完成所需的多步骤任务。"

---

### 2. 这份指南的定位

这是一份**面向创业公司和开发者**的技术指南，聚焦如何在 Google Cloud 上构建生产级 AI Agent。

**按你的阶段选择阅读路径**：

| 你的阶段 | 建议 |
|---------|------|
| 刚接触 AI Agent | 从 Section 1（核心概念）开始 |
| 准备动手构建 | 直接跳到 Section 2（ADK 构建） |
| Agent 已经搭好了 | 看 Section 3（安全、稳定、可扩展） |

---

### 3. Google Cloud Agent 生态总览

Google 提供了**三条路径**来使用 AI Agent：

```
┌──────────────────┬───────────────────┬──────────────────┐
│   自己构建        │   使用 Google 预建  │   引入合作伙伴     │
│   Build your own  │   Use Google Cloud │   Bring in        │
│                  │   agents           │   partner agents   │
├──────────────────┴───────────────────┴──────────────────┤
│        MCP 和 A2A 协议实现互操作                          │
└─────────────────────────────────────────────────────────┘
```

---

### 4. 路径一：自己构建（Build your own）

有两个子选项：

#### A. ADK（Agent Development Kit）— 代码优先

**适合人群**：开发者、技术创业团队、需要高度控制的场景

**核心能力**：
| 能力 | 说明 |
|------|------|
| 编排逻辑 (Orchestration) | Agent 的核心推理过程，如 ReAct 框架 |
| 工具定义与注册 | 自定义函数和 API，让 Agent 能操作数据和外部系统 |
| 上下文管理 | 记忆系统——记住用户偏好和对话历史 |
| 评估与可观测性 | 内置测试、调试、生产监控工具 |
| 容器化 | 打包成标准容器，可部署到任何环境 |
| 多 Agent 协作 | 多个专业 Agent 协同工作 |

**对创业公司的意义**：
- 不只是做聊天机器人，而是**自动化工作流**
- 连接你的私有 API 和数据，建立**竞争壁垒**
- 利用 Memory 系统**记住客户**，提供个性化体验
- 内置评估工具，**有信心地发布**

#### B. Google Agentspace — 无代码/应用优先

**适合人群**：成熟创业团队、非技术人员也需要参与构建 Agent 的场景

**核心能力**：
- 统一跨公司 SaaS 应用的搜索
- 多模态数据理解（文本、图片、图表、视频）
- 预建 Agent 库（深度研究、创意生成等）
- 无代码 Agent 设计器（Agent Designer）

---

### 5. 路径二：使用 Google Cloud 预建 Agent

Google 提供了多个开箱即用的 AI 助手：

| 产品 | 用途 | 亮点 |
|------|------|------|
| **Gemini Code Assist** | 编程助手 | IDE 集成、PR 审查、多文件重构、Agent 驱动开发 |
| **Gemini Cloud Assist** | 云基础设施管理 | 自然语言设计架构、排障、安全分析、成本优化 |
| **Gemini in Colab Enterprise** | 数据科学 | Python 代码生成、数据可视化、笔记本自动摘要 |

**Gemini Code Assist 特别值得关注**：
- 支持 VS Code、JetBrains、Android Studio
- 命令行工具 Gemini CLI（开源）
- GitHub PR 自动审查
- 能做**跨文件重构**的 Agent 驱动开发
- 支持 MCP 协议的 Human-in-the-Loop 工作流

---

### 6. 路径三：引入合作伙伴 Agent

通过 Google Cloud Marketplace 和 Agent Garden，可以集成第三方或开源 Agent。这些 Agent 支持 A2A 协议，能与你自建的 Agent 互操作。

---

### 7. MCP 与 A2A 协议初识

这两个协议是 Google Agent 生态的"粘合剂"：

| 协议 | 全称 | 作用 |
|------|------|------|
| **MCP** | Model Context Protocol | 标准化 Agent 连接外部数据源和工具的方式 |
| **A2A** | Agent2Agent Protocol | 标准化不同 Agent 之间通信协作的方式 |

**类比理解**：
- MCP 就像 USB——让 Agent 能"插上"各种外部工具
- A2A 就像 HTTP——让不同 Agent 之间能"对话"

> 这两个协议在模块四会深入讲解。

---

## 问答记录

> 待补充

---

## 重点标记

1. **Agent ≠ 聊天机器人**：Agent 能规划、执行、使用工具、完成复杂任务
2. **三条路径选择**：代码优先用 ADK，无代码用 Agentspace，快速集成用预建 Agent
3. **MCP + A2A 是生态基础**：一个管工具连接，一个管 Agent 间通信
4. **ADK 是本指南的核心主线**：后续 Section 2 和 3 都围绕 ADK 展开
5. **容器化是关键设计**：ADK 构建的 Agent 可以部署到任何容器环境
