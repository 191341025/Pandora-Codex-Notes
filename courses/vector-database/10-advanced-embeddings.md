# 模块十：高级嵌入技术

> 对应 PDF: 10_Advanced_Embedding_Techniques.pdf，第 1-3 页

## 概念地图

- **核心概念** (必须内化): Matryoshka 嵌套嵌入的截断特性与分层搜索策略、MaxSim 评分机制（ColPali 的 late interaction 核心）、统一多模态搜索引擎的"专家模型各司其职+路由层统一调度"架构
- **实操要点** (动手时需要): CLIP 图文编码与 pgvector 存储流程、Matryoshka 三阶段渐进检索的实现（128D→256D→768D）、ColPali 多向量嵌入的序列化存储与 MaxSim 搜索
- **背景知识** (扩展理解): CLIP 对比学习的训练原理、ColPali 基于 PaliGemma VLM 的视觉理解机制、三种技术的存储/延迟/精度权衡

---

## 概念讲解

### 1. 多模态搜索的需求背景

**定义**：多模态搜索（Multi-modal Search）是指在一个统一的搜索系统中同时检索多种内容类型——照片、文本片段、复杂文档——而不是为每种类型维护一套独立系统。

**核心思想**：传统做法是"一种内容一套管线"，图片有图片的索引、文本有文本的索引、PDF 有 PDF 的解析器，彼此割裂。现代嵌入技术的进步使得我们可以把不同模态的内容映射到可比较的向量空间，从而用一套搜索接口覆盖所有类型。

**为什么重要**：
- **基础设施重复**：分模态建设意味着每种类型都要单独维护索引、查询逻辑和硬件资源，运维成本倍增。
- **搜索体验割裂**：用户想搜"带有attention机制架构图的论文"，在传统体系里需要分别查图片库和文档库，再人工交叉比对。
- **信息损失**：独立系统之间缺少语义关联，无法跨模态做推理（比如"找到和这张照片描述相似的论文段落"）。

**本章要实现的三种技术**：

| 技术 | 目标内容类型 | 一句话描述 |
|------|-------------|-----------|
| CLIP | 图片 | 用自然语言搜索图片，图文跨模态匹配 |
| Matryoshka Embeddings | 文本 | 可变维度嵌入，灵活权衡精度和效率 |
| ColPali | PDF 文档 | 直接理解 PDF 页面的视觉布局，不需要文本提取 |

最终目标是把三者整合进一个 `UnifiedSearchEngine`，对外暴露统一的 `search_all()` 接口。

---

### 2. 公共工具类

在动手实现三种嵌入技术之前，PDF 先定义了两个全章复用的工具函数。这是典型的"先搭基础设施再写业务逻辑"的工程模式。

#### 2.1 设备检测（Device Detection）

**用途**：三种模型（CLIP、Matryoshka、ColPali）都能从 GPU 加速中受益。这个函数自动检测当前机器最优的计算设备。

**优先级顺序**：Apple Silicon MPS > NVIDIA CUDA > CPU

```python
import torch

def get_device():
    """
    Detect best available device for PyTorch models.
    Priority: Apple Silicon MPS > NVIDIA CUDA > CPU
    """
    if torch.backends.mps.is_available():
        return "mps"
    elif torch.cuda.is_available():
        return "cuda"
    else:
        return "cpu"

# Use throughout the chapter
device = get_device()
print(f"Using device: {device}")
```

> **为什么 MPS 排在 CUDA 前面**：因为在 Mac 上 `torch.cuda.is_available()` 一定是 `False`，而在 NVIDIA 机器上 `torch.backends.mps.is_available()` 也是 `False`，所以这个优先级只是语义上的，实际不会冲突。但把 MPS 放第一个更清晰地表达了"苹果设备优先用 MPS"的意图。

#### 2.2 MaxSim 评分函数

**用途**：专门为 ColPali 设计的评分方式。ColPali 产生的是多向量嵌入（每个页面有多个 patch 向量，每个查询有多个 token 向量），不能直接用简单的余弦相似度比较两个单向量。

**MaxSim 的计算逻辑**：
1. 对查询和文档的嵌入分别做 L2 归一化（使其适用余弦相似度）
2. 计算相似度矩阵：`(num_query_tokens, num_patches)`
3. 对每个 query token，找到它与所有 document patch 中最高的相似度
4. 把所有 query token 的最大相似度求和，作为最终得分

简单说就是：**每个查询词都去找文档里和它最像的那个局部区域，然后把这些"最佳匹配分"加起来**。

**直觉建立**：想象你拿着一份购物清单（查询）走进一家大超市（文档页面）。清单上有三项："牛奶""面包""鸡蛋"。你不需要在超市里找到一个**同时**摆着这三样东西的货架——你只需要分别找到牛奶在哪个区域最容易拿到、面包在哪个区域最容易拿到、鸡蛋在哪个区域最容易拿到，然后把三次"找到的容易程度"加起来，就是这家超市对你这份清单的匹配分。MaxSim 就是这个逻辑：对查询的每个 token（清单上的每一项），在文档的所有 patch（超市的每个区域）中找到最佳匹配，然后求和。这种"逐项最佳匹配再汇总"的方式比单向量余弦相似度保留了更多细粒度信息——它不要求查询和文档在"整体方向"上相似，而是允许局部对局部的精确匹配。边界：MaxSim 的计算量正比于 `num_query_tokens * num_patches`，文档越长/patch 越多，计算开销越大。

```python
import numpy as np

def maxsim_score(query_embeddings, document_embeddings):
    """
    Compute MaxSim score between query and document multi-vector embeddings.

    For each query token, find the maximum similarity to any document patch,
    then sum across all query tokens.

    Args:
        query_embeddings: (num_query_tokens, dim) array
        document_embeddings: (num_patches, dim) array
    Returns:
        float: MaxSim score
    """
    # Normalize embeddings for cosine similarity
    query_norm = query_embeddings / (
        np.linalg.norm(query_embeddings, axis=1, keepdims=True) + 1e-8
    )
    doc_norm = document_embeddings / (
        np.linalg.norm(document_embeddings, axis=1, keepdims=True) + 1e-8
    )

    # Compute similarity matrix: (num_query_tokens, num_patches)
    similarity_matrix = np.matmul(query_norm, doc_norm.T)

    # For each query token, find max similarity to any document patch
    max_similarities = np.max(similarity_matrix, axis=1)

    # Sum across query tokens
    return float(np.sum(max_similarities))
```

> **注意 `+ 1e-8`**：这是防止除以零的标准做法。如果某个向量恰好全为零（虽然实际中极罕见），归一化时分母为零会导致 NaN。

