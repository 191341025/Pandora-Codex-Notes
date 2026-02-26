# 模块二：Embeddings 嵌入技术详解

> 对应 PDF: 02_Embeddings.pdf，第 1-4 页

---

## 概念地图

- **核心概念** (必须内化): Word2Vec 的范式转变（从离散符号到连续向量空间）、静态嵌入 vs 动态嵌入的本质区别、专用嵌入模型在生产系统中的角色
- **实操要点** (动手时需要): Sentence Transformers 的 6 条最佳实践（批处理/长度处理/归一化/GPU/内存/错误处理）、ChromaDB + SentenceTransformer 的 RAG Pipeline 实现、模型选型（all-MiniLM-L6-v2 vs all-mpnet-base-v2）
- **背景知识** (扩展理解): 稀疏嵌入 vs 密集嵌入的历史演进、三种 Transformer 架构处理嵌入的差异、零样本学习的嵌入基础

---

## 概念讲解

### 1. 向量嵌入的本质与必要性（2.1）

**定义**：向量嵌入（Vector Embedding）就是把非结构化数据（文本、图片、音频、视频）映射成一组有序数字列表（向量），让机器能"理解"这些数据之间的含义和关系。

**核心思想**：传统数据结构对数字、日期、分类值处理得很好，但一碰到人类自然产生的丰富内容——文本、图片、音频、视频——就力不从心了。嵌入就是连接"人类表达"和"机器计算"的桥梁：把原始数据转换成数学向量，而这些向量能保留数据内在的语义和关系。

**直觉建立**：想象你是一个调香师，面前有几百种香水。每种香水的"味道"是一种非结构化的感官体验——你没法用一个数字来描述它。但调香师会用一组维度来"编码"每种香水：花香浓度 0.8、木质调 0.3、甜度 0.6、清新度 0.2……这组数字就是香水的"嵌入向量"。有了这组数字，你就可以做数学运算了：计算两种香水的相似度、找到和某种香水最像的其他香水、甚至做"减去甜味加上烟熏味"的向量算术来探索新配方。向量嵌入对文本、图片做的事情完全一样——把人类感知的丰富内容"翻译"成一组数字，让机器能够计算"相似度"和"关系"。关键区别在于：调香师的维度是人工定义的，而向量嵌入的维度是模型自动从数据中学到的。

**为什么重要/有效**：

1. **语义搜索（Semantic Search）**：不再局限于精确关键词匹配或死板的分类过滤。搜"sunset by the ocean"能找到标签完全不同的海边落日图片；搜"renewable energy solutions"能找到讨论太阳能和风能的文档，即使没有精确匹配的短语。
2. **LLM 和生成式 AI 的基石**：当 GPT-4 写诗或 Claude 分析法律文件时，底层都是在向量上做复杂数学运算。每一层模型都在不断变换这些向量表示。可以说，**现代人工智能建立在向量嵌入的基础之上**。
3. **向量数据库的存在意义**：嵌入产生了大量向量，需要专门的系统来高效存储和查询——这就是向量数据库（Vector Database）。它们提供了内置的相似向量检索能力，是现代 AI 应用的核心基础设施。

**示例**：
- 搜索 "renewable energy solutions" → 匹配到讨论 solar power 和 wind power 的文档（没有精确短语匹配）
- 搜索 "sunset by the ocean" → 找到标签为 "beach evening" 的图片（语义相近，关键词不同）

**适用场景**：
- 任何需要"按意思搜索"而非"按关键词搜索"的场景
- 多媒体内容的跨模态理解（文本描述找图片、语音找文档等）
- 作为所有现代 AI 系统（LLM、推荐系统、RAG）的底层数据表示

---

### 2. Word2Vec：改变一切的突破（2.2）

**定义**：Word2Vec 是 2013 年由 Google 的 Tomas Mikolov 团队提出的词嵌入模型，首次实现了将词语表示为稠密向量（Dense Vector），让向量的几何关系直接反映语义关系。

**核心思想**：用一个浅层神经网络在大规模文本上训练，但真正要的不是预测结果，而是**隐藏层的权重**——这些权重就是词嵌入。核心直觉是：经常出现在相似上下文中的词，会获得相似的向量表示。

**直觉建立**：想象你刚搬到一个陌生城市，不认识任何人。你不知道每个人的职业、性格、爱好，但你可以观察谁和谁经常一起出现：张三总是和律师、法官、法庭出现在同一个场景里；李四总是和医生、护士、手术室出现在一起。即使从没有人告诉你张三是律师、李四是医生，你也能从"共同出现的伙伴"推断出他们的身份，并且知道张三和另一个常和律师一起出现的人"很相似"。Word2Vec 做的事情一模一样——它不需要任何人工标注，只需要观察哪些词经常出现在相同的上下文窗口中。"coffee"和"tea"总是和"morning""cup""drink"一起出现，所以它们获得了相似的向量。这个方法的精妙之处在于：模型训练的目标是"预测上下文"，但真正有价值的产物是训练过程中学到的隐藏层权重——这些权重就是词嵌入。

**范式转变：从 Word2Vec 之前到之后**

| 方面 | Word2Vec 之前 | Word2Vec 之后 |
|------|-------------|-------------|
| 文本表示 | 词袋模型（Bag of Words）、TF-IDF | 稠密向量（Dense Vectors） |
| 词语关系 | "king" 和 "queen" 数学上与 "king" 和 "bicycle" 一样无关 | "king" 和 "queen" 在向量空间中相近 |
| 语义捕获 | 基于手工规则、WordNet 等词库，脆弱且有限 | 从原始文本自动学习，无需人工标注 |
| 向量维度 | 稀疏、维度等于词汇量（可能 10 万+） | 稠密、通常 100-1000 维，每维都有意义 |

**两种训练架构**：

| 架构 | Skip-gram | CBOW (Continuous Bag of Words) |
|------|-----------|------|
| 输入 → 输出 | 一个词 → 预测它的上下文 | 上下文 → 预测中间的词 |
| 直觉 | "给你 coffee 这个词，周围可能出现什么？" | "周围出现了 hot、morning、cup，中间是什么词？" |

