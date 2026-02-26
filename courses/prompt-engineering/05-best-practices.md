# 模块五：最佳实践与总结

> 对应 PDF 第 54-67 页

---

## 概念地图

- **核心概念** (必须内化): Instructions vs Constraints 原则、结构化输出（JSON/Schema）策略
- **实操要点** (动手时需要): Prompt 文档化模板、变量化 prompt、JSON Repair、输出格式实验
- **背景知识** (扩展理解): 多模态 prompting 概述、跨模型适配策略

---

## 概念讲解

### 前言

前面四个模块讲的是"有哪些技术可以用"。本模块讲的是"不管用什么技术，都应该遵守的实操原则"——一套经过实践验证的最佳实践清单。

### 1. 提供示例（Provide Examples）

**最重要的一条最佳实践，没有之一。**

Few-shot 示例是最有效的 prompt 增强手段，因为它直接给模型一个"瞄准的靶子"——你想要什么样的输出格式、什么样的语气、什么样的内容深度，一个示例胜过千言万语的描述。

> 这个原则和 Module 02-1 的 few-shot prompting 完全一致。如果你在那个模块已经内化了 few-shot 的设计要点，这里只是再次强调它在所有场景中的普适性。

### 2. 简洁设计（Design with Simplicity）

Prompt 应该**简洁、清晰、直接**。如果你自己都觉得 prompt 读起来绕，模型大概率也会被绕晕。

**反面示例**：

```
❌ BEFORE:
I am visiting New York right now, and I'd like to hear more
about great locations. I am with two 3 year old kids. Where
should we go during our vacation?
```

**正面示例**：

```
✅ AFTER:
Act as a travel guide for tourists. Describe great places
to visit in New York Manhattan with a 3 year old.
```

改进了什么：去掉了无关信息（"I am visiting right now"、"our vacation"），直接说清楚角色（travel guide）、地点（Manhattan）、约束条件（with a 3 year old）。

**动作动词清单**——用明确的动词开头，让模型知道要做什么：

> Act, Analyze, Categorize, Classify, Contrast, Compare, Create, Describe, Define, Evaluate, Extract, Find, Generate, Identify, List, Measure, Organize, Parse, Pick, Predict, Provide, Rank, Recommend, Return, Retrieve, Rewrite, Select, Show, Sort, Summarize, Translate, Write

### 3. 明确输出要求（Be Specific about the Output）

模糊的指令得到模糊的结果。你越具体，模型越知道往哪使劲。

| 做法 | 示例 |
|------|------|
| ✅ 具体 | `Generate a 3 paragraph blog post about the top 5 video game consoles. The blog post should be informative and engaging, and it should be written in a conversational style.` |
| ❌ 模糊 | `Generate a blog post about video game consoles.` |

具体在哪：段落数（3段）、话题范围（top 5）、风格要求（informative, engaging, conversational）。

### 4. 用指令代替约束（Instructions over Constraints）

这是一个反直觉但很重要的原则：**告诉模型"要做什么"比告诉它"不要做什么"更有效**。

- **Instruction（指令）**：明确告诉模型期望的格式、风格、内容
- **Constraint（约束）**：设定模型不应该做的事

**为什么指令优于约束**：
1. 指令直接传达目标，约束让模型猜什么是被允许的
2. 指令给模型创造空间，约束限制模型的可能性
3. 多个约束之间可能**互相矛盾**，导致模型无所适从

| 做法 | 示例 |
|------|------|
| ✅ 指令优先 | `Generate a 1 paragraph blog post about the top 5 video game consoles. Only discuss the console, the company who made it, the year, and total sales.` |
| ❌ 约束堆积 | `Generate a 1 paragraph blog post about the top 5 video game consoles. Do not list video game names.` |

指令版明确说了"讨论什么"，模型知道要关注什么；约束版只说了"不讨论什么"，模型可能在其他方面也跑偏。

**实操策略**：
1. **先写指令**——把你想要的直接说清楚
2. **约束作为补充**——只在安全（防止有害内容）或严格格式要求时使用
3. **用正面表述替代否定**：`Do not use jargon` → `Use simple, everyday language`

### 5. 控制输出长度（Control Max Token Length）

两种方式配合使用：

| 方式 | 示例 | 特点 |
|------|------|------|
| 模型配置 | `max_tokens=100` | 硬截断，到数就停 |
| Prompt 指令 | `"Explain quantum physics in a tweet length message."` | 软引导，模型主动控制篇幅 |

最佳实践：**两者结合**——prompt 中描述期望长度，配置中设一个合理的 token 上限作为安全网。

### 6. 使用变量化 Prompt（Use Variables in Prompts）

当你的 prompt 要在不同输入上复用时，**把可变部分抽成变量**，不要硬编码。

