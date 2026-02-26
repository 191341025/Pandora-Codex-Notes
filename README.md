<div align="center">

# Pandora-Codex-Notes

**英文技术文档的中文结构化精读**

*English technical docs → Structured Chinese notes.*
*Turn dense documentation into clear mental models.*

[![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/)
[![GitHub stars](https://img.shields.io/github/stars/191341025/Pandora-Codex-Notes?style=social)](https://github.com/191341025/Pandora-Codex-Notes)

[中文](#关于本仓库) · [English](#about)

</div>

---

## About

Structured Chinese learning notes distilled from English technical documents — official docs, guides, whitepapers, and academic papers, **across any domain**.

These are **not** translations or excerpts — each topic is systematically restructured with a pedagogical framework: concept maps, intuitive analogies (with explicit boundary annotations), comparison tables, and practical guidance.

**Who is this for:**
- Anyone who reads English technical docs and wants structured Chinese materials
- Engineers who want depth beyond surface-level tutorials
- Learners who prefer building mental models over memorizing facts

**Current focus:** AI & LLM technologies. More domains coming.

> [Jump to content catalog →](#课程笔记)

---

## 关于本仓库

我经常需要阅读各种英文技术文档——官方指南、技术白皮书、学术论文。为了真正消化这些内容，我把它们精炼成了**结构化的中文笔记**。

这些笔记不是简单的翻译或摘抄，而是经过教学设计的**知识重构**——
我的目标是：**读完我的笔记后，你能跳过原文的大部分内容，直接建立准确的心智模型。**

目前的内容以 **AI/LLM** 为主，但这套方法适用于任何技术领域。未来会逐步扩展到更多方向。

### 笔记有什么不同？

<table>
<tr>
<td width="50%">

**普通笔记**
- 逐页摘抄原文要点
- 中英混杂的流水账
- 读完还是得回去翻原文

</td>
<td width="50%">

**这里的笔记**
- 每个模块有**概念地图**，分"必须内化 / 动手要点 / 扩展理解"三层
- 用精心设计的**类比**建立直觉，并标注类比的边界
- 关键对比用**表格**一目了然
- 标注"什么时候用"和"常见误用"，面向实战

</td>
</tr>
</table>

---

## 学习路径：AI/LLM

当前 AI/LLM 方向的笔记遵循一条**从基础到生产的渐进式路径**，每一步都建立在前一步之上：

```
A  Prompt Engineering          ── 和 AI 对话的技术        ✅ 已完成
B  Tool Use / Function Calling ── 让 AI 调用外部工具      ✅ 已完成
C  Structured Output           ── 让 AI 输出可编程数据     🔜 计划中
D  Prompt Chaining / Workflow  ── 多步骤 AI 调用编排       🔜 计划中
E  RAG                         ── 给 AI 接入自定义知识      🔜 计划中
F  Agent Design Patterns       ── 让 AI 自主决策与行动     🔜 计划中
G  Evaluation & Testing        ── 系统化评估 AI 质量       🔜 计划中
H  Production Deployment       ── 生产环境落地             🔜 计划中
```

> 不追求一步到位，不跳跃式学习。学到"足够用"就往前走。

---

## 课程笔记

<!--
  状态说明：
  ✅ 已完成并验证  |  📝 进行中  |  🔜 计划中
-->

| 课程 | 主题 | 源材料作者 | 模块数 | 状态 |
|------|------|-----------|:------:|:----:|
| [Prompt Engineering](courses/prompt-engineering/) | 提示工程基础到高级技术 | Lee Boonstra · Google | 6 | ✅ |
| [Claude Tool Use](courses/claude-tool-use/) | Anthropic Tool Use API 完整指南 | Anthropic | 5 | ✅ |
| [Claude Code Guide](courses/claude-code-guide/) | Claude Code 钩子、扩展与专业模式 | Anthropic | 6 | ✅ |
| [Claude Skills Development](courses/claude-skills-development/) | Claude 技能从设计到分发 | Anthropic | 5 | ✅ |
| [Claude Cowork](courses/claude-cowork/) | Anthropic Cowork 协作平台 | Anthropic | 5 | ✅ |
| [Agentic Design Patterns](courses/agentic-design-patterns/) | 21 种 AI Agent 设计模式深度拆解 | Antonio Gulli | 18 | ✅ |
| [AI Agents Technical Guide](courses/ai-agents/) | AI Agent 技术全景指南 | Google Cloud | 8 | ✅ |
| [Building Effective Agents](courses/building-effective-agents/) | Anthropic 官方 Agent 构建白皮书 | Erik Schluntz, Barry Zhang · Anthropic | 5 | ✅ |
| [AnythingLLM](courses/anythingllm/) | 开源 LLM 平台：RAG、Agent、工作流 | Mintplex Labs | 6 | ✅ |
| [Google ADK](courses/google-adk/) | Google Agent Development Kit 开发实战 | Google | 6 | ✅ |
| [OpenAI Agents SDK](courses/openai-agents-sdk/) | OpenAI Agents SDK 从基础到多 Agent 系统 | Henry Habib · Packt | 9 | ✅ |
| [Vector Database](courses/vector-database/) | 向量数据库：从嵌入到 RAG 实战 | O'Reilly Media | 10 | ✅ |

> 更多课程持续整理中。关注本仓库以获取更新。

## 论文精读

学术论文（主要来自 [arXiv](https://arxiv.org/) 等开放平台）的结构化中文精读。每篇论文浓缩为**七个板块**：一句话总结 → 为什么值得关注 → 核心贡献 → 方法直觉 → 关键发现 → 局限与开放问题 → 启发与联想。

| 论文 | 领域 | 作者 | 关键词 |
|------|------|------|--------|
| [CLEF HIPE-2026](papers/2026-02-hipe2026/) | NLP / 数字人文 | Opitz, Raclé, Boros et al. | 多语言关系抽取、知识图谱、OCR |

> 有想精读的 arXiv 论文？同样欢迎[提交请求](https://github.com/191341025/Pandora-Codex-Notes/issues/new?template=course-request.yml)。

---

## 笔记长什么样？

以 **Prompt Engineering / Temperature 参数**为例——

> ### 直觉建立
>
> 想象你面前有一个装满彩球的箱子，每个球代表一个可能的下一个 token，球的大小代表它的概率——概率越高，球越大。
>
> - **Temperature = 0**：手只会抓到最大的那个球。每次都是。结果完全确定。
> - **Temperature 很低（0.1-0.2）**：大球还是占优势，但偶尔你也会摸到第二大的球。
> - **Temperature 接近 1**：球的大小差异被"压平"了，输出变得多样、有创意。
> - **Temperature 很高（>1）**：所有球几乎一样大，随便哪个 token 都可能被选中。
>
> **类比边界**：这个类比没有体现 temperature 对 softmax 函数的数学作用——实际上 temperature 是 softmax 的除数，低 temperature 让概率分布更"尖锐"（赢者通吃），高 temperature 让分布更"平坦"（雨露均沾）。

每个概念都有：**直觉类比 → 边界标注 → 参数推荐表 → 常见误用提醒 → 什么时候用**。

---

## 如何使用

**在线阅读**：直接在 GitHub 上点击目录浏览

**本地阅读**：
```bash
git clone https://github.com/191341025/Pandora-Codex-Notes.git
```
用任何 Markdown 编辑器（VS Code、Typora、Obsidian）打开即可。

---

## 想看某份文档变成结构化笔记？

你手上有一份**英文技术文档**（PDF / Markdown / 纯文本），不限领域，觉得内容不错但啃起来费劲？

**提一个 Issue，我来帮你结构化。**

<a href="https://github.com/191341025/Pandora-Codex-Notes/issues/new?template=course-request.yml">
  <img src="https://img.shields.io/badge/-提交文档请求-blue?style=for-the-badge" alt="Request">
</a>

提交时请告诉我：
- 文档名称和链接（或描述来源）
- 文档主题和大致页数
- 你最想弄明白的部分（可选）

我会评估文档质量和适用性，优先处理 Star 数和呼声较高的请求。

---

## 更新计划

> 以下为已规划或在考虑中的方向。Star 本仓库可以第一时间收到更新通知。

### AI & LLM

| 主题 | 源材料 | 作者 | 状态 |
|------|--------|------|:----:|
| Structured Output | 结构化输出技术 | Anthropic / OpenAI | 计划中 |
| 更多论文精读 | arXiv 等学术平台 | — | 持续中 |

### 系统设计与架构

| 主题 | 源材料 | 作者 | 状态 |
|------|--------|------|:----:|
| 数据密集型应用设计 | Designing Data-Intensive Applications | Martin Kleppmann · O'Reilly | 计划中 |
| 软件架构基础 | Fundamentals of Software Architecture | Mark Richards & Neal Ford · O'Reilly | 计划中 |
| 软件架构难题 | Software Architecture: The Hard Parts | Neal Ford, Mark Richards et al. · O'Reilly | 计划中 |

### 编程语言

| 主题 | 源材料 | 作者 | 状态 |
|------|--------|------|:----:|
| Rust 系统编程 | Programming Rust, 3rd Edition (2024) | Jim Blandy & Jason Orendorff · O'Reilly | 计划中 |
| 流畅的 Python | Fluent Python, 2nd Edition | Luciano Ramalho · O'Reilly | 计划中 |
| Go 语言 | The Go Programming Language | Donovan & Kernighan | 计划中 |

### 基础设施与 DevOps

| 主题 | 源材料 | 作者 | 状态 |
|------|--------|------|:----:|
| Kubernetes | Kubernetes: Up and Running, 3rd Edition | Burns, Beda & Hightower · O'Reilly | 计划中 |
| 云原生 DevOps | Cloud Native DevOps with Kubernetes, 2nd Edition | Arundel & Domingus · O'Reilly | 计划中 |

### 数据工程

| 主题 | 源材料 | 作者 | 状态 |
|------|--------|------|:----:|
| 数据工程基础 | Fundamentals of Data Engineering | Joe Reis & Matt Housley · O'Reilly | 计划中 |
| 数据工程设计模式 | Data Engineering Design Patterns | O'Reilly | 计划中 |

---

如果这些笔记对你有帮助，欢迎 **Star** 支持，这是我持续整理分享的最大动力。

---

## 关于作者

**Ethan** — 一个喜欢把复杂文档拆明白的开发者。

这些笔记源于我自己的学习过程。我相信**最好的学习方式是把知识教给别人**——整理这些笔记既是为了加深自己的理解，也希望能帮到同样在这条路上的你。

有问题或建议？欢迎开 [Issue](https://github.com/191341025/Pandora-Codex-Notes/issues) 交流。

---

## License

本仓库内容采用 [CC BY-NC-SA 4.0](LICENSE) 许可协议。

简单说：**可以看、可以转、可以改，但要署名、不能卖钱、改了也要开放。**

- 转载请注明来源和作者
- 不可用于付费课程、付费专栏等商业用途
- 改编作品需以相同协议分享

详见 [LICENSE](LICENSE) 文件。
