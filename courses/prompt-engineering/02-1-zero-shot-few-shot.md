# 模块二-1：Zero-shot 与 Few-shot Prompting

> 对应 PDF 第 13-17 页

---

## 概念地图

- **核心概念** (必须内化): Zero-shot、Few-shot 的本质区别与适用场景
- **实操要点** (动手时需要): Few-shot 示例设计技巧、边缘案例处理
- **背景知识** (扩展理解): One-shot 作为 Few-shot 的特例

---

## 概念讲解

### 1. Zero-shot Prompting（零样本提示）

**一句话说清**：直接给任务描述，不给任何示例，让 LLM 自己搞定。

这是最简单的 prompt 形式——你只需要告诉模型"做什么"，不需要展示"怎么做"。叫 "zero-shot" 就是因为**零个示例**。

**用起来什么样**：

| 配置 | 值 |
|------|-----|
| Goal | Classify movie reviews as positive, neutral or negative |
| Model | gemini-pro |
| Temperature | 0.1 |
| Token Limit | 5 |
| Top-K | N/A |
| Top-P | 1 |

```
Prompt:
Classify movie reviews as POSITIVE, NEUTRAL or NEGATIVE.
Review: "Her" is a disturbing study revealing the direction
humanity is headed if AI is allowed to keep evolving,
unchecked. I wish there were more movies like this masterpiece.
Sentiment:

Output: POSITIVE
```

注意这个例子的巧妙之处：评论中同时出现了 "disturbing"（负面词）和 "masterpiece"（正面词），模型需要理解整体语义才能正确分类。Temperature 设为 0.1（低随机性），因为分类任务不需要创意，需要确定性。

**什么时候用**：
- 任务本身很明确，模型凭训练数据就能理解（比如翻译、情感分类、摘要）
- 你不关心输出格式的精确控制
- 快速原型验证——先试 zero-shot，不行再加示例

**什么时候不够用**：
- 输出需要特定格式（JSON、表格、特定字段）
- 任务有领域特殊性，模型可能误解你的意图
- 需要处理边缘案例

### 2. One-shot Prompting（单样本提示）

给模型**一个示例**来展示你期望的输入→输出模式。模型看到这个示例后，会模仿其模式来处理新的输入。

One-shot 是 Few-shot 的特例（只有一个示例），单独提出来是因为它代表了一个重要的思维跳跃：**从"描述任务"到"展示任务"**。

**什么时候一个示例就够**：
- 输出格式相对简单
- 任务模式很直观，一个例子就能看懂
- Token 预算有限，没空间放多个示例

### 3. Few-shot Prompting（少样本提示）

给模型**多个示例**来展示任务模式。和 one-shot 同样的道理，但多个示例让模型更稳定地捕捉到你想要的模式。

**直觉建立**：

想象你在培训一个新员工。Zero-shot 就是只给一份任务说明书——"把客户订单整理成表格"。聪明的员工可能直接搞定，但格式可能和你预期不一样。One-shot 就是你做了一条给他看——"看，就这么整理"。Few-shot 就是你做了三五条给他看，还特意挑了几个复杂的情况——"普通订单这么写，半半披萨这么写，特殊要求这么写"。

> **类比边界**：真实的新员工会"学会"技能并记住，但 LLM 的 few-shot "学习"是临时的——下次对话又从零开始，除非你再次提供示例。

**用起来什么样**：

以下示例展示了用 few-shot 让模型把披萨订单解析为 JSON：

```
Prompt:
Parse a customer's pizza order into valid JSON:

EXAMPLE:
I want a small pizza with cheese, tomato sauce, and pepperoni.
JSON Response:
{
  "size": "small",
  "type": "normal",
  "ingredients": [["cheese", "tomato sauce", "pepperoni"]]
}

EXAMPLE:
Can I get a large pizza with tomato sauce, basil and mozzarella
{
  "size": "large",
  "type": "normal",
  "ingredients": [["tomato sauce", "basil", "mozzarella"]]
}

Now, I would like a large pizza, with the first half cheese
and mozzarella. And the other tomato sauce, ham and pineapple.
JSON Response:

Output:
{
  "size": "large",
  "type": "half-half",
  "ingredients": [["cheese", "mozzarella"], ["tomato sauce", "ham", "pineapple"]]
}
```