```
Prompt:
VARIABLES
{city} = "Amsterdam"

PROMPT
You are a travel guide. Tell me a fact about the city: {city}

Output:
Amsterdam is a beautiful city full of canals, bridges, and
narrow streets...
```

**为什么在乎**：
- 同一个 prompt 模板可以服务多个输入（换 `{city}` 就行）
- 集成到应用程序中时，变量是参数化的基础
- 避免重复劳动——信息只定义一次，多处引用

### 7. 实验输入格式和写作风格

同一个目标，不同的 prompt 格式会得到不同结果：

| 格式 | 示例 | 输出特点 |
|------|------|----------|
| **提问式** | `What was the Sega Dreamcast and why was it revolutionary?` | 更像回答问题，可能比较简短 |
| **陈述式** | `The Sega Dreamcast was a sixth-generation console released in 1999. It...` | 模型会"接着写"，延续你的风格 |
| **指令式** | `Write a single paragraph that describes the Sega Dreamcast and explains why it was revolutionary.` | 最可控，明确了格式（one paragraph）和内容 |

**最佳实践**：对同一任务尝试不同格式，选效果最好的。没有万能格式——取决于模型、任务和你的需求。

### 8. Few-shot 分类任务：打乱类别顺序

做分类任务的 few-shot 时，**示例中的类别要打乱顺序**，不要所有正面示例放一起、所有负面示例放一起。

**为什么**：如果示例按类别排序（先 3 个正面，再 3 个负面），模型可能学到的是**"最后几个示例的类别"**而不是**"如何判断类别"**。打乱顺序迫使模型关注内容特征，而不是位置模式。

建议起步：**6 个 few-shot 示例**，类别交替出现，然后测试准确率。

### 9. 适配模型更新

模型会迭代更新（新版本、新数据、新能力）。**你的 prompt 可能在新版本上表现不同**。

最佳实践：
- 关注模型变更日志
- 在新版本上重新测试已有 prompt
- 利用新特性优化 prompt（而不是永远用旧的写法）
- 用 Vertex AI Studio 等工具保存和版本化 prompt

### 10. 实验输出格式与结构化输出

对于非创意任务（提取、分类、排序、解析），优先让输出返回 **JSON 或 XML** 等结构化格式。

**JSON 输出的六大优势**：

| 优势 | 说明 |
|------|------|
| 格式一致 | 每次输出结构相同，便于程序处理 |
| 聚焦数据 | 模型只输出你要的字段 |
| 减少幻觉 | 结构约束让模型不容易"自由发挥" |
| 关系感知 | JSON 天然支持嵌套和引用 |
| 数据类型 | 数字、字符串、布尔值有明确类型 |
| 可排序 | datetime 等字段可以预排序 |

### 11. JSON Repair

JSON 输出虽好，但有一个现实问题：**JSON 比纯文本消耗更多 token**（大括号、引号、键名都是 token）。如果输出窗口不够大，JSON 可能在中途被截断——少了一个 `}` 就变成了无效 JSON。

**解决方案**：使用 `json-repair` 库（PyPI 可安装）自动修复不完整的 JSON。

```bash
pip install json-repair
```

这个库能智能地补全缺失的括号、引号，让截断的 JSON 变得可用。在生产环境中处理 LLM 输出时，这是一个必备的防御层。

### 12. 使用 Schema 控制输入

JSON 不仅可以约束输出，还可以**结构化输入**。通过提供 JSON Schema，给模型一个清晰的数据蓝图：

```json
{
  "type": "object",
  "properties": {
    "name": { "type": "string", "description": "Product name" },
    "category": { "type": "string", "description": "Product category" },
    "price": { "type": "number", "format": "float" },
    "features": {
      "type": "array",
      "items": { "type": "string" }
    },
    "release_date": { "type": "string", "format": "date" }
  }
}
```

然后提供符合 schema 的数据：

```json
{
  "name": "Wireless Headphones",
  "category": "Electronics",
  "price": 99.99,
  "features": ["Noise cancellation", "Bluetooth 5.0", "20-hour battery life"],
  "release_date": "2023-10-27"
}
```

**好处**：
- 模型清楚知道每个字段的含义和类型
- 大批量数据处理时，schema 让模型的注意力聚焦在相关字段上
- 包含日期字段让模型具备"时间感知"

### 13. 与其他 Prompt 工程师协作

如果你需要为某个任务找到最佳 prompt，不要一个人闷头试——**找多个人同时尝试**。每个人遵循同样的最佳实践，但不同的人写出的 prompt 往往有性能差异。通过对比多人的尝试，你更容易找到最优方案。

### 14. CoT 专项最佳实践

从 Module 03 延伸的实操要点：