**向量算术——最"神奇"的特性**：

```
king - man + woman ≈ queen
```

这说明模型学到了真实的语义和句法关系（性别、时态、国别等），不是简单的统计巧合。

**训练机制简述**：
- 训练一个浅层神经网络（不是深层的）
- 在大量文本上训练，不需要标注数据
- 提取隐藏层的权重作为词向量
- 经常出现在相似上下文中的词（如 "coffee" 和 "tea"）获得相似向量
- 很少出现在相似上下文中的词在向量空间中距离很远
- 计算效率高，可以在海量文本上训练

**后续发展**：

| 模型 | 改进点 |
|------|--------|
| **GloVe** | 显式建模词共现统计（word co-occurrence statistics），而非通过预测任务隐式学习 |
| **fastText** | 引入子词信息（subword information），能处理未见过的词和形态变化 |

**历史意义**：Word2Vec 引发的连锁反应——"学习到的嵌入"概念从词级迅速扩展到句子、段落、整篇文档。**通过预测任务学习上下文表示**这一思路，成为现代语言模型的基石。今天的 Transformer 架构（GPT、BERT）可以看作 Word2Vec 核心洞察的直接后代——只是规模更大、架构更精巧。

> **常见误用**：在需要理解上下文的场景中直接使用 Word2Vec 的静态词向量。Word2Vec 给每个词只有一个固定向量——"bank"不管出现在"river bank"还是"bank account"中都是同一个向量。如果你的应用需要区分多义词（大多数真实场景都需要），应该使用 Transformer 系的动态嵌入或专用嵌入模型。用 Word2Vec 处理多义词会导致搜索结果混淆不同含义，严重降低相关性。

---

### 3. Doc2Vec：从词到文档（2.3）

**定义**：Doc2Vec（原名 Paragraph Vector），由 Mikolov 和 Le 在 2014 年提出，解决了 Word2Vec 的一个根本局限：如何把更长的文本（句子、段落、整篇文档）表示为单一向量。

**核心思想**：在 Word2Vec 的训练过程中，额外引入一个**文档向量（Document Vector）**。这个文档向量就像一个"记忆"，捕获了整篇文档的主题和全局语义。在训练过程中，文档向量被当作上下文中的"另一个词"，但它在同一文档的所有滑动窗口中保持一致，从而同时捕获文档的整体主题和词语的上下文含义。

**直觉建立**：Word2Vec 给每个词一张"名片"，上面写着这个词的语义坐标。但如果你想描述一整篇文章的含义怎么办？最简单的方法是把所有词的名片堆在一起取平均——但这就像把一道菜的所有食材打成糊，你知道大致用了什么材料，却完全丢失了"这道菜是什么"的信息（词序、结构、整体主题全没了）。Doc2Vec 的做法更聪明：它给每篇文档也发了一张专属"名片"。在训练过程中，这张文档名片参与每一个词的预测——就像一个始终在场的"导演"，确保每个词的理解都带着整篇文档的主题色彩。导演（文档向量）在整篇文档的所有场景（滑动窗口）中保持同一个身份，所以它最终学到的就是这篇文档的"整体风格和主题"。

**为什么需要 Doc2Vec（Word2Vec 的局限）**：
- 用 Word2Vec 表示文档，通常是对所有词向量取平均或拼接
- 平均操作会丢失关键信息：**词序**、**文档结构**、**整体含义**
- 简单平均永远无法达到 Doc2Vec 的效果

**Word2Vec vs Doc2Vec 对比**：

| 方面 | Word2Vec | Doc2Vec |
|------|----------|---------|
| 表示粒度 | 词级别 | 文档级别（句子/段落/文档） |
| 输出 | 每个词一个固定向量 | 每篇文档一个固定长度向量 |
| 能否处理歧义 | "bank" 不管上下文都是同一个向量 | 能根据整篇文档上下文区分 "river bank" 和 "bank account" |
| 句子相似度 | "The cat sat on the mat" 和 "A kitten rested on the rug" 共同词少，相似度低 | 能识别两者语义几乎相同 |

**具体例子**：

> Word2Vec 处理 "The bank is by the river"，会挣扎于 "bank" 到底是金融机构还是河岸，因为它通常只是对各词向量做平均或组合。
>
> Doc2Vec 则会学到一个编码了完整上下文的表示，清楚地知道这是一句关于地理的话，而非金融。
>
> 同理，Doc2Vec 能识别 "The movie was fantastic" 和 "The film was excellent" 表达了几乎相同的情感，而简单的词向量平均可能因为选词不同而错过这一点。

**适用场景**：
- **文档分类**（Document Classification）
- **推荐系统**（Recommendation Systems）
- **抄袭检测**（Plagiarism Detection）
- 两篇关于同一事件但用词完全不同的新闻文章 → 获得相似向量
- 两篇话题不同但共享一些常见词的文章 → 向量空间中距离很远

**历史地位**：Doc2Vec 是第一个成功实现"变长文本 → 固定长度向量"的方法，为后来的 Universal Sentence Encoder 和 Transformer 的 attention-based embeddings 铺平了道路。虽然现在更新的方法在性能上已经超越了它，但它对"如何在不同尺度上表示文本"的优雅解法至今仍有影响。

---

### 4. 稀疏嵌入 vs 密集嵌入（Sparse vs Dense Embeddings）

**定义**：这是嵌入技术历史上两种根本不同的数据表示范式。

**直觉建立**：想象两种描述一个人的方式。稀疏嵌入就像一份超长的 Yes/No 清单："是否会弹钢琴？否。是否会做饭？是。是否会开飞机？否。……" 这份清单有 10 万个问题（对应词汇表大小），每个人只在极少数问题上标"是"，其余全是"否"。优点是直观——你一眼就能看出这个人会什么；缺点是极度浪费空间，而且"会做饭"和"会烹饪"被当作完全不同的两个条目，毫无关联。密集嵌入则像一份 300 维的"性格雷达图"：每个维度不是具体的技能，而是某种抽象特质（比如"动手能力""艺术敏感度""逻辑思维"等），每个维度都有一个连续数值。这份雷达图紧凑得多，而且"会做饭"和"会烹饪"的人会得到几乎相同的雷达图——因为它们反映的底层特质是一样的。代价是你很难解释单个维度具体代表什么。

