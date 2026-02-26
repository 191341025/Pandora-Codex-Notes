# 模块一：Cowork 概述与核心架构

> 对应 PDF 第 1、4-5 页

---

## 概念讲解

### 1. Cowork 是什么

**定义**：Cowork 是 Claude Desktop 应用中的一个 **Research Preview 功能**，把 Claude Code 的 Agent 能力带到了编程之外的知识工作场景。简单说就是——你不用开终端，也能让 Claude 像一个真正的同事一样，自主完成复杂的多步骤任务。

**核心思想**：从"你问我答"的聊天模式，进化为"你描述目标、我交付成果"的委托模式。Anthropic 自己的描述是：感觉更像是给同事留消息（leaving messages for a coworker），而不是和聊天机器人一来一回。

**背景信息**：
- **来源**：Anthropic 官方出品，基于 Claude Code 的同一套 Agent 架构
- **平台**：目前仅限 Claude Desktop for macOS（Windows 版规划中）
- **订阅要求**：Pro、Max、Team、Enterprise 计划
- **开发周期**：Anthropic 团队用 Claude Code 本身来构建 Cowork，据报道大约只花了 10 天
- **官方文档源**：
  - Getting Started with Cowork: `https://support.claude.com/en/articles/13345190-getting-started-with-cowork`
  - Using Cowork Safely: `https://support.claude.com/en/articles/13364135-using-cowork-safely`
  - Cowork for Team and Enterprise: `https://support.claude.com/en/articles/13455879-cowork-for-team-and-enterprise-plans`
  - Knowledge Work Plugins (GitHub): `https://github.com/anthropics/knowledge-work-plugins`

> **有意思的点**：Cowork 是用 Claude Code 自己开发出来的——这本身就是 Agent 能力的最佳证明。

**与传统 Chat 的本质区别**：

| 维度 | 传统 Chat | Cowork |
|------|-----------|--------|
| 交互模式 | 一问一答 | 委托任务，异步执行 |
| 文件访问 | 需手动上传/下载 | 直接读写本地文件 |
| 任务复杂度 | 单轮/少轮对话 | 多步骤、长时间运行 |
| 输出形式 | 文本回复 | 文档、表格、PPT 等成品文件 |
| 并行能力 | 无 | Sub-Agent 多线程并行 |
| 超时限制 | 有对话超时和上下文限制 | 无，可长时间运行 |

---

### 2. 六大核心能力（Key Capabilities）

**（1）直接本地文件访问（Direct Local File Access）**

Claude 可以直接读写你电脑上的本地文件，不需要你手动上传或下载。这意味着你可以说"整理我 Downloads 文件夹里的文件"，Claude 就能直接去操作。

**（2）Sub-Agent 协调（Sub-agent Coordination）**

Claude 会把复杂任务拆解成更小的子任务，然后协调多个并行工作流同时完成。这是 Cowork 能处理大规模任务的关键——不是串行一步步做，而是像一个项目经理一样分配工作。

**（3）专业级输出（Professional Outputs）**

能生成真正可用的专业文档：
- Excel 表格（带 VLOOKUP、条件格式等**可工作的公式**）
- PowerPoint 演示文稿
- 格式化文档

> **关键区别**：不是生成一个需要你再手动修复的 CSV，而是直接给你一个公式都能正常运行的 .xlsx 文件。

**（4）长时间任务（Long-running Tasks）**

不会因为对话超时或上下文限制而中断。复杂任务可以运行很长时间，你可以离开去做别的事，回来看结果。

**（5）浏览器集成（Browser Integration）**

搭配 Claude in Chrome 扩展使用时，Claude 还能完成需要浏览器访问的任务。但需注意安全风险（详见模块四）。

**（6）连接器集成（Connector Integration）**

可以和 Asana、Notion 等工具集成，还包含 Skills 来提升文档和演示文稿的创建质量。

---

### 3. 任务执行流程

当你在 Cowork 中启动一个任务时，Claude 会按以下 5 步执行：

```
1. 分析请求 → 制定计划
2. 拆解子任务（如果是复杂任务）
3. 在 VM 环境中执行工作
4. 协调多个并行工作流（如果合适）
5. 将完成的输出直接交付到你的文件系统
```

**全程透明**：你可以看到 Claude 正在计划什么、正在做什么。可以随时介入纠偏，也可以放手让 Claude 独立运行。

**适用场景**：
- 需要处理多个文件的批量操作
- 需要多步骤分析的研究工作
- 从零散素材生成完整文档
- 需要长时间运行的数据处理

---

### 4. VM 架构（虚拟机隔离）

**技术实现**：Cowork 使用 Apple 的 **VZVirtualMachine（Virtualization Framework）** 在你的电脑上运行一个虚拟机。

**运行机制**：
1. 下载并启动一个**定制的 Linux 根文件系统**
2. 你授权访问的文件会被**挂载**到 VM 内的沙箱路径：`/sessions/[session-name]/mnt/[folder]`
3. Claude 的所有操作都在这个隔离环境中执行

**安全意义**：
- **隔离性**：VM 环境与你的主操作系统是分离的
- **受控环境**：Claude 在定义好的边界内操作，文件和网络访问都是受控的
- **但不是绝对安全**：Claude 确实能访问你授权的本地文件，能对这些文件做实际修改

> **注意**：虽然有 VM 隔离，但 Claude 对你授权的文件是有真实读写能力的。在处理敏感文件时，务必审查 Claude 的计划后再放行。

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **Cowork = Claude Code 的能力 + Desktop 的易用性**：不用开终端，也能让 Claude 自主完成多步骤知识工作
2. **委托模式而非对话模式**：描述目标 → 离开 → 回来看成品，像给同事分配任务
3. **Sub-Agent 并行是核心竞争力**：复杂任务不是串行执行，而是拆解后多线程并行
4. **VM 隔离提供安全边界**：Apple VZVirtualMachine + Linux 沙箱，但授权文件仍可被真实修改
5. **Research Preview 状态**：功能仍在预览阶段，有已知限制（详见模块五）
6. **10 天开发周期**：Cowork 本身是用 Claude Code 构建的，这是 Agent 能力的自证