| 原则 | 说明 |
|------|------|
| **答案放在推理之后** | 因为推理步骤的 token 会影响最终答案的预测——先推理再回答，和先回答再解释，效果完全不同 |
| **能提取最终答案** | CoT 和 Self-consistency 都需要从输出中程序化地提取答案，所以让答案和推理分离（比如 "The answer is X"） |
| **Temperature 设为 0** | CoT 基于贪心解码，推理任务通常只有一个正确答案 |

### 15. 文档化 Prompt 迭代（Document Prompt Attempts）

**最容易被忽视但最有长期价值的实践**。

Prompt 输出会因为模型版本、采样参数、甚至同一 prompt 的不同次运行而变化。如果不记录，你会：
- 忘记之前什么 prompt 管用
- 无法复现好的结果
- 在模型更新后不知道哪些 prompt 需要重新测试

**推荐的文档模板**：

| 字段 | 内容 |
|------|------|
| Name | prompt 名称和版本号 |
| Goal | 一句话描述目标 |
| Model | 模型名和版本 |
| Temperature | 值 |
| Token Limit | 值 |
| Top-K | 值 |
| Top-P | 值 |
| Prompt | 完整 prompt 文本 |
| Output | 完整输出（可以有多个） |

**进阶字段**（推荐）：
- **Iteration** — 第几次迭代
- **Result** — OK / NOT OK / SOMETIMES OK
- **Feedback** — 需要改进什么
- **RAG 信息**（如适用）— query、chunk 设置、chunk 输出

**Prompt 管理的生命周期**：

```
实验阶段：用文档模板记录每次尝试
    ↓ 确认效果后
代码阶段：prompt 存为独立文件（和代码分离，便于维护）
    ↓ 部署后
运维阶段：自动化测试 + 评估流程，监控 prompt 在生产中的表现
```

### 16. 多模态 Prompting（简述）

多模态 prompting 是指用**多种输入格式**（文本 + 图片 + 音频 + 代码）来引导模型。这和本白皮书讲的文本 prompting 是不同的领域，但同样的核心原则（清晰、具体、提供示例）仍然适用。

### 17. 全文总结

本白皮书覆盖的 prompting 技术全景：

```
基础层
├── Zero-shot         — 无示例，直接提问
├── One/Few-shot      — 提供示例，展示模式
├── System prompting  — 定义任务和规则
├── Role prompting    — 分配角色身份
└── Contextual prompting — 提供背景信息

推理层
├── Step-back         — 先退后进
├── Chain of Thought  — 逐步推理
├── Self-consistency  — 多次推理投票
├── Tree of Thoughts  — 树状探索
└── ReAct             — 推理 + 行动

元技术
└── Automatic Prompt Engineering — 用 LLM 优化 prompt

应用层
└── Code Prompting    — 写/解释/翻译/调试代码
```

**核心心法**：Prompt engineering 是一个**迭代过程**。写 prompt → 测试 → 分析结果 → 改进 → 再测试。没有一次写对的完美 prompt，只有持续优化的过程。换模型、换版本、换配置，都需要重新实验。

---

## 重点标记

1. **Instructions > Constraints**：告诉模型"做什么"比告诉它"不做什么"更有效。约束只用于安全和严格格式场景。
2. **JSON 是非创意任务的最佳输出格式**：格式一致、减少幻觉、可程序化处理。配合 json-repair 应对截断问题。
3. **Schema 可以同时约束输入和输出**：给模型数据蓝图，让它聚焦于相关字段。
4. **文档化 prompt 迭代是长期投资**：用结构化模板记录每次尝试，是你在模型更新、团队协作、生产维护中的救命稻草。
5. **Few-shot 分类要打乱类别顺序**：避免模型学到位置模式而非内容特征。
6. **CoT 的答案必须放在推理之后**：推理步骤改变了预测概率，先推理后回答和先回答后解释效果完全不同。
7. **Prompt 和代码要分离存放**：便于独立维护和版本化。

---

## 自测：你真的理解了吗？

**Q1**：你写了一个 prompt：`Generate a blog post about AI. Do not mention ChatGPT. Do not discuss ethics. Do not use technical jargon. Keep it under 500 words.` 这个 prompt 有什么问题？你会怎么改写？

**Q2**：你的 LLM 应用返回 JSON 格式的商品推荐列表，但偶尔输出的 JSON 缺少最后的 `]}`，导致应用崩溃。你有哪些策略来解决这个问题？

**Q3**：你做了一个情感分类的 few-shot prompt，6 个示例按顺序是：正面、正面、正面、负面、负面、负面。模型对新输入总是倾向于分类为负面。可能的原因是什么？怎么改？

**Q4**：你在一月份用 Gemini 1.0 测试的 prompt 效果很好，三个月后换成 Gemini 1.5 效果变差了。你应该怎么做？如果你之前没有文档化 prompt，会面临什么困难？

**Q5**：同事争论"prompt 就写在代码里就行了，没必要单独存"。你怎么反驳？