**稀疏嵌入（Sparse Embeddings）**：在深度学习革命之前统治 NLP 的技术，代表有 One-hot Encoding 和 TF-IDF。

> **直观理解**：想象一下用一个 100,000 维的向量表示 "cat" 这个词——整个向量全是 0，只有对应 "cat" 的那个位置是 1。文档则表示为一个稀疏向量，标记哪些词出现了、频率多少。直观但极度低效——绝大部分都是 0，而且维度随词汇量线性增长。

**密集嵌入（Dense Embeddings）**：以 Word2Vec 为代表，将词或文档映射到低得多的维度空间（通常 100-1000 维），每一维都携带有意义的信息。

> **直观理解**：一个 300 维的 Word2Vec 嵌入，不同维度可能编码了性别、数量、正式程度、语义类别等概念——虽然这些维度不一定是人能直接解释的。

**详细对比表**：

| 特性 | 稀疏嵌入（One-hot / TF-IDF） | 密集嵌入（Word2Vec 等） |
|------|---------------------------|---------------------|
| **维度** | 等于词汇量（可能 10 万+） | 通常 100-1000 维 |
| **向量内容** | 绝大部分为 0 | 每个维度都有数值 |
| **可解释性** | 高——每维对应一个具体词/特征 | 低——单个维度不好直接解释 |
| **计算复杂度** | 简单 | 训练需要大量数据和算力 |
| **内存效率** | 低（大量 0 浪费空间） | 高（紧凑、信息密度大） |
| **语义关系** | 完全无法捕获（"king" 和 "queen" 一样无关） | 能捕获丰富的语义关系 |
| **向量算术** | 无意义 | 支持有意义的向量运算 |
| **适合算法** | 传统机器学习（SVM、逻辑回归等） | 深度学习、神经网络 |

**现代实践——两者结合**：

一些搜索引擎同时使用两种方式：
- **密集嵌入**负责语义相似性搜索
- **稀疏嵌入**负责精确关键词匹配
- 两者结合（Hybrid Search），取长补短

> **关键洞察**：虽然神经网络时代密集嵌入已成为主流范式，但稀疏表示在**可解释性**和**精确匹配**至关重要的场景下仍然有用。

---

### 5. Transformer 架构与嵌入的关系（2.4）

**定义**：向量嵌入是所有现代大语言模型（LLM）的基本构件——它是人类语言和机器理解之间的桥梁。每一个进入语言模型的词、句子或 token，都必须先转换成向量表示。

**核心思想**：嵌入不只是输入格式——它们是模型进行所有推理和生成操作的**数学空间**。Word2Vec 和 Doc2Vec 教会我们如何创建有意义的向量表示，Transformer 则教会我们如何**在大规模下动态操作和生成**这些表示。

**三种 Transformer 架构与嵌入的关系**：

| 架构 | 代表模型 | 处理嵌入的方式 | 擅长任务 |
|------|---------|-------------|---------|
| **Encoder-Only**（仅编码器） | BERT（Bidirectional Encoder Representations from Transformers） | 双向处理整个序列，创建**动态、上下文相关的嵌入**。同一个 "bank" 在 "river bank" 和 "bank account" 中获得不同向量。每一层逐步精化嵌入，最终层产出高度上下文化的表示 | 文本分类、命名实体识别、情感分析、问答 |
| **Decoder-Only**（仅解码器） | GPT 系列 | 顺序处理嵌入，每个位置只关注前面的位置（从左到右，像人类写字一样）。嵌入不仅捕获含义，还要包含**预测下一个 token 的信息**。输入嵌入结合了 token + position + segment 信息，最终层将嵌入投影为下一个 token 的概率分布 | 文本续写、创意写作、代码生成、对话 AI |
| **Encoder-Decoder**（编码器-解码器） | T5（Text-to-Text Transfer Transformer）、BART | 编码器创建输入的上下文嵌入，解码器在生成输出时同时关注编码输入和自身已生成序列。**交叉注意力（Cross-attention）** 让解码器聚焦于编码输入的相关部分 | 翻译、摘要、文本简化、结构化数据生成 |

**嵌入在所有三种架构中扮演的四种角色**：

| 角色 | 说明 |
|------|------|
| **输入嵌入（Input Embeddings）** | 将 token 转换为模型可处理的向量 |
| **位置嵌入（Positional Embeddings）** | 编码序列顺序信息 |
| **层嵌入（Layer Embeddings）** | 捕获越来越抽象的特征（每层变换） |
| **输出嵌入（Output Embeddings）** | 在生成式模型中，将向量映射回 token 概率 |

**为什么 Transformer 嵌入比 Word2Vec 强大**：
- Word2Vec 是**静态的**——同一个词不管上下文，向量都一样
- Transformer 是**动态的**——同一个词在不同上下文中获得不同向量
- Transformer 通过自注意力（Self-attention）、交叉注意力（Cross-attention）、前馈网络（Feed-forward Networks）动态更新和精化嵌入，同时保持向量运算的数学性质

---

### 6. 专用嵌入模型（Embedding Models）（2.5）

**定义**：专门设计用于生成高质量文本向量表示的独立模型（如 OpenAI 的 `text-embedding-ada-002`、Cohere 的 `embed-multilingual-v3`）。与通用 LLM 不同，它们**只做一件事**：把文本映射为语义精准的向量。

**核心思想**：不需要一个能写诗、能聊天的大模型——很多场景只需要一个小而快、专注于生成好嵌入的模型。

**与之前讨论的模型/方法的区别**：

| 区别维度 | 专用嵌入模型 | Word2Vec / Doc2Vec | 通用 LLM (GPT, BERT) |
|----------|------------|-------------------|---------------------|
| **架构** | 基于 Transformer，但移除了文本生成组件，专注于嵌入生成 | 浅层神经网络 | 完整 Transformer |
| **一致性（Consistency）** | 同一输入始终产生同一向量——适合搜索和检索 | 静态，天然一致 | 内部表示动态变化，不直接适合检索 |
| **计算效率** | 比完整 LLM 小得多、快得多 | 很轻量 | 大而慢 |
| **优化目标** | 专门为相似性比较训练（常用对比学习），确保相似文本映射到相似向量 | 预测上下文/中心词 | 预测下一个 token 或遮蔽 token |