---

### 3. CLIP 图文跨模态搜索

#### 3.1 理解 CLIP

**定义**：CLIP（Contrastive Language-Image Pre-training）是 OpenAI 开发的一种模型，它把图像和文本映射到同一个向量空间，使得语义相近的图文对在空间中距离很近。

**核心思想**：传统图像识别模型是分类式的——训练时就定好了"猫/狗/车/人"这几个类别，推理时只能输出这些类别的概率。CLIP 完全不同，它学习的是**图像和自然语言描述之间的关系**。这意味着你可以用任意文本去搜索图片，根本不需要提前定义类别。

**为什么有效**：
- 训练数据是互联网上数亿组"图片+描述"配对，覆盖面极广
- 采用对比学习（Contrastive Learning）：让匹配的图文对靠近、不匹配的远离
- 结果是模型获得了丰富的视觉和语义表征能力，能**零样本泛化**（zero-shot generalize）到训练时从未见过的概念

**工作原理图解**：

```
文本: "sunset over ocean"  ──→  [Text Encoder]  ──→  文本向量 (512D)
                                                           ↕ 余弦相似度
图片: beach_sunset.jpg     ──→  [Image Encoder] ──→  图像向量 (512D)
```

图片和文本各自经过独立的编码器，但编码到同一个 512 维空间。语义匹配的图文对，其向量余弦相似度高。

#### 3.2 适用场景

CLIP 在以下场景表现出色：

- **照片库搜索**：用户输入"sunset over mountains"这样的自然描述来搜索照片
- **电商产品发现**：理解"red leather jacket with zippers"这类查询
- **内容审核**：找到匹配特定违规描述的图片
- **视觉问答**：桥接用户提问和图像内容之间的鸿沟

共同特征是：**查询模式多样且不可预测**，无法提前穷举类别，需要零样本能力。

#### 3.3 局限性与权衡

| 方面 | 具体限制 |
|------|---------|
| **嵌入固定** | 512 维固定大小，对简单场景可能是过度开销 |
| **细粒度弱** | 区分特定犬种、理解技术图表等细粒度任务表现差（训练数据是通用网页数据） |
| **计算开销** | 比传统关键词搜索贵很多，需要 GPU 加速 |
| **训练偏见** | 继承了网页规模训练数据中的偏见 |
| **存储成本** | 每张图 2KB（512 个 float32 × 4 字节），大图库会快速累积 |

**关键权衡**：
- **精度 vs 速度**：更大的变体（如 ViT-L/14）更准但更慢
- **泛化 vs 专精**：开箱即用覆盖面广，但在特定领域可能不如专用模型，需要微调
- **通用能力 vs 存储负担**：通用性带来的代价是每张图都要存 512 维向量

> **常见误用**：把 CLIP 用于细粒度视觉区分任务（如区分不同品种的蘑菇、识别特定零件型号），然后对准确率失望。CLIP 的训练数据是通用网页图文对，它擅长的是"日落""猫""红色跑车"这种粗粒度语义匹配，而非"松茸 vs 牛肝菌"这种专家级区分。如果你的场景需要细粒度识别，应该在 CLIP 基础上用领域数据做微调，或者直接用专用的分类模型。

#### 3.4 代码实现

**加载模型**：

使用 HuggingFace Transformers 库，加载 OpenAI 的 `clip-vit-base-patch32` 模型。

```python
from transformers import CLIPModel, CLIPProcessor
from PIL import Image

# Load model and processor
clip_model = CLIPModel.from_pretrained("openai/clip-vit-base-patch32").to(device)
clip_processor = CLIPProcessor.from_pretrained("openai/clip-vit-base-patch32")
clip_model.eval()
```

> `clip_model.eval()` 关闭 dropout 等训练期行为，确保推理结果稳定。

**基本图文匹配**：

给一张图和几段描述，看哪段描述和图最匹配。

```python
# Load an image
image = Image.open("photo.jpg")

# Define possible descriptions
texts = ["a cat sleeping", "a dog playing", "a bird flying"]

# Process inputs
inputs = clip_processor(text=texts, images=image, return_tensors="pt", padding=True)
inputs = {k: v.to(device) for k, v in inputs.items()}

# Get similarities
with torch.no_grad():
    outputs = clip_model(**inputs)
    logits_per_image = outputs.logits_per_image
    probs = logits_per_image.softmax(dim=1)

# Display results
for text, prob in zip(texts, probs[0]):
    print(f"{text}: {prob.item():.2%}")
```

这段代码的流程：图片和三段文本同时输入 → 模型输出图片对每段文本的相似度 → softmax 归一化为概率分布 → 输出各描述的匹配概率。

**批量编码图片**：

实际场景中不可能逐张处理，需要批量编码提高效率。

```python
from pathlib import Path

def encode_images_batch(image_paths, batch_size=32):
    """Encode multiple images efficiently with CLIP"""
    all_embeddings = []
    for i in range(0, len(image_paths), batch_size):
        batch_paths = image_paths[i:i+batch_size]
        images = [Image.open(p) for p in batch_paths]
        inputs = clip_processor(images=images, return_tensors="pt", padding=True)
        inputs = {k: v.to(device) for k, v in inputs.items()}

        with torch.no_grad():
            image_features = clip_model.get_image_features(**inputs)
            # Normalize for cosine similarity
            image_features = image_features / image_features.norm(dim=-1, keepdim=True)

        all_embeddings.append(image_features.cpu().numpy())
    return np.vstack(all_embeddings)
```

> **关键细节**：`get_image_features` 只返回图像编码器的输出向量，不走完整的图文匹配管线，效率更高。归一化后可以直接用余弦距离做搜索。

**存入 PostgreSQL + pgvector**：

```python
import psycopg2

# Connect and create table
conn = psycopg2.connect("dbname=image_search user=postgres password=password")
cur = conn.cursor()

cur.execute("""
    CREATE TABLE IF NOT EXISTS photos (
        id SERIAL PRIMARY KEY,
        file_path TEXT,
        embedding vector(512)
    )
""")

# Create HNSW index for fast similarity search
cur.execute("""
    CREATE INDEX IF NOT EXISTS photos_embedding_idx
    ON photos USING hnsw (embedding vector_cosine_ops)
""")
conn.commit()
```

> 使用 pgvector 扩展的 `vector(512)` 类型存储嵌入，HNSW 索引支持高效近似最近邻搜索，`vector_cosine_ops` 表示用余弦距离作为度量。

**索引与搜索**：