关键观察：
- 两个示例建立了 JSON 结构的模式（size、type、ingredients 字段）
- 实际输入引入了一个**边缘案例**——"半半披萨"（一半一种配料，另一半另一种）
- 模型正确推断出 `"type": "half-half"` 和嵌套数组结构，即使示例中没有直接展示过这种情况

### 4. Few-shot 示例设计要点

示例不是随便给几个就行。设计好的示例直接决定 few-shot 的效果。

**数量**：一般 3-5 个示例是个好起点。太少可能不够模型学习模式，太多可能超出 token 限制。复杂任务可能需要更多。

**质量四原则**：

| 原则 | 说明 | 反例 |
|------|------|------|
| **相关性** | 示例必须和实际任务是同一类型 | 用翻译示例教分类——答非所问 |
| **多样性** | 覆盖不同的输入变体 | 5 个示例全是相同模式——模型只学会了一种情况 |
| **高质量** | 示例本身必须正确 | 示例中有拼写错误或格式不一致——模型会模仿错误 |
| **包含边缘案例** | 特意展示不寻常但合法的输入 | 只有"正常"示例——遇到异常输入时模型不知所措 |

> **常见误用**：示例中有一个小错误（比如 JSON 少了一个逗号），你可能觉得无所谓，但模型会忠实地"学习"这个错误并在输出中复制。**一个错误示例的破坏力远大于你的想象**。

### Zero-shot vs One-shot vs Few-shot 对比

| 维度 | Zero-shot | One-shot | Few-shot |
|------|-----------|----------|----------|
| 示例数 | 0 | 1 | 2+ |
| Token 消耗 | 最少 | 中等 | 最多 |
| 格式控制力 | 弱 | 中等 | 强 |
| 适用场景 | 简单明确的任务 | 格式简单的任务 | 复杂/格式严格的任务 |
| 迭代成本 | 低（改 prompt 快） | 中 | 高（改示例耗时） |
| 推荐策略 | 先试 zero-shot | — | zero-shot 不够再升级 |

---

## 重点标记

1. **Zero-shot 是起点**：任何任务先用 zero-shot 试一试，不行了再加示例升级为 few-shot。
2. **Few-shot 的核心价值是"展示而非描述"**：与其用文字解释输出格式，不如直接给一个正确的示例。
3. **示例质量 > 示例数量**：3 个精心设计的示例胜过 10 个随意堆砌的示例。一个有错误的示例可能毁掉整个 prompt。
4. **边缘案例是关键**：Few-shot 示例要特意包含异常/特殊输入，帮助模型应对实际场景中的多样性。
5. **记录你的 prompt 迭代**：Prompt engineering 是迭代过程，用结构化表格（Goal/Model/Config/Prompt/Output）记录每次尝试，方便对比和复现。

---

## 自测：你真的理解了吗？

**Q1**：你需要让模型把自然语言日期（"下周三"、"后天"、"3月15号"）转成 ISO 格式（2025-03-15）。你会用 zero-shot 还是 few-shot？为什么？

**Q2**：你设计了一个 few-shot prompt，给了 5 个示例，全是正面情感分类。模型在处理负面评论时表现很差。可能的原因是什么？怎么改？

**Q3**：同事说"few-shot 示例越多效果越好"。这句话有什么问题？

**Q4**：你的 few-shot 示例中有一个 JSON 输出格式是 `{"name": "Alice",}` （末尾多了一个逗号）。模型很可能会怎么做？这说明了什么原则？