**在 RAG 系统中的三大角色**：

**1. 文档处理流水线（Document Processing Pipeline）**：
- 文档被切分成块（chunks）
- 每个块通过嵌入模型生成向量
- 向量存入向量数据库以供高效检索

**2. 查询处理（Query Processing）**：
- 用户查询通过**同一个**嵌入模型转换为向量
- 从数据库中检索相似向量
- 检索到的内容作为上下文提供给 LLM 生成回答

**3. 混合搜索系统（Hybrid Search Systems）**：
- 嵌入模型提供语义搜索能力
- 可与传统关键词搜索结合，效果更好
- 支持跨模态搜索（文本、图片、其他数据类型）

**实际应用场景**：
- **语义搜索**（Semantic Search）：按意思而非关键词找文档
- **内容推荐**（Content Recommendation）：推荐相似文章/产品/媒体
- **重复检测**（Duplicate Detection）：在大数据集中识别相似内容
- **数据去重**（Data Deduplication）：找到并删除数据库中近似重复的条目
- **知识库构建**（Knowledge Base Construction）：组织和关联相关信息
- **问答系统**（Question Answering）：为特定问题找到相关上下文
- **内容聚类**（Content Clustering）：自动将相似文档或段落分组

**最新发展趋势**：

| 趋势 | 说明 |
|------|------|
| **多模态嵌入（Multi-modal Embeddings）** | 跨文本、图片、代码创建兼容的嵌入 |
| **指令调优嵌入（Instruction-Tuned Embeddings）** | 根据特定指令调整嵌入策略 |
| **领域专用嵌入（Domain-Specific Embeddings）** | 针对法律、医学、科研等特定领域优化 |
| **分层嵌入（Hierarchical Embeddings）** | 在多个粒度级别（词、句子、段落）创建嵌入 |
| **高效嵌入（Efficient Embedding）** | 降低嵌入维度同时保持语义质量 |

> **关键洞察**：虽然大语言模型更受关注，但在实际生产中，**嵌入模型的选择和使用方式往往决定了一个 AI 应用是否真的实用和有效**。

> **常见误用**：用不同的嵌入模型分别编码文档和查询。在 RAG 系统中，文档入库时用模型 A 生成向量，查询时用模型 B 生成向量，然后尝试做相似性搜索。这会导致灾难性后果——不同模型的向量空间完全不同，"距离近"在两个空间中含义不同，搜索结果会完全不可靠。必须确保文档嵌入和查询嵌入使用**同一个模型**。如果需要更换模型，所有文档必须用新模型重新嵌入。

**代码示例：用 ChromaDB + SentenceTransformer 实现完整 RAG Pipeline**

```python
from sentence_transformers import SentenceTransformer
from chromadb import Client, Settings
import chromadb

# 1. 初始化嵌入模型
# all-MiniLM-L6-v2 是轻量但有效的模型，在速度和质量之间取得良好平衡
model = SentenceTransformer('all-MiniLM-L6-v2')

# 2. 初始化 ChromaDB 作为向量存储
# 使用内存存储（演示用）
chroma_client = Client(Settings(is_persistent=False))
collection = chroma_client.create_collection(name="climate_docs")

# 3. 准备文档
documents = [
    "Climate change is affecting global weather patterns causing more extreme events.",
    "Rising sea levels threaten coastal communities worldwide.",
    "Greenhouse gas emissions continue to rise despite international agreements."
]

# 4. 生成嵌入
# model.encode() 将文本转换为密集向量（嵌入）
embeddings = model.encode(documents)

# 5. 存储文档和嵌入
# ChromaDB 需要嵌入为 list 格式，所以要从 numpy array 转换
collection.add(
    embeddings=[e.tolist() for e in embeddings],
    documents=documents,
    ids=[f"doc_{i}" for i in range(len(documents))]
)

# 6. 处理查询
query = "How does climate change affect weather?"
query_embedding = model.encode(query)

# 7. 相似性搜索
results = collection.query(
    query_embeddings=[query_embedding.tolist()],
    n_results=2  # 返回最相似的 2 篇文档，默认使用余弦相似度
)

# 8. 输出结果
for doc in results['documents'][0]:
    print(f"Retrieved document: {doc}")
```

**代码逐步解读**：

| 步骤 | 代码 | 说明 |
|------|------|------|
| 模型初始化 | `SentenceTransformer('all-MiniLM-L6-v2')` | 加载预训练模型，输出 384 维嵌入，速度质量平衡好 |
| 向量存储初始化 | `Client(Settings(is_persistent=False))` | 内存模式的 ChromaDB，创建一个 collection |
| 文档嵌入 | `model.encode(documents)` | 返回 shape 为 `(n_documents, 384)` 的 numpy 数组，自动批处理 |
| 存储 | `collection.add(...)` | 同时存储原始文档、嵌入和唯一 ID |
| 查询嵌入 | `model.encode(query)` | 用同一个模型编码查询，确保在同一向量空间 |
| 相似搜索 | `collection.query(...)` | 默认余弦相似度，返回 top-k 最相似文档 |

**生产环境还需要加的东西**：
- 错误处理
- 大文档集的批处理
- 长文本的分块（chunking）
- 持久化存储配置
- 距离阈值过滤
- 元数据处理

---

### 7. Sentence Transformers：文本嵌入的瑞士军刀（2.6）

**定义**：Sentence Transformers（`sentence-transformers` 库），由 SBERT.net 开发，是目前最实用、最广泛使用的文本嵌入生成库。它提供了一系列专门为句子嵌入优化的预训练模型，接口简单，性能强劲。

**三大主流模型对比**：