```python
# Index photos
image_paths = list(Path("photos").glob("*.jpg"))
embeddings = encode_images_batch(image_paths)

for path, embedding in zip(image_paths, embeddings):
    cur.execute(
        "INSERT INTO photos (file_path, embedding) VALUES (%s, %s)",
        (str(path), embedding.tolist())
    )
conn.commit()

# Search with text query
def search_photos(query_text, top_k=5):
    """Search photos using natural language"""
    # Encode text query
    inputs = clip_processor(text=[query_text], return_tensors="pt", padding=True)
    inputs = {k: v.to(device) for k, v in inputs.items()}

    with torch.no_grad():
        text_features = clip_model.get_text_features(**inputs)
        text_features = text_features / text_features.norm(dim=-1, keepdim=True)

    query_embedding = text_features[0].cpu().numpy()

    # Search database using cosine distance
    cur.execute("""
        SELECT file_path, 1 - (embedding <=> %s::vector) AS similarity
        FROM photos
        ORDER BY embedding <=> %s::vector
        LIMIT %s
    """, (query_embedding.tolist(), query_embedding.tolist(), top_k))
    return cur.fetchall()

# Example searches
results = search_photos("sunset over ocean")
for path, similarity in results:
    print(f"{path}: {similarity:.3f}")
```

搜索的核心思路：把文本查询也编码成 512 维向量 → 在数据库中用 `<=>` 运算符（pgvector 的余弦距离）做最近邻查询 → `1 - cosine_distance` 得到相似度分数。

> **Key Insight（PDF 原文强调）**：CLIP 的 512 维嵌入同时捕获了视觉信息和语义信息，使得"people laughing at a party"或"red sports car in the city"这样的查询无需任何定制训练即可工作。

---

### 4. Matryoshka 可变维度嵌入

#### 4.1 理解 Matryoshka 嵌入

**定义**：Matryoshka Representation Learning（MRL）是一种让嵌入向量具有**层级信息结构**的训练方法。名字来自俄罗斯套娃（Matryoshka）——大娃套小娃，每一层都是完整的。

**核心思想**：传统嵌入模型输出固定维度（比如 768 维），你要么全用要么不用，没有中间态。Matryoshka 嵌入的革命性在于：**前 N 维就是一个有效的 N 维嵌入**。前 64 维能用、前 128 维能用、前 256 维也能用，它们分别是越来越精确的表征。

**直觉建立**：想象你在画一幅肖像画。第一笔画出脸的轮廓（最关键的信息——这是一张脸）；接下来几笔画出五官的大致位置（更精确了——能认出是男是女）；再接着添加发型、肤色细节（能认出是谁了）；最后画上睫毛、皱纹等微小细节（完美还原）。Matryoshka 嵌入的训练方式就是强制模型按照这个逻辑排列维度：**前面的维度捕获最粗糙但最重要的语义轮廓，后面的维度逐步添加细节**。所以你可以在任意位置"停笔"（截断），得到的都是一幅完整的、只是精细度不同的画像。传统嵌入则像是同时画了 768 个独立的像素点——你不能只留前 128 个像素点，因为它们不构成一幅连贯的画。关键边界：截断到极小维度（如 32D）时，丢失的细节会累积到影响识别的程度——就像只画了一个圈，你分不清这是人脸还是盘子。

**原理**：
- 训练时使用修改过的损失函数，在**多个截断点**同时优化精度
- 结果是信息按重要性排序：最重要的语义信息集中在最前面的维度，后面的维度逐步添加更细粒度的细节
- 一个 768 维的 Matryoshka 嵌入，天然包含了有效的 512D、256D、128D 甚至 64D 表征

**精度保持能力**（PDF 给出的数据）：

| 维度 | 相对于 768D 全维度的精度 |
|------|------------------------|
| 128D | ~95% |
| 64D | ~90% |

这意味着用 128 维就能保留 95% 的搜索精度，存储却只需全维度的 1/6。

#### 4.2 适用场景

Matryoshka 嵌入在以下场景特别有价值：

- **大规模文档检索**：百万到十亿级文档，传统全维度方式在存储和计算上都不可承受
- **移动/边缘部署**：资源受限的设备用低维度，服务器端用全维度
- **实时搜索系统**：严格的延迟要求，无法负担全维度比较
- **分层推荐系统**：初筛用小维度做粗匹配，精排再用大维度
- **差异化延迟需求**：有些查询需要即时响应（用低维），有些可以等更高精度（用高维）

**核心优势**：用同一套嵌入满足不同精度-速度需求，不需要维护多个独立模型。

#### 4.3 局限性与权衡

| 方面 | 具体限制 |
|------|---------|
| **模型要求** | 不是所有预训练模型都支持截断，需要专门用 Matryoshka loss 训练的模型 |
| **激进截断的代价** | 到很小维度（如 32D）时精度下降会变得明显 |
| **额外复杂性** | 需要管理截断逻辑和可能的多个索引结构 |
| **需要调参** | 选择合适的截断点需要深入理解业务的精度/速度需求 |
| **存储 vs 计算抉择** | 可以存全维度运行时截断（灵活但慢），也可以预计算多个截断版本（快但占更多存储） |

> **常见误用**：对普通（非 Matryoshka 训练的）嵌入模型做截断，以为"前 128 维也能用"。普通模型的 768 维之间没有重要性排序——信息均匀分布在所有维度上，截断前 128 维等于随机丢掉 5/6 的信息，搜索精度会断崖式下降。只有专门用 Matryoshka loss 训练的模型才支持有效截断。在使用前务必确认模型文档中明确标注了 Matryoshka/MRL 支持。

#### 4.4 代码实现

**加载支持 Matryoshka 的模型**：

```python
from sentence_transformers import SentenceTransformer

# Load Matryoshka-capable model
text_model = SentenceTransformer("nomic-ai/nomic-embed-text-v1.5")

texts = [
    "Machine learning is a subset of artificial intelligence",
    "Deep learning uses neural networks with multiple layers"
]

# Encode at different dimensions
embedding_128d = text_model.encode(texts, truncate_dim=128)
embedding_256d = text_model.encode(texts, truncate_dim=256)
embedding_768d = text_model.encode(texts)  # Full dimension

print(f"128D shape: {embedding_128d.shape}")  # (2, 128)
print(f"768D shape: {embedding_768d.shape}")  # (2, 768)

# Important: First 128 dims of 768D embedding equal the 128D embedding
assert np.allclose(embedding_768d[:, :128], embedding_128d)
```

> **重要验证**：最后一行 `assert` 证明了 Matryoshka 的核心特性——768D 嵌入的前 128 维和直接生成的 128D 嵌入完全一致。这不是近似，是精确相等。所以你可以只编码一次（全维度），然后按需截断。

**多维度数据库 Schema**：

