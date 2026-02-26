# 模块二-2：System / Role / Contextual Prompting

> 对应 PDF 第 18-24 页

---

## 概念地图

- **核心概念** (必须内化): System / Role / Contextual Prompting 三者的区别与组合使用
- **实操要点** (动手时需要): 如何写有效的 System Prompt、如何设定角色风格
- **背景知识** (扩展理解): 三种 prompting 的重叠区域

---

## 概念讲解

### 总览：三种框架类技术

上一模块讲的 zero-shot / few-shot 是"要不要给示例"的问题。本模块讲的 System / Role / Contextual prompting 是另一个维度的问题——**怎么给模型设定"身份"和"背景"**。

这三种技术经常一起使用，但各自有明确的侧重点：

| 类型 | 核心问题 | 比喻 |
|------|----------|------|
| **System Prompting** | 模型要**做什么** | 给员工下达工作职责 |
| **Role Prompting** | 模型要**像谁** | 让员工扮演特定角色 |
| **Contextual Prompting** | 模型要**知道什么** | 给员工提供项目背景资料 |

三者经常**重叠**——一个好的 prompt 可以同时设定系统任务、分配角色、提供上下文。分开讲是为了让你理解每种技术在做什么，实际使用时灵活组合。

### 1. System Prompting（系统提示）

System prompting 是给模型定义**总体任务和行为规则**——"你是什么系统，你的工作是什么，输出应该是什么格式"。

**为什么在乎**：System prompt 是你对模型行为的"顶层指令"。没有 system prompt，模型就是个通用聊天助手；有了 system prompt，它就变成一个有明确职责的专用工具。

**用起来什么样**：

**示例 1：控制输出格式**

```
Prompt:
Classify movie reviews as positive, neutral or negative.
Only return the label in uppercase.

Review: "Her" is a disturbing study revealing the direction
humanity is headed if AI is allowed to keep evolving,
unchecked. It's so disturbing I couldn't watch it.
Sentiment:

Output: NEGATIVE
```

关键点：`Only return the label in uppercase` 这句 system 级指令让模型只返回标签，不输出多余文字。即使 temperature 设为 1（高创意），模型也遵守了格式要求。

**示例 2：结构化 JSON 输出**

```
Prompt:
Classify movie reviews as positive, neutral or negative.
Return valid JSON:

Review: "Her" is a disturbing study revealing the direction
humanity is headed if AI is allowed to keep evolving,
unchecked. It's so disturbing I couldn't watch it.

Schema:
MOVIE:
{
  "sentiment": String "POSITIVE" | "NEGATIVE" | "NEUTRAL",
  "name": String
}
MOVIE REVIEWS:
{
  "movie_reviews": [MOVIE]
}
JSON Response:

Output:
{
  "movie_reviews": [
    {
      "sentiment": "NEGATIVE",
      "name": "Her"
    }
  ]
}
```

这个例子展示了 system prompting 的一个重要应用：**用 JSON schema 约束输出格式**。通过给出明确的 schema 定义，模型被迫按结构化方式输出，好处有三：
1. 输出可以直接被程序解析
2. 数据天然有序（datetime 等字段可以预排序）
3. **强制结构化可以减少幻觉**——模型需要填充具体字段，而不是自由发挥

**System prompting 的另一个重要用途：安全控制**。在 prompt 中加一句 `You should be respectful in your answer.` 就能控制输出的安全性和礼貌程度。

### 2. Role Prompting（角色提示）

Role prompting 是给模型**分配一个角色身份**——"你是旅行顾问 / 编辑 / 幼儿园老师 / 励志演讲家"。模型会根据角色调整自己的语气、用词、专业深度和关注点。

**为什么在乎**：同样的问题，不同角色给出的回答截然不同。让模型扮演"地理老师"回答旅游问题，和扮演"旅行达人"回答，得到的内容重点、语气、专业深度完全不一样。角色相当于给模型一个"输出人格的蓝图"。

**用起来什么样**：

**示例 1：旅行顾问（正经风格）**

```
Prompt:
I want you to act as a travel guide. I will write to you
about my location and you will suggest 3 places to visit
near me. In some cases, I will also give you the type of
places I will visit.

My suggestion: "I am in Amsterdam and I want to visit
only museums."
Travel Suggestions:

Output:
1. Rijksmuseum: World-renowned museum, known for Dutch
   masterpieces like Rembrandt's "The Night Watch."...
2. Van Gogh Museum: Immerse yourself in the captivating
   works of Vincent van Gogh...
3. Stedelijk Museum Amsterdam: Discover modern and
   contemporary art from around the world...
```