| 特性 | all-MiniLM-L6-v2 | all-mpnet-base-v2 | paraphrase-multilingual-mpnet-base-v2 |
|------|-------------------|-------------------|---------------------------------------|
| **维度** | 384 | 768 | 768 |
| **模型大小** | ~80MB | ~420MB | ~970MB |
| **速度** | 极快（GPU 上 2000 句/秒） | 中等（GPU 上 1000 句/秒） | 较慢 |
| **语言** | 英文为主 | 英文为主 | 50+ 语言 |
| **优点** | 速度-性能比极佳、体积小、通用性好 | 最先进的性能、语义理解更强、跨领域更稳健 | 跨语言能力出色、各语言性能一致、可跨语言比较文本 |
| **缺点** | 专业任务不如大模型、准确度较低 | 体积大、推理慢 | 体积大、比单语模型慢 |
| **最佳场景** | 生产环境（速度和效率优先） | 准确度关键的应用 | 多语言应用 |

> **本书选择**：后续章节主要使用 `all-MiniLM-L6-v2`，因为它在企业应用中代表了最佳平衡——足够快（实时应用）、足够小（容易部署）、足够准（多数场景够用）、社区支持好、生产就绪。

**训练方法**：

| 训练技术 | 说明 |
|----------|------|
| **知识蒸馏（Knowledge Distillation）** | 小模型（如 all-MiniLM-L6-v2）被训练去模仿大模型的行为，同时保持高效 |
| **对比学习（Contrastive Learning）** | 用相似句子对（正样本）和不相似句子对（负样本）训练 |
| **多目标训练** | 自然语言推理（NLI）数据、释义检测、语义文本相似度（STS）基准、问答对 |

**代码示例：单语 vs 多语模型对比**

```python
from sentence_transformers import SentenceTransformer
import numpy as np

# 加载两个模型
monolingual_model = SentenceTransformer('all-MiniLM-L6-v2')
multilingual_model = SentenceTransformer('paraphrase-multilingual-mpnet-base-v2')

# 不同语言的同义句
sentences = {
    'english': "The weather is beautiful today.",
    'spanish': "El clima está hermoso hoy.",
    'french': "Le temps est magnifique aujourd'hui.",
    'german': "Das Wetter ist heute wunderschön."
}

# 余弦相似度计算
def compute_similarity(emb1, emb2):
    return np.dot(emb1, emb2) / (np.linalg.norm(emb1) * np.linalg.norm(emb2))

# 跨语言相似度对比
def compare_sentences(model, sentences):
    embeddings = {
        lang: model.encode(text, convert_to_numpy=True)
        for lang, text in sentences.items()
    }
    print(f"\nSimilarity scores for {model.__class__.__name__}:")
    for lang1 in sentences:
        for lang2 in sentences:
            if lang1 < lang2:  # 避免重复比较
                sim = compute_similarity(embeddings[lang1], embeddings[lang2])
                print(f"  {lang1} vs {lang2}: {sim:.4f}")

# 测试两个模型
for model in [monolingual_model, multilingual_model]:
    compare_sentences(model, sentences)

# 批处理示例（效率更高）
texts = list(sentences.values())
batch_embeddings = multilingual_model.encode(texts, batch_size=8)
```

**运行结果你会看到**：
1. 多语言模型对同一句话的不同语言版本产生相似分数
2. 单语模型对非英文文本表现不一致
3. 批处理显著快于逐句编码

---

**最佳实践 6 条——每条详解**：

#### 实践一：批处理（Batch Processing）

```python
# ✅ 好的做法
embeddings = model.encode(sentences, batch_size=32)

# ❌ 坏的做法
embeddings = [model.encode(sentence) for sentence in sentences]
```

**为什么重要**：
- 批处理允许并行处理多个句子，更高效地利用 GPU/CPU 资源
- 比逐句处理快 **10-100 倍**
- 减少内存碎片和系统开销

**不做的后果**：
- 处理速度显著变慢
- 因重复加载模型导致更高的内存开销
- 硬件能力（尤其是 GPU）利用不充分
- 生产环境中成本更高

---

#### 实践二：长度处理（Length Handling）

```python
# ✅ 好的做法
max_seq_length = model.max_seq_length

def chunk_text(text, max_length=max_seq_length):
    # 将长文本切分为不超过最大长度的块
    chunks = [text[i:i + max_length] for i in range(0, len(text), max_length)]
    return chunks

# 处理长文档
long_text_embeddings = model.encode(chunk_text(long_document))
```

**为什么重要**：
- 防止重要信息被截断
- 确保长文档处理的一致性
- 保持模型在长文本上的性能
- 避免意外行为或错误

**不做的后果**：
- **静默截断**（Silent Truncation）——模型不会报错，直接丢掉超长部分
- 可能丢失关键信息
- 长文档结果不一致
- 嵌入质量可能下降

---

#### 实践三：归一化（Normalization）

```python
# ✅ 好的做法
embeddings = model.encode(sentences, normalize_embeddings=True)

# 或手动归一化
from sklearn.preprocessing import normalize
embeddings = normalize(embeddings)
```

**为什么重要**：
- 使余弦相似度计算更高效
- 确保不同长度文本的相似度分数一致
- 减少文本长度对相似度计算的影响
- 对于跨文档嵌入比较非常重要

**不做的后果**：
- 相似度分数不一致
- 比较时存在长度偏差（Length Bias）
- 搜索结果不可靠
- 下游处理更复杂

> **常见误用**：对长文本和短文本的嵌入不做归一化就直接计算余弦相似度。虽然余弦相似度本身对向量长度不敏感（因为它看的是方向），但部分嵌入模型输出的向量长度隐含了"信息量"或"置信度"信息，不归一化可能导致长文本系统性地获得更高的内积分数。尤其当你用内积（dot product）而非余弦相似度做搜索时，不归一化会让长文本的嵌入在搜索中占据不公平的优势。

---

#### 实践四：GPU 利用（GPU Utilization）

```python
# ✅ 好的做法（带错误处理）
import torch
device = 'cuda' if torch.cuda.is_available() else 'cpu'
model = SentenceTransformer('all-MiniLM-L6-v2', device=device)
```

**为什么重要**：
- 比 CPU 处理快 **10-50 倍**
- 处理大量文本的必备条件
- 使实时应用成为可能
- 生产环境中更具成本效益