为分层搜索存储多个维度的嵌入。

```python
# Create table with multiple embedding columns
cur.execute("""
    CREATE TABLE IF NOT EXISTS text_items (
        id SERIAL PRIMARY KEY,
        content TEXT,
        embedding_128 vector(128),
        embedding_256 vector(256),
        embedding_768 vector(768)
    )
""")

# Index each dimension
for dim in [128, 256, 768]:
    cur.execute(f"""
        CREATE INDEX IF NOT EXISTS text_embedding_{dim}_idx
        ON text_items USING hnsw (embedding_{dim} vector_cosine_ops)
    """)
conn.commit()
```

> **设计思路**：一张表三列嵌入，每列独立建 HNSW 索引。这是"预计算多个截断版本"策略的数据库实现，用空间换时间。

**索引文本**：

```python
documents = [
    "Introduction to vector databases and similarity search",
    "Advanced indexing techniques for high-dimensional vectors",
    "Building scalable retrieval systems with embeddings"
]

for doc in documents:
    # Encode once at full dimension
    full_embedding = text_model.encode(doc)

    # Truncate to each target dimension
    emb_128 = full_embedding[:128].tolist()
    emb_256 = full_embedding[:256].tolist()
    emb_768 = full_embedding.tolist()

    cur.execute("""
        INSERT INTO text_items (content, embedding_128, embedding_256, embedding_768)
        VALUES (%s, %s, %s, %s)
    """, (doc, emb_128, emb_256, emb_768))
conn.commit()
```

只编码一次，然后用 Python 切片 `[:128]`、`[:256]` 截断。

#### 4.5 分层搜索策略（Tiered Search）

这是 Matryoshka 嵌入最精彩的应用——**三阶段渐进式检索**：

```
128D → 粗筛 1000 个候选 → 256D → 精筛 100 个 → 768D → 最终 top_k
```

```python
def search_text_tiered(query, top_k=10):
    """
    Three-tier search: 128D → 256D → 768D
    Each stage narrows candidates with increasing accuracy
    """
    # Encode query once
    query_embedding = text_model.encode(query)

    # Stage 1: Fast filtering with 128D (1000 candidates)
    query_128 = query_embedding[:128].tolist()
    cur.execute("""
        SELECT id, content, 1 - (embedding_128 <=> %s::vector) AS similarity
        FROM text_items
        ORDER BY embedding_128 <=> %s::vector
        LIMIT 1000
    """, (query_128, query_128))
    stage1_ids = [row[0] for row in cur.fetchall()]

    # Stage 2: Refinement with 256D (100 candidates)
    query_256 = query_embedding[:256].tolist()
    cur.execute("""
        SELECT id, content, 1 - (embedding_256 <=> %s::vector) AS similarity
        FROM text_items
        WHERE id = ANY(%s)
        ORDER BY embedding_256 <=> %s::vector
        LIMIT 100
    """, (query_256, stage1_ids, query_256))
    stage2_ids = [row[0] for row in cur.fetchall()]

    # Stage 3: Final ranking with 768D (top_k results)
    query_768 = query_embedding.tolist()
    cur.execute("""
        SELECT id, content, 1 - (embedding_768 <=> %s::vector) AS similarity
        FROM text_items
        WHERE id = ANY(%s)
        ORDER BY embedding_768 <=> %s::vector
        LIMIT %s
    """, (query_768, stage2_ids, query_768, top_k))
    return cur.fetchall()

# Example usage
results = search_text_tiered("How do vector databases work?")
for doc_id, content, similarity in results:
    print(f"{similarity:.3f}: {content[:60]}...")
```

**每一阶段做了什么**：
1. **Stage 1（128D）**：从全量数据中用最廉价的 128 维比较，快速圈定 1000 个粗候选
2. **Stage 2（256D）**：在 1000 个候选中用 256 维做更精确的排序，缩小到 100 个
3. **Stage 3（768D）**：在 100 个候选中用全维度做最终精排，返回 top_k

> **Performance Benefit（PDF 原文强调）**：分层搜索只对最终候选计算昂贵的 768D 相似度，而不是对整个数据集。在百万文档规模上，这可以将计算量**减少 100 倍**，同时保持 **98%+ 的召回率**。

---

### 5. ColPali PDF 文档搜索

#### 5.1 理解 ColPali

**定义**：ColPali 是一种基于 Vision Language Model 的文档检索方法，它把 PDF 的每一页当作一张图片来理解，完全跳过传统的文本提取步骤。

**核心思想**：传统文档搜索走的是一条脆弱的管线：OCR 提取文字 → 文本分块 → 嵌入每个块 → 祈祷过程中没丢失什么重要信息。这条路在表格、图表、公式、手写内容面前都会崩溃——表格变成乱序文本、图表被忽略、数学公式变成乱码。

ColPali 的思路截然不同：**直接把页面当图来看**。它用 Vision Language Model（基于 PaliGemma）为每个页面生成**多个嵌入向量**（每个图像 patch 一个），保留了空间关系和视觉线索。搜索时采用 late interaction 机制（就是前面定义的 MaxSim），让查询的每个 token 和文档的每个 patch 逐一比较。

**为什么有效**：
- 现代 VLM（如 PaliGemma）在预训练阶段已经学会了"阅读"和"理解"视觉内容
- ColPali 在此基础上做检索微调，让模型生成的嵌入专门为搜索优化
- 多向量表示 + late interaction 保留了细粒度的空间信息

**与传统方法的对比**：

| 方面 | 传统 OCR + 文本嵌入 | ColPali |
|------|-------------------|---------|
| **处理管线** | OCR → 分块 → 嵌入（多步骤，每步都可能丢信息） | 页面图片 → 多向量嵌入（端到端） |
| **表格处理** | 表格结构被 OCR 打乱，变成无序文本 | 直接理解表格的视觉布局 |
| **图表/公式** | 完全忽略或变成乱码 | 作为视觉信息被模型理解 |
| **布局语义** | 丢失（高亮、颜色编码、空间关系） | 完整保留 |
| **OCR 质量依赖** | 高度依赖，扫描质量差时灾难性退化 | 不依赖 OCR |

#### 5.2 适用场景

- **科学论文检索**：公式、实验结果表格、图表和正文同等重要
- **财务报告搜索**：季度数据以复杂格式化表格和图表呈现
- **技术文档**：充满架构图和代码片段
- **表单/发票处理**：空间布局定义了字段之间的关系
- **幻灯片搜索**：视觉设计和布局是内容含义不可分割的一部分

共同特征：**视觉格式承载了语义信息**，纯文本提取会丢失关键上下文。

#### 5.3 局限性与权衡

