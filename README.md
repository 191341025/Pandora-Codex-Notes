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

| 课程 | 主题 | 源材料 | 模块数 | 状态 |
|------|------|--------|:------:|:----:|
| [Prompt Engineering](courses/prompt-engineering/) | 提示工程基础到高级技术 | Google Prompt Engineering v7 (68p) | 5 | ✅ |
| [Claude Tool Use](courses/claude-tool-use/) | Anthropic Tool Use API 完整指南 | Anthropic 官方文档 (18篇) | 5 | ✅ |
| [Claude Code Guide](courses/claude-code-guide/) | Claude Code 钩子、扩展与专业模式 | Claude Code Complete Guide (289p) | 6 | ✅ |
| [Building Skills for Claude](courses/building-skills-for-claude/) | Claude 技能从设计到分发 | Building Skills Guide (33p) | 5 | ✅ |
| [Claude Cowork](courses/claude-cowork/) | Anthropic Cowork 协作平台 | Anthropic Cowork Documentation (17p) | 5 | ✅ |

> 更多课程持续整理中。关注本仓库以获取更新。

## 论文精读

| 论文 | 领域 | 关键词 |
|------|------|--------|
| [CLEF HIPE-2026](papers/2026-02-hipe2026/digest.md) | NLP / 历史文本分析 | 多语言关系抽取、知识图谱、OCR |

论文精读采用**七板块结构**：一句话总结 → 为什么值得关注 → 核心贡献 → 方法直觉 → 关键发现 → 局限与开放问题 → 启发与联想。

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

- [ ] AI Agent 设计模式课程（21 种模式深度拆解）
- [ ] OpenAI Agents SDK 开发指南
- [ ] Anthropic 官方 Agent 构建白皮书
- [ ] 向量数据库与 RAG 实现
- [ ] Structured Output 课程
- [ ] 更多论文精读

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