**不做的后果**：
- 处理速度显著变慢
- 运营成本更高
- 实时应用性能差
- 资源利用效率低

---

#### 实践五：内存管理（Memory Management）

```python
# ✅ 好的做法
import gc
import torch

def process_large_dataset(sentences, batch_size=32):
    embeddings = []
    for i in range(0, len(sentences), batch_size):
        batch = sentences[i:i + batch_size]
        batch_embeddings = model.encode(batch)
        embeddings.extend(batch_embeddings)
        # 需要时清理 GPU 内存
        if torch.cuda.is_available():
            torch.cuda.empty_cache()
        gc.collect()
    return embeddings
```

**为什么重要**：
- 防止大数据集导致内存溢出（OOM）错误
- 确保长时间运行时性能稳定
- 生产环境的刚需
- 使处理超大数据集成为可能

**不做的后果**：
- 内存耗尽导致随机崩溃
- 性能随时间劣化
- 系统不稳定
- 无法处理大数据集

---

#### 实践六：错误处理（Error Handling）

```python
# ✅ 好的做法
from typing import List, Optional
import numpy as np

def safe_encode(
    model: SentenceTransformer,
    texts: List[str],
    batch_size: int = 32
) -> Optional[np.ndarray]:
    try:
        # 处理空输入或无效输入
        if not texts or not isinstance(texts, list):
            return None

        # 去除空字符串和其他边界情况
        valid_texts = [str(t) for t in texts if t and str(t).strip()]
        if not valid_texts:
            return None

        return model.encode(valid_texts, batch_size=batch_size)
    except Exception as e:
        print(f"Error encoding texts: {e}")
        return None
```

**为什么重要**：
- 防止非法输入导致崩溃
- 让系统更健壮
- 更容易排查问题
- 生产可靠性更好

**不做的后果**：
- 意外输入导致系统崩溃
- 难以诊断问题
- 用户体验差
- 维护成本更高

---

### 8. 嵌入层与零样本学习（2.7）

**定义**：Transformer 的嵌入层不只是一个把 token 转成向量的查找表——它是让模型能够在**没有明确训练**的情况下理解和泛化到新概念的基础，这种能力叫做零样本学习（Zero-Shot Learning）。

**直觉建立**：想象你从小学习了"水果"这个概念——你见过苹果、香蕉、橙子。现在有人给你看一个你从未见过的"火龙果"，问你这是不是水果。你不需要专门学过火龙果，就能判断它很可能是水果——因为它和你已知的水果共享某些特征（有果肉、有种子、长在植物上）。这就是零样本学习的核心思路。在嵌入空间中，模型把所有已知概念放在了一个有结构的空间里——"水果"区域的向量彼此靠近，"车辆"区域的向量彼此靠近。当一个新概念出现时，模型只需要看它的嵌入向量落在空间的哪个区域附近，就能推断出它属于什么类别。这依赖两个关键性质：一是嵌入的**组合性**（新概念可以由已知概念的特征组合而成），二是**共享语义空间**（相似概念在嵌入空间中自然聚集）。

**Transformer 嵌入的三组件（完整代码实现）**

```python
import torch
import torch.nn as nn

class TransformerEmbeddings(nn.Module):
    def __init__(self, vocab_size, embedding_dim, max_sequence_length):
        super().__init__()
        # Token 嵌入：将词转换为向量
        self.token_embeddings = nn.Embedding(vocab_size, embedding_dim)
        # 位置嵌入：添加序列顺序信息
        self.position_embeddings = nn.Embedding(max_sequence_length, embedding_dim)
        # 段嵌入：区分输入的不同部分（如问题 vs 上下文）
        self.segment_embeddings = nn.Embedding(2, embedding_dim)
        # 层归一化（稳定训练）
        self.layer_norm = nn.LayerNorm(embedding_dim)
        self.dropout = nn.Dropout(0.1)

    def forward(self, input_ids, segment_ids=None):
        seq_length = input_ids.size(1)
        # 创建位置索引 (0, 1, 2, ...)
        position_ids = torch.arange(seq_length, device=input_ids.device)
        position_ids = position_ids.unsqueeze(0).expand_as(input_ids)
        # 若未提供 segment_ids，默认全 0
        if segment_ids is None:
            segment_ids = torch.zeros_like(input_ids)
        # 三个嵌入相加
        embeddings = (
            self.token_embeddings(input_ids) +     # 词义
            self.position_embeddings(position_ids) + # 序列顺序
            self.segment_embeddings(segment_ids)     # 输入段落
        )
        # 归一化 + Dropout
        embeddings = self.layer_norm(embeddings)
        embeddings = self.dropout(embeddings)
        return embeddings
```

**三组件详解**：

| 组件 | 作用 | 类比 |
|------|------|------|
| **Token Embedding** | 把每个 token 映射为向量，编码词义 | 字典里每个词的定义 |
| **Position Embedding** | 编码 token 在序列中的位置 | 句子中词的顺序（"dog bites man" ≠ "man bites dog"） |
| **Segment Embedding** | 区分输入的不同段落（如 BERT 的问题段 vs 上下文段） | 标记"这是提问"还是"这是参考文本" |

> 最终嵌入 = Token Embedding + Position Embedding + Segment Embedding，然后过 LayerNorm 和 Dropout。

**嵌入如何支撑零样本学习**：

零样本学习的意思是模型能处理训练时从未见过的类别/任务。嵌入层之所以能支撑这一能力，有两个关键原因：

**1. 组合性（Compositional Nature）**：
- 嵌入捕获了概念之间的语义关系
- 新概念可以通过已知嵌入的组合来理解
- 模型能通过组合熟悉的元素来处理全新的情况

**2. 共享语义空间（Shared Semantic Space）**：
- 含义相似的词在嵌入空间中聚集在一起
- 这种聚集使模型能泛化到未见过的样本
- 模型可以推断出从未一起出现过的概念之间的关系

**零样本分类代码示例**：