| 方面 | 具体数据 |
|------|---------|
| **存储** | 每页约 512KB（是传统嵌入 2KB 的 100-500 倍），因为每页有数百个 patch 向量 |
| **速度** | 查询延迟 100-500ms（传统方式 10-50ms） |
| **元数据盲区** | 无法利用文档元数据（作者、日期、标签等） |
| **GPU 需求** | 推理需要大量 GPU 显存，大规模部署成本高 |

**核心权衡**：
- **存储 vs 精度**：多向量存储让细粒度匹配成为可能（能精确定位到特定图表或表格），但代价是每页数百倍的存储开销
- **速度 vs 质量**：late interaction 机制让 ColPali 能找到真正相关的内容，但也让它比单向量比较慢得多
- **通用 vs 专精**：开箱即用但在医疗记录、法律合同等专业领域可能需要微调

#### 5.4 代码实现

**安装依赖**：

```bash
pip install colpali-engine torch pillow pdf2image
```

**加载模型**：

```python
from colpali_engine.models import ColPali, ColPaliProcessor
from pdf2image import convert_from_path

# Load model
colpali_model = ColPali.from_pretrained(
    "vidore/colpali-v1.2",
    torch_dtype=torch.float16 if device != "cpu" else torch.float32
).to(device)
colpali_processor = ColPaliProcessor.from_pretrained("vidore/colpali-v1.2")
colpali_model.eval()
```

> **注意 `torch.float16`**：在 GPU 上使用半精度浮点可以减少一半显存占用，对检索精度影响极小。CPU 上不支持 float16 加速所以回退到 float32。

**处理 PDF 页面**：

每一页转成图片，然后生成多向量嵌入。

```python
def process_pdf(pdf_path):
    """Convert PDF pages to multi-vector embeddings"""
    # Convert PDF to images (one per page)
    images = convert_from_path(pdf_path, dpi=150)

    page_embeddings = []
    for page_num, image in enumerate(images):
        inputs = colpali_processor.process_images([image]).to(device)
        with torch.no_grad():
            # Returns multi-vector embeddings: (num_patches, 128)
            embeddings = colpali_model(**inputs)

        page_embeddings.append({
            'page_num': page_num,
            'embeddings': embeddings[0].cpu().numpy(),
            'num_patches': embeddings.shape[1]
        })
    return page_embeddings

# Process a research paper
paper_embeddings = process_pdf("papers/attention_is_all_you_need.pdf")
print(f"Processed {len(paper_embeddings)} pages")
print(f"Page 1 has {paper_embeddings[0]['num_patches']} patch embeddings")
```

每页会产生 `num_patches` 个 128 维向量，而不是一个单一向量。这就是 ColPali 多向量表示的来源。

**编码文本查询**：

```python
def encode_query(query_text):
    """Encode text query into multi-vector representation"""
    inputs = colpali_processor.process_queries([query_text]).to(device)
    with torch.no_grad():
        query_embeddings = colpali_model(**inputs)
    return query_embeddings[0].cpu().numpy()

# Example
query = encode_query("transformer architecture with attention mechanism")
print(f"Query has {query.shape[0]} token embeddings")
```

查询也被编码成多个 token 向量，每个 token 一个。

**使用 MaxSim 搜索文档**：

```python
# Score query against document pages
query = encode_query("transformer attention mechanism")

for page in paper_embeddings:
    score = maxsim_score(query, page['embeddings'])
    print(f"Page {page['page_num']}: {score:.3f}")
```

**数据库存储**：

由于是多向量（维度不固定），不能用 pgvector 的 `vector` 类型，改用序列化的 `BYTEA` 存储。

```python
import io

# Create documents table
cur.execute("""
    CREATE TABLE IF NOT EXISTS documents (
        id SERIAL PRIMARY KEY,
        file_path TEXT,
        page_num INTEGER,
        num_patches INTEGER,
        embedding_data BYTEA
    )
""")
cur.execute("""
    CREATE INDEX IF NOT EXISTS doc_path_page_idx
    ON documents(file_path, page_num)
""")
conn.commit()

# Store document page
def store_document_page(file_path, page_num, embeddings):
    """Store multi-vector embeddings for a document page"""
    buffer = io.BytesIO()
    np.save(buffer, embeddings)
    cur.execute("""
        INSERT INTO documents (file_path, page_num, num_patches, embedding_data)
        VALUES (%s, %s, %s, %s)
    """, (file_path, page_num, embeddings.shape[0], buffer.getvalue()))

# Index documents
papers = Path("papers").glob("*.pdf")
for paper in papers:
    page_embeddings = process_pdf(paper)
    for page in page_embeddings:
        store_document_page(str(paper), page['page_num'], page['embeddings'])
conn.commit()
```

> **注意**：这里的索引是 `(file_path, page_num)` 的 B-Tree 索引，不是向量索引。因为多向量存储为 BYTEA，pgvector 无法直接在上面建 HNSW。搜索时需要全量扫描并在应用层用 MaxSim 计算分数。这是 ColPali 存储成本高的一个体现。

**搜索存储的文档**：

```python
def search_documents(query_text, top_k=5):
    """Search all stored documents"""
    query_embeddings = encode_query(query_text)

    # Retrieve all document pages
    cur.execute("SELECT id, file_path, page_num, embedding_data FROM documents")

    results = []
    for row in cur.fetchall():
        doc_id, file_path, page_num, embedding_bytes = row

        # Deserialize embeddings
        buffer = io.BytesIO(embedding_bytes)
        doc_embeddings = np.load(buffer)

        # Compute score using our utility function
        score = maxsim_score(query_embeddings, doc_embeddings)
        results.append({
            'document': file_path,
            'page': page_num,
            'score': score
        })

    # Sort and return top k
    results.sort(key=lambda x: x['score'], reverse=True)
    return results[:top_k]

# Example search
results = search_documents("attention mechanism architecture")
for r in results:
    print(f"{r['document']} (page {r['page']}): {r['score']:.3f}")
```

> **Key Advantage（PDF 原文强调）**：ColPali 无需文本提取就能找到相关内容，保留了图表、公式和布局信息——这些在传统管线中全部丢失。

---

### 6. 统一多模态搜索系统（Unified Search Engine）

#### 6.1 架构思想

**核心洞察**：我们不需要一个"万能模型"来处理所有类型——更好的策略是让每种内容类型由最擅长的专家模型处理，然后在搜索层面做整合。