**示例 2：同一角色 + 幽默风格**

```
Prompt:
I want you to act as a travel guide. I will write to you
about my location and you will suggest 3 places to visit
near me in a humorous style.

My suggestion: "I am in Manhattan."
Travel Suggestions:

Output:
1. Behold the Empire State of Mind: Ascend to the dizzying
   heights of the Empire State Building... Prepare to feel
   like King Kong atop the Big Apple, minus the giant
   ape-sized banana.
2. Get Artsy-Fartsy at MoMA: Unleash your inner art
   aficionado...
3. Shop 'Til You Drop on Fifth Avenue: Indulge in some
   retail therapy... that will make your wallet cry.
```

**风格选择**：PDF 列出了一些有效的角色风格参考：

> Confrontational, Descriptive, Direct, Formal, Humorous, Influential, Informal, Inspirational, Persuasive

你可以组合使用：`act as a travel guide in a humorous and inspirational style`。

**什么时候用**：
- 需要特定领域的专业视角（医生、律师、工程师）
- 需要特定语气和风格（正式、幽默、鼓励性）
- 需要输出一致的"人格"特征（比如客服机器人要始终友善）

### 3. Contextual Prompting（上下文提示）

Contextual prompting 是给模型提供**和当前任务直接相关的背景信息**——"你在为一个 80 年代复古游戏博客写文章"、"用户是一个 3 岁小孩的家长"。

**为什么在乎**：没有上下文，模型给的是通用回答；有了上下文，回答变得精准、有针对性。上下文信息是**动态的**——同一个角色、同一个系统指令，搭配不同的上下文会产出完全不同的内容。

**用起来什么样**：

```
Prompt:
Context: You are writing for a blog about retro 80's
arcade video games.

Suggest 3 topics to write an article about with a few
lines of description of what this article should contain.

Output:
1. The Evolution of Arcade Cabinet Design:
   This article would explore the evolution of arcade
   cabinet designs, from the early wood and metal cabinets
   of the 1970s...
2. Blast From The Past: Iconic Arcade Games of The 80's:
   ...history, impact, and legacy of some of the most
   iconic and influential arcade games...
3. The Rise and Retro Revival of Pixel Art:
   ...evolution of pixel art as a defining visual style
   of early arcade games...
```

如果去掉上下文 `You are writing for a blog about retro 80's arcade video games`，模型可能给出关于任何类型博客的建议。上下文把模型的"注意力"聚焦到了特定领域。

### 三者的组合使用

实际工作中，这三种技术很少单独使用。一个完整的 prompt 通常包含：

```
[System] 你是一个客服助手，负责回答产品问题。只回答和产品相关的问题，
        拒绝无关话题。

[Role]   以专业但友善的语气回答，像一个有耐心的技术支持工程师。

[Context] 用户正在使用我们的 v3.2 版本产品，运行在 Windows 11 上。
         已知问题列表：[列表]

[User Query] 我的软件启动后黑屏怎么办？
```

**组合要点**：
- System 定义"能做什么、不能做什么"（边界）
- Role 定义"用什么方式做"（风格）
- Context 定义"基于什么信息做"（知识）

三者各管一面，互不矛盾时效果最佳。

---

## 重点标记

1. **System Prompting 管"做什么"**：定义任务职责、输出格式、安全边界。JSON schema 是一个强大的 system prompting 技巧。
2. **Role Prompting 管"像谁"**：分配角色身份 + 输出风格。同一任务换不同角色，输出的视角和深度完全不同。
3. **Contextual Prompting 管"知道什么"**：提供任务相关的背景信息，让通用模型变成领域专家。
4. **三者经常组合使用**：System 划边界，Role 定风格，Context 给知识——各管一面，互不矛盾。
5. **结构化输出（JSON）可以减少幻觉**：强制模型按 schema 填充字段，比让它自由发挥更可靠。

---

## 自测：你真的理解了吗？

**Q1**：你要做一个面向老年人的健康咨询聊天机器人。请分别写出 System prompt、Role prompt 和 Context prompt 各一句话。

**Q2**：你在 system prompt 里写了"只回答中文"，在 role prompt 里写了"你是一个英国绅士"。模型很可能怎么处理这个矛盾？你应该怎么避免？

**Q3**：你的 prompt 只有 `Tell me about Amsterdam`（零上下文）。列出三种不同的 contextual prompt 前缀，它们会让模型分别从什么角度回答？

**Q4**：为什么说"用 JSON schema 约束输出可以减少幻觉"？背后的机制是什么？