```python
from sentence_transformers import SentenceTransformer, util
import torch

class ZeroShotClassifier:
    def __init__(self, model_name='all-mpnet-base-v2'):
        self.model = SentenceTransformer(model_name)

    def classify(self, text, candidate_labels):
        # 编码输入文本
        text_embedding = self.model.encode(text, convert_to_tensor=True)
        # 为每个候选标签构造提示语
        label_prompts = [f"This text is about {label}" for label in candidate_labels]
        label_embeddings = self.model.encode(label_prompts, convert_to_tensor=True)
        # 计算余弦相似度
        similarities = util.pytorch_cos_sim(text_embedding, label_embeddings)[0]
        # 构建结果字典
        results = {
            label: float(score)
            for label, score in zip(candidate_labels, similarities)
        }
        return results

# 使用示例
classifier = ZeroShotClassifier()
text = "The new quantum computer can perform calculations in seconds that would take classical computers thousands of years."
labels = ["technology", "sports", "cooking", "politics"]
results = classifier.classify(text, labels)

print("Zero-shot classification results:")
for label, score in sorted(results.items(), key=lambda x: x[1], reverse=True):
    print(f"  {label}: {score:.3f}")
```

**核心原理**：不需要为"量子计算"这个主题训练分类器——只要把文本和候选标签都编码到同一个语义空间，比较相似度就行了。这就是嵌入层带来的泛化能力。

**上下文理解示例**：

```python
# 上下文如何影响嵌入
texts = [
    "The bank by the river is eroding",    # bank = 河岸
    "The bank approved my loan"             # bank = 银行
]
embeddings = model.encode(texts)
# 这两句话虽然都含 "bank"，但嵌入完全不同
```

**语义关系推理（类比推理）**：

```python
def find_analogies(model, a1, a2, b1):
    """找到满足 a1 : a2 :: b1 : b2 的 b2"""
    e_a1 = model.encode(a1)
    e_a2 = model.encode(a2)
    e_b1 = model.encode(b1)
    # 计算关系向量
    relationship = e_a2 - e_a1
    # 预测 b2 的嵌入
    predicted_b2 = e_b1 + relationship
    return predicted_b2
```

**跨模态能力：CLIP（Contrastive Language-Image Pre-training）**

```python
from transformers import CLIPProcessor, CLIPModel

class MultiModalEmbeddings:
    def __init__(self):
        self.model = CLIPModel.from_pretrained("openai/clip-vit-base-patch32")
        self.processor = CLIPProcessor.from_pretrained("openai/clip-vit-base-patch32")

    def get_embeddings(self, text=None, image=None):
        inputs = self.processor(text=text, images=image, return_tensors="pt")
        outputs = self.model(**inputs)
        # 从共享空间获取嵌入
        if text:
            return outputs.text_embeds
        return outputs.image_embeds
```

> 文本和图片被映射到同一个向量空间，可以用文字搜图、用图搜文。

**局限与注意事项**：

| 局限 | 说明 |
|------|------|
| **嵌入质量** | 训练数据越多通常零样本效果越好；领域专用词汇可能需要特殊处理；非常专业或技术性的概念质量会下降 |
| **计算成本** | 更大的嵌入维度提供更好的表示但增加内存使用；需要在模型大小和性能需求之间权衡 |
| **偏见与公平** | 嵌入会继承训练数据中的偏见；需要在不同领域和上下文中验证零样本性能 |

---

### 9. Word2Vec 向量算术实战（2.8）

**定义**：用 Word2Vec 预训练模型实际操作向量算术，直观感受词嵌入是如何编码语义关系的。

**核心思想**：如果 Word2Vec 真的学到了语义，那么"词语之间的关系"应该能用"向量之间的数学运算"来表达。

**完整实战代码**：

**Step 1：环境安装**

```bash
pip install gensim
pip install numpy
pip install nltk
```

**Step 2：加载预训练模型**

```python
import gensim.downloader as api
import numpy as np
from typing import List, Tuple

# 下载并加载预训练 Word2Vec 模型
# 首次运行会下载约 1.6GB 数据
print("Loading Word2Vec model...")
word2vec_model = api.load('word2vec-google-news-300')
print("Model loaded!")

def cosine_similarity(vec1: np.ndarray, vec2: np.ndarray) -> float:
    """计算两个向量的余弦相似度"""
    return np.dot(vec1, vec2) / (np.linalg.norm(vec1) * np.linalg.norm(vec2))

def find_similar_words(vector: np.ndarray, n: int = 5) -> List[Tuple[str, float]]:
    """找到与输入向量最相似的 n 个词"""
    return word2vec_model.similar_by_vector(vector, topn=n)
```

> **关于模型**：Google 的预训练 Word2Vec 模型，在新闻文章上训练，包含约 300 万词汇的 300 维向量。

**Step 3：实现向量算术函数**

```python
def vector_arithmetic(*words_and_weights: Tuple[str, float]) -> np.ndarray:
    """
    对词进行带权重的向量算术
    示例: vector_arithmetic(("king", 1), ("man", -1), ("woman", 1))
    """
    resulting_vector = np.zeros(word2vec_model.vector_size)
    for word, weight in words_and_weights:
        if word not in word2vec_model:
            raise ValueError(f"Word '{word}' not found in vocabulary")
        resulting_vector += weight * word2vec_model[word]
    return resulting_vector

def print_analogy_results(result_vector: np.ndarray,
                          original_words: List[str],
                          n_results: int = 5):
    """以可读格式打印向量算术结果"""
    similar_words = find_similar_words(result_vector, n_results)
    print("\nVector arithmetic:", " + ".join(original_words))
    print("\nMost similar words:")
    for word, similarity in similar_words:
        if word not in original_words:  # 不显示输入词
            print(f"  {word}: {similarity:.4f}")
```

**Step 4：经典 King-Queen 类比**

```python
def demonstrate_royal_analogy():
    print("\n=== Royal Analogy Demonstration ===")
    # king - man + woman = queen
    result = vector_arithmetic(
        ("king", 1),
        ("man", -1),
        ("woman", 1)
    )
    print_analogy_results(
        result,
        ["king", "man", "woman"],
        n_results=5
    )

demonstrate_royal_analogy()
```

**预期输出**：