**直觉建立**：想象一家大型医院的分诊台。患者来了（查询进来了），分诊台不会把每个患者都送给一个"全科万能医生"——而是根据症状描述路由到对应的专科：骨科（CLIP 处理图片）、内科（Matryoshka 处理文本）、影像科（ColPali 处理文档）。每个专科医生用自己最擅长的诊断方法给出结论，最后分诊台汇总各科结果给患者一个综合报告。`UnifiedSearchEngine` 就是这个分诊台：它不试图成为全能模型，而是做好路由和结果整合。这种架构的优势是每个"专科"可以独立升级（换更好的 CLIP 版本不影响 ColPali），劣势是跨模态的分数不直接可比（骨科打 8 分和内科打 8 分含义不同）——生产环境需要做分数归一化。

```
                         UnifiedSearchEngine
                        /        |          \
                  CLIP          Matryoshka      ColPali
                (图片搜索)     (文本搜索)     (文档搜索)
                    |              |              |
               photos 表      text_items 表   documents 表
               vector(512)    vector(128/256/768)  BYTEA
```

#### 6.2 完整实现

`UnifiedSearchEngine` 类整合了前三部分的所有实现。

```python
class UnifiedSearchEngine:
    """
    Combines three search approaches:
    - CLIP for photos (Part 1)
    - Matryoshka for text (Part 2)
    - ColPali for documents (Part 3)
    """

    def __init__(self, db_connection_string):
        self.device = get_device()  # Use our utility function
        self.conn = psycopg2.connect(db_connection_string)
        self.cur = self.conn.cursor()

        # Load all models (details in Parts 1-3)
        self._load_models()
        self._initialize_database()

    def _load_models(self):
        """Load CLIP, Matryoshka, and ColPali models"""
        print(f"Initializing models on {self.device}...")

        # CLIP for images
        self.clip_model = CLIPModel.from_pretrained(
            "openai/clip-vit-base-patch32"
        ).to(self.device)
        self.clip_processor = CLIPProcessor.from_pretrained(
            "openai/clip-vit-base-patch32"
        )

        # ColPali for documents
        self.colpali_model = ColPali.from_pretrained(
            "vidore/colpali-v1.2",
            torch_dtype=torch.float16 if self.device != "cpu" else torch.float32
        ).to(self.device)
        self.colpali_processor = ColPaliProcessor.from_pretrained(
            "vidore/colpali-v1.2"
        )

        # Matryoshka for text
        self.text_model = SentenceTransformer("nomic-ai/nomic-embed-text-v1.5")

    def _initialize_database(self):
        """Create tables for all content types"""
        # Photos table
        self.cur.execute("""
            CREATE TABLE IF NOT EXISTS photos (
                id SERIAL PRIMARY KEY,
                file_path TEXT,
                embedding vector(512)
            )
        """)
        self.cur.execute("""
            CREATE INDEX IF NOT EXISTS photos_embedding_idx
            ON photos USING hnsw (embedding vector_cosine_ops)
        """)

        # Documents table
        self.cur.execute("""
            CREATE TABLE IF NOT EXISTS documents (
                id SERIAL PRIMARY KEY,
                file_path TEXT,
                page_num INTEGER,
                embedding_data BYTEA
            )
        """)

        # Text table
        self.cur.execute("""
            CREATE TABLE IF NOT EXISTS text_items (
                id SERIAL PRIMARY KEY,
                content TEXT,
                embedding_128 vector(128),
                embedding_256 vector(256),
                embedding_768 vector(768)
            )
        """)
        for dim in [128, 256, 768]:
            self.cur.execute(f"""
                CREATE INDEX IF NOT EXISTS text_embedding_{dim}_idx
                ON text_items USING hnsw (embedding_{dim} vector_cosine_ops)
            """)
        self.conn.commit()
```

**索引方法**（三种内容类型各一个）：

```python
    def index_photo(self, image_path):
        """Index photo using CLIP"""
        image = Image.open(image_path)
        inputs = self.clip_processor(images=image, return_tensors="pt")
        inputs = {k: v.to(self.device) for k, v in inputs.items()}
        with torch.no_grad():
            embedding = self.clip_model.get_image_features(**inputs)
            embedding = embedding / embedding.norm(dim=-1, keepdim=True)
        self.cur.execute(
            "INSERT INTO photos (file_path, embedding) VALUES (%s, %s)",
            (str(image_path), embedding[0].cpu().numpy().tolist())
        )
        self.conn.commit()

    def index_document(self, pdf_path):
        """Index PDF using ColPali"""
        images = convert_from_path(pdf_path, dpi=150)
        for page_num, image in enumerate(images):
            inputs = self.colpali_processor.process_images([image]).to(self.device)
            with torch.no_grad():
                embeddings = self.colpali_model(**inputs)
            buffer = io.BytesIO()
            np.save(buffer, embeddings[0].cpu().numpy())
            self.cur.execute("""
                INSERT INTO documents (file_path, page_num, embedding_data)
                VALUES (%s, %s, %s)
            """, (str(pdf_path), page_num, buffer.getvalue()))
        self.conn.commit()

    def index_text(self, text_content):
        """Index text using Matryoshka"""
        full_embedding = self.text_model.encode(text_content)
        self.cur.execute("""
            INSERT INTO text_items (content, embedding_128, embedding_256, embedding_768)
            VALUES (%s, %s, %s, %s)
        """, (
            text_content,
            full_embedding[:128].tolist(),
            full_embedding[:256].tolist(),
            full_embedding.tolist()
        ))
        self.conn.commit()
```

**搜索方法**（各走各的专用逻辑）：

```python
    def search_photos(self, query_text, top_k=5):
        """Search photos with CLIP"""
        inputs = self.clip_processor(text=[query_text], return_tensors="pt")
        inputs = {k: v.to(self.device) for k, v in inputs.items()}
        with torch.no_grad():
            query_embedding = self.clip_model.get_text_features(**inputs)
            query_embedding = query_embedding / query_embedding.norm(dim=-1, keepdim=True)
        query_np = query_embedding[0].cpu().numpy()
        self.cur.execute("""
            SELECT file_path, 1 - (embedding <=> %s::vector) AS similarity
            FROM photos
            ORDER BY embedding <=> %s::vector
            LIMIT %s
        """, (query_np.tolist(), query_np.tolist(), top_k))
        return [{"path": row[0], "score": row[1], "type": "photo"}
                for row in self.cur.fetchall()]

    def search_documents(self, query_text, top_k=5):
        """Search documents with ColPali + MaxSim"""
        inputs = self.colpali_processor.process_queries([query_text]).to(self.device)
        with torch.no_grad():
            query_embeddings = self.colpali_model(**inputs)[0].cpu().numpy()
        self.cur.execute("SELECT id, file_path, page_num, embedding_data FROM documents")
        results = []
        for row in self.cur.fetchall():
            doc_id, file_path, page_num, embedding_data = row
            buffer = io.BytesIO(embedding_data)
            doc_embeddings = np.load(buffer)
            score = maxsim_score(query_embeddings, doc_embeddings)
            results.append({
                "path": file_path, "page": page_num,
                "score": score, "type": "document"
            })
        results.sort(key=lambda x: x["score"], reverse=True)
        return results[:top_k]

    def search_text(self, query_text, top_k=10):
        """Search text with Matryoshka tiered retrieval"""
        query_embedding = self.text_model.encode(query_text)
        # Stage 1: 128D
        query_128 = query_embedding[:128].tolist()
        self.cur.execute("""
            SELECT id FROM text_items
            ORDER BY embedding_128 <=> %s::vector LIMIT 100
        """, (query_128,))
        stage1_ids = [row[0] for row in self.cur.fetchall()]
        # Stage 2: 256D
        query_256 = query_embedding[:256].tolist()
        self.cur.execute("""
            SELECT id FROM text_items
            WHERE id = ANY(%s)
            ORDER BY embedding_256 <=> %s::vector LIMIT 20
        """, (stage1_ids, query_256))
        stage2_ids = [row[0] for row in self.cur.fetchall()]
        # Stage 3: 768D
        query_768 = query_embedding.tolist()
        self.cur.execute("""
            SELECT content, 1 - (embedding_768 <=> %s::vector) AS similarity
            FROM text_items
            WHERE id = ANY(%s)
            ORDER BY embedding_768 <=> %s::vector LIMIT %s
        """, (query_768, stage2_ids, query_768, top_k))
        return [{"content": row[0], "score": row[1], "type": "text"}
                for row in self.cur.fetchall()]
```

**统一搜索入口**：

```python
    def search_all(self, query_text, top_k=5):
        """Search across all content types"""
        photo_results = self.search_photos(query_text, top_k)
        doc_results = self.search_documents(query_text, top_k)
        text_results = self.search_text(query_text, top_k)

        # Merge and sort by score
        all_results = photo_results + doc_results + text_results
        all_results.sort(key=lambda x: x["score"], reverse=True)
        return all_results[:top_k]
```

> **注意**：`search_all` 是把三种搜索的结果直接按分数排序合并。但要注意，三种模型的分数量纲不一定一致（CLIP 是余弦相似度、ColPali 是 MaxSim 总分、Matryoshka 也是余弦相似度），直接比较可能不够精确。生产环境中可能需要做分数归一化。

> **常见误用**：在 `search_all` 中直接按原始分数排序跨模态结果，认为"分数越高越相关"。实际上 ColPali 的 MaxSim 分数是多 token 求和的结果，值域远大于余弦相似度的 [-1, 1] 范围，导致 ColPali 的结果几乎总是排在最前面，文本和图片结果被淹没。正确做法：在合并前对每种模态的分数做 z-score 归一化或 min-max 归一化，或者用 Reciprocal Rank Fusion（RRF）按排名而非分数做融合。

**使用示例**：

```python
# Initialize
engine = UnifiedSearchEngine("dbname=unified_search user=postgres password=password")

# Index different content types
for photo in Path("photos").glob("*.jpg"):
    engine.index_photo(photo)

for doc in Path("documents").glob("*.pdf"):
    engine.index_document(doc)

for text in ["ML fundamentals", "Neural network basics"]:
    engine.index_text(text)

# Search across everything
results = engine.search_all("machine learning", top_k=5)
for r in results:
    if r['type'] == 'photo':
        print(f"[PHOTO] {r['path']}: {r['score']:.3f}")
    elif r['type'] == 'document':
        print(f"[DOC] {r['path']} p.{r['page']}: {r['score']:.3f}")
    else:
        print(f"[TEXT] {r['content'][:50]}...: {r['score']:.3f}")
```

---

### 7. 生产环境注意事项

#### 7.1 批量处理

逐条索引太慢，生产环境要做批处理。PDF 给出了 CLIP 图片批量索引的示例：

```python
def index_photos_batch(engine, image_paths, batch_size=32):
    """Batch process photos for faster indexing"""
    for i in range(0, len(image_paths), batch_size):
        batch = image_paths[i:i+batch_size]
        images = [Image.open(p) for p in batch]
        inputs = engine.clip_processor(images=images, return_tensors="pt", padding=True)
        inputs = {k: v.to(engine.device) for k, v in inputs.items()}

        with torch.no_grad():
            embeddings = engine.clip_model.get_image_features(**inputs)
            embeddings = embeddings / embeddings.norm(dim=-1, keepdim=True)

        for path, embedding in zip(batch, embeddings):
            engine.cur.execute(
                "INSERT INTO photos (file_path, embedding) VALUES (%s, %s)",
                (str(path), embedding.cpu().numpy().tolist())
            )
        engine.conn.commit()
```

核心是把 `batch_size` 张图一次送进模型，利用 GPU 并行提高吞吐。

#### 7.2 查询缓存

高频查询可以缓存嵌入结果，避免重复编码。

```python
from functools import lru_cache

class CachedSearchEngine(UnifiedSearchEngine):
    @lru_cache(maxsize=100)
    def _cached_text_embedding(self, query_text):
        """Cache text embeddings for repeated queries"""
        return self.text_model.encode(query_text)
```

> `lru_cache` 在内存中缓存最近 100 个不同查询的嵌入结果。对于搜索热词场景效果显著。

#### 7.3 模型选择与参数权衡

**CLIP 变体**：

| 变体 | 特点 |
|------|------|
| `clip-vit-base-patch32` | 平衡选择，512 维嵌入 |
| `clip-vit-large-patch14` | 更高精度，但更慢 |

**ColPali 变体**：

| 变体 | 特点 |
|------|------|
| `vidore/colpali-v1.2` | 最高精度 |
| `vidore/colsmol-v0.1` | 更快推理、更小体积 |

**PDF 处理 DPI 选择**：

| DPI | 速度 | 质量 |
|-----|------|------|
| 72 | 快 | 低 |
| 150 | 适中（推荐） | 适中 |
| 300 | 慢 | 高 |

#### 7.4 智能查询路由

根据查询内容的特征，自动把请求路由到最合适的搜索引擎，而不是每次都搜全部。

```python
def intelligent_search(engine, query_text, top_k=5):
    """Route query based on keywords and characteristics"""
    query_lower = query_text.lower()

    if any(word in query_lower for word in ['photo', 'image', 'picture']):
        return engine.search_photos(query_text, top_k)
    elif any(word in query_lower for word in ['pdf', 'document', 'paper']):
        return engine.search_documents(query_text, top_k)
    elif len(query_text.split()) > 10:
        return engine.search_text(query_text, top_k)
    else:
        return engine.search_all(query_text, top_k)
```