```
=== Royal Analogy Demonstration ===
Vector arithmetic: king - man + woman

Most similar words:
  queen: 0.7118
  monarch: 0.6523
  princess: 0.6342
  kings: 0.6103
  prince: 0.5932
```

**Step 5：更多有趣的类比**

```python
def explore_more_analogies():
    print("\n=== More Analogy Examples ===")

    # 例 1: Paris 之于 France，如同 Berlin 之于 ...?
    result1 = vector_arithmetic(
        ("Paris", -1),
        ("France", 1),
        ("Berlin", 1)
    )
    print("\nParis : France :: Berlin : ?")
    print_analogy_results(result1, ["Paris", "France", "Berlin"])

    # 例 2: walking 之于 walked，如同 running 之于 ...?
    result2 = vector_arithmetic(
        ("walking", -1),
        ("walked", 1),
        ("running", 1)
    )
    print("\nwalking : walked :: running : ?")
    print_analogy_results(result2, ["walking", "walked", "running"])

    # 例 3: good 之于 better，如同 bad 之于 ...?
    result3 = vector_arithmetic(
        ("good", -1),
        ("better", 1),
        ("bad", 1)
    )
    print("\ngood : better :: bad : ?")
    print_analogy_results(result3, ["good", "better", "bad"])

explore_more_analogies()
```

**Step 6：交互式探索工具**

```python
def explore_custom_analogy():
    """交互式类比探索工具"""
    print("\n=== Custom Analogy Explorer ===")
    print("Format: word1 is to word2 as word3 is to ???")

    try:
        word1 = input("Enter word1: ").strip()
        word2 = input("Enter word2: ").strip()
        word3 = input("Enter word3: ").strip()

        result = vector_arithmetic(
            (word1, -1),
            (word2, 1),
            (word3, 1)
        )
        print(f"\n{word1} : {word2} :: {word3} : ?")
        print_analogy_results(result, [word1, word2, word3])
    except ValueError as e:
        print(f"Error: {e}")
        print("Make sure all words are in the vocabulary!")

if __name__ == "__main__":
    explore_custom_analogy()
```

**Word2Vec 向量算术能揭示的语义关系模式**：

| 关系类型 | 示例 |
|----------|------|
| **性别关系** | king / queen, actor / actress |
| **动词时态** | walk / walked, run / ran |
| **国家-首都** | France / Paris, Germany / Berlin |
| **比较级** | good / better, bad / worse |

---

## 重点标记

1. **嵌入是现代 AI 的基石**：向量嵌入不只是技术细节，它是人工智能"思考"的介质。所有 LLM 的推理和生成都发生在嵌入的数学空间中。
2. **Word2Vec 的范式转变**：从"词是离散符号"到"词是连续向量空间中的点"，2013 年的这一突破引发了整个深度学习 NLP 革命。向量算术（king - man + woman = queen）是其最直观的证明。
3. **静态 vs 动态嵌入**：Word2Vec/Doc2Vec 是静态嵌入（同一个词永远同一个向量），Transformer 是动态嵌入（同一个词根据上下文获得不同向量）。这是理解技术演进的关键分界线。
4. **稀疏 vs 密集是根本性的权衡**：稀疏嵌入可解释但低效且不捕获语义，密集嵌入紧凑高效但难以解释。现代最佳实践是两者结合（Hybrid Search）。
5. **专用嵌入模型 ≠ LLM**：专用嵌入模型更小、更快、输出更一致，专为相似性比较优化。在 RAG 等生产系统中，嵌入模型的选择往往比 LLM 的选择更影响最终效果。
6. **Sentence Transformers 的 6 条最佳实践**：批处理（10-100x 加速）、长度处理（防止静默截断）、归一化（消除长度偏差）、GPU 利用（10-50x 加速）、内存管理（防 OOM）、错误处理（防崩溃）。每一条都有"不做会怎样"的明确后果。
7. **零样本学习依赖嵌入的组合性和共享语义空间**：正因为嵌入空间中相似概念聚集、新概念可以由已知概念组合，模型才能处理训练时从未见过的类别。
8. **all-MiniLM-L6-v2 是本书后续章节的主力模型**：384 维、80MB、2000 句/秒——在速度、体积、准确度之间的最优平衡。
9. **Word2Vec 向量算术不只是演示**：它证明了嵌入空间中存在可解释的语义维度（性别、时态、国别、比较级等），这是理解所有后续嵌入技术的直觉基础。

---

## 自测：你真的理解了吗？

> 以下问题考的是理解，不是记忆。试着先思考再看答案。

**Q1**：你正在构建一个法律文档检索系统，用户会输入如"关于知识产权侵权的判例"这样的查询。你的同事建议直接用 Word2Vec 把每个文档的所有词向量取平均来表示文档，然后做余弦相似度搜索。这个方案有什么根本性缺陷？你会推荐用什么替代方案？为什么？

**Q2**：你的团队在选型嵌入模型：方案 A 是 all-MiniLM-L6-v2（384 维、80MB、2000 句/秒），方案 B 是 all-mpnet-base-v2（768 维、420MB、1000 句/秒）。你的系统需要实时处理用户查询（延迟要求 <100ms），文档库有 500 万篇，部署在一台 16GB 内存的服务器上。你选哪个？如果准确度是绝对优先（比如医疗诊断辅助系统），你的选择会变吗？

**Q3**：有人说"密集嵌入在所有方面都优于稀疏嵌入，稀疏嵌入已经过时了"。这个说法对吗？举一个具体场景说明密集嵌入会失败但稀疏嵌入能成功的情况。现代搜索引擎为什么采用两者结合的混合搜索？

**Q4**：你用 SentenceTransformer 处理一个包含 10 万篇长文档（每篇平均 5000 词）的数据集。你没有做任何长度处理，直接调用 `model.encode(documents)`。会发生什么？模型会报错吗？结果会有什么问题？你应该怎么做？

**Q5**：你的 RAG 系统上线三个月后，团队决定把嵌入模型从 all-MiniLM-L6-v2 升级到一个更新、更准确的模型。你只更新了查询端的模型，文档库中的向量还是旧模型生成的。用户反馈说"搜索结果变得乱七八糟"。分析这个问题的根因，以及正确的升级流程应该是什么。