路由策略：
- 查询中含 photo/image/picture → 走 CLIP 图片搜索
- 查询中含 pdf/document/paper → 走 ColPali 文档搜索
- 查询文本较长（>10 词） → 走 Matryoshka 文本搜索
- 其他 → 全搜索

> 这是简单的基于关键词的路由。生产环境可以用更智能的方式，比如训练一个轻量分类器来判断查询意图。

---

### 8. 三种技术综合对比

| 维度 | CLIP | Matryoshka Embeddings | ColPali |
|------|------|----------------------|---------|
| **目标内容** | 图片 | 文本 | PDF 文档 |
| **输入类型** | 图像 + 文本 | 文本 | PDF 页面（当图片处理） |
| **嵌入类型** | 单向量 (512D) | 单向量 (可变: 64-768D) | 多向量 (每页数百个 128D) |
| **搜索机制** | 余弦相似度 | 余弦相似度（分层） | MaxSim (late interaction) |
| **核心优势** | 零样本图文跨模态匹配 | 灵活的精度-效率权衡 | 无需文本提取，保留视觉布局 |
| **存储/条目** | ~2KB | 0.25-3KB（取决于维度） | ~512KB/页 |
| **查询延迟** | 中等 | 低（低维阶段极快） | 高 (100-500ms) |
| **GPU 需求** | 需要 | 编码时需要 | 强需求（模型大） |
| **数据库支持** | pgvector `vector(512)` | pgvector 多列 | BYTEA 序列化 |
| **典型场景** | 照片库搜索、电商、内容审核 | 大规模文档检索、移动端、实时搜索 | 科学论文、财报、技术文档 |
| **关键权衡** | 存储换灵活性 | 微小精度损失换巨大效率提升 | 存储和速度换文档理解力 |
| **模型示例** | `openai/clip-vit-base-patch32` | `nomic-ai/nomic-embed-text-v1.5` | `vidore/colpali-v1.2` |

---

### 9. 全章关键收获（Key Takeaways）

PDF 最后总结了六条关键经验，值得逐条记住：

1. **针对不同内容类型使用专用模型**——one size doesn't fit all，不要试图用一个模型解决所有问题
2. **归一化嵌入后再做余弦相似度**——确保评分一致性
3. **利用硬件加速（MPS/CUDA）**——不加速的话这些模型实际不可用
4. **对大规模场景应用分层搜索策略**——Matryoshka 的三阶段检索是典型范式
5. **尽可能批量处理**——最大化吞吐
6. **根据场景在精度、速度、存储之间做取舍**——没有银弹，只有 tradeoff

> **一句话总结**：CLIP 用存储换灵活性，Matryoshka 用微小精度损失换巨大效率提升，ColPali 用存储和速度换前所未有的文档理解力。三者合一，构成现代搜索系统的强力工具箱。

---

## 重点标记

1. **CLIP 的核心突破**：把图像和文本映射到同一个 512 维空间，实现零样本跨模态搜索，无需为特定任务训练
2. **Matryoshka 的嵌套特性**：前 N 维就是有效的 N 维嵌入（`embedding_768d[:, :128] == embedding_128d`），一次编码多处复用
3. **分层搜索的效率增益**：128D 粗筛 → 256D 精筛 → 768D 精排，百万文档规模计算量减少 100x，召回率 98%+
4. **ColPali 跳过 OCR**：直接把 PDF 页面当图片理解，保留表格、图表、公式等传统管线必然丢失的信息
5. **MaxSim 评分机制**：为每个 query token 找到文档中最匹配的 patch，然后求和——这是 ColPali late interaction 的核心
6. **ColPali 的存储代价**：每页约 512KB（传统嵌入的 100-500 倍），是选择 ColPali 时必须权衡的主要成本
7. **统一搜索引擎的设计哲学**：不追求一个万能模型，而是专家模型各司其职 + 路由层统一调度
8. **生产优化三板斧**：批量处理提吞吐、查询缓存减重复、智能路由降开销
9. **嵌入归一化是必须步骤**：所有使用余弦相似度的场景都要先归一化，否则结果不可靠
10. **模型变体选择**：CLIP 有 base/large、ColPali 有 v1.2/colsmol，根据精度-速度需求选择；PDF 渲染建议 150 DPI 作为默认值

---

## 自测：你真的理解了吗？

> 以下问题考的是理解，不是记忆。试着先思考再看答案。

**Q1**：你的团队在构建一个内部知识库搜索系统，内容包括：10 万张产品照片、50 万篇技术文档（纯文本）、2 万份 PDF 格式的财务报表（大量表格和图表）。存储预算有限。你会为每种内容类型分别选择 CLIP、Matryoshka、ColPali 中的哪一个？如果存储预算削减 50%，你会优先对哪种内容的嵌入方案做调整？为什么？

**Q2**：一个同事用 `sentence-transformers` 的普通模型（非 Matryoshka 训练的）生成了 768 维嵌入，然后直接用 `embedding[:128]` 截断到 128 维做搜索，发现召回率从 92% 暴跌到 41%。他问你为什么 Matryoshka 嵌入截断后能保持 95% 精度而他的不行。你怎么解释？他现在的 768 维嵌入已经入库了 100 万条，有没有什么补救方案不需要全部重新编码？

**Q3**：在 Matryoshka 分层搜索中，Stage 1 用 128D 筛选 1000 个候选，Stage 2 用 256D 筛选 100 个，Stage 3 用 768D 精排出 top 10。如果 Stage 1 的 1000 个候选中漏掉了真正最相关的文档（即召回失败），后面两个阶段能补救吗？这种风险在什么条件下最大？你会如何缓解？

**Q4**：你用 ColPali 索引了一批科学论文，搜索"Figure 3 showing the comparison of model accuracy"时，ColPali 准确地找到了包含该图表的页面。但当你对同一批论文用传统的 OCR + 文本嵌入方案搜索同样的查询时，返回结果完全不相关。为什么两种方法的差异如此大？ColPali 在什么类型的查询上反而不如传统方案？

**Q5**：你的 `UnifiedSearchEngine.search_all()` 在线上运行后，发现返回的 top 5 结果中几乎全是 PDF 文档（ColPali 结果），图片和文本结果很少出现。分析日志发现 ColPali 的 MaxSim 分数范围是 15-45，而 CLIP 和 Matryoshka 的余弦相似度范围是 0.3-0.9。问题出在哪里？你会如何修改 `search_all()` 的结果融合逻辑来解决这个问题？
