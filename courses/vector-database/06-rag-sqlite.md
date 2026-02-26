# 模块六：RAG 系统基础（SQLite VSS + Ollama）

> 对应 PDF: 06_RAG_System_with_SQLite_VSS_and_Ollama.pdf，第 1-2 页

---

## 概念地图

- **核心概念** (必须内化): RAG 架构的运作原理（检索 + 生成的协同）、混合搜索的融合策略（语义搜索 + 关键词搜索加权融合）、Prompt 防幻觉设计（约束 LLM 只用检索内容）
- **实操要点** (动手时需要): 文本分块参数选择（chunk_size/overlap 的权衡）、VSS + FTS5 双索引搭建、Ollama API 调用与 temperature 设置
- **背景知识** (扩展理解): BM25 算法的排序逻辑、RAG 从 demo 到生产的关键差距（增量更新/版本管理/监控）、高级 RAG 模式（多跳推理/交叉编码器重排序）

---

## 概念讲解

### 1. RAG 系统解决什么问题

**定义**：检索增强生成（Retrieval-Augmented Generation, RAG）是一种将信息检索与大语言模型（LLM）的文本生成能力结合起来的系统架构。简单说，就是先从你的数据库里把相关内容找出来，再让 LLM 基于这些内容来回答问题。

**核心思想**：LLM 有一个根本性局限——它的知识在训练那一刻就冻结了（knowledge is frozen at training time），既无法访问私有数据，也无法获取最新信息。RAG 通过给 LLM "外挂"一个检索机制，让它能用最新的、领域专属的知识来回答问题。

**直觉建立**：把 LLM 想象成一个极其聪明但"闭关修炼"了好几年的专家。他知识渊博，表达能力强，但有两个致命缺陷：(1) 他不知道"闭关"期间外面发生了什么；(2) 他从没看过你公司的内部文档。现在你问他"我们公司最新的退货政策是什么？"他要么瞎编（幻觉），要么老实说不知道。RAG 就像是给这位专家配了一个助理——每次有人提问，助理先去档案柜里把最相关的文件翻出来，摊在专家面前，然后专家基于这些文件来回答。助理负责"找对资料"（检索），专家负责"说人话"（生成）。关键约束是：专家只能基于助理翻出来的资料回答，不能自由发挥——这就是防幻觉的核心机制。注意边界：如果助理找错了资料（检索不准），专家再聪明也只能基于错误的资料给出错误的答案——所以 RAG 系统的质量上限取决于检索质量，而不仅仅是 LLM 的能力。

**为什么重要/有效**：
- **防止幻觉（Hallucination）**：LLM 直接回答时可能编造事实，RAG 把答案"锚定"在真实数据上，确保事实准确
- **访问私有数据**：企业内部文档、个人笔记等不在 LLM 训练集里的内容，通过 RAG 也能被利用
- **时效性**：训练数据有截止日期，RAG 让 LLM 能回答关于最新信息的问题

**示例**：本章构建的是一个基于 Reddit 内容的问答系统。当用户提问时，系统会：
1. 在存储的 Reddit 帖子中搜索最相关的信息
2. 检索出最佳匹配的内容分块
3. 把这些上下文提供给 LLM
4. LLM **仅基于检索到的信息** 生成自然语言回答

**适用场景**：
- 企业知识库问答（内部文档、FAQ）
- 技术文档助手（API 文档、代码库搜索）
- 客户支持系统（基于历史工单回答新问题）
- 任何需要让 LLM 回答"它没见过的数据"的场景

---

### 2. 系统架构：五大组件

**定义**：本章的 RAG 系统由五个核心组件协同工作，每个组件各司其职。

**直觉建立**：把整个 RAG 系统想象成一家餐厅的运作流程。**向量数据库层**（SQLite VSS）是食材仓库——所有食材按种类和特征编号存放。**嵌入引擎**（SentenceTransformers）是食材编码员——每样食材进库时，编码员给它贴上一个多维度的"味道指纹"标签。**混合搜索系统**（VSS + FTS5）是采购员——顾客点菜时，采购员同时按"味道相似度"（语义搜索）和"食材名称"（关键词搜索）两条路去仓库找食材，然后把两路结果综合排序。**LLM 集成层**（Ollama）是大厨——拿到采购员找来的食材后，按菜谱（Prompt）做出成品菜。**RAG 管道编排器**是餐厅经理——协调从顾客点菜到上菜的全流程。注意：这五个组件缺一不可，但你可以替换其中任何一个的具体实现（换数据库、换嵌入模型、换 LLM），架构模式不变。

**五大组件及职责**：

| 组件 | 技术选型 | 职责 |
|------|----------|------|
| **向量数据库层**（Vector Database Layer） | SQLite VSS | 存储内容分块及其嵌入向量，支持快速相似性搜索 |
| **嵌入引擎**（Embedding Engine） | SentenceTransformers | 将文本转换为捕获语义含义的稠密向量表示 |
| **混合搜索系统**（Hybrid Search System） | VSS + FTS5 | 结合语义向量搜索和传统关键词搜索，实现最优检索 |
| **LLM 集成层**（LLM Integration） | Ollama | 提供本地 LLM 推理，生成自然语言回答 |
| **RAG 管道编排器**（RAG Pipeline Orchestrator） | Python 自定义 | 协调检索和生成的完整流程 |

**数据流路径**：

系统有两条主要数据流——**写入流**和**查询流**：

```
写入流（Ingestion）:
Content ingestion → Text chunking → Embedding generation → Vector storage

查询流（Query）:
Query processing → Hybrid search → Context retrieval → Prompt construction → LLM generation
```

> **关键理解**：写入流是"一次性"把数据灌进去的过程（离线），查询流是用户每次提问时实时执行的过程（在线）。两条路用的是同一个 Embedding 模型——这一点至关重要，因为查询向量和存储向量必须在同一个向量空间才能做相似性比较。

**适用场景**：
- 这种"五层架构"是 RAG 系统的经典设计，无论你用什么具体技术栈，概念结构都是类似的
- SQLite + Ollama 的组合特别适合原型开发、个人项目、隐私敏感场景（全部本地运行，无需云服务）

---

### 3. 数据库基础与向量支持（6.1 节）

#### 3.1 技术栈与依赖

PDF 开头的 import 部分揭示了完整的技术栈：

```python
# Chapter 6: Minimal RAG System with SQLite VSS and Ollama
# Simplified version demonstrating core Retrieval-Augmented Generation
import sqlite3
import requests
import json
import time
import hashlib
from sentence_transformers import SentenceTransformer
from typing import List, Dict, Optional
```

**关键点**：
- `sqlite3`：Python 内置，无需额外安装，提供数据库存储
- `requests`：用于调用 Ollama 的 HTTP API
- `SentenceTransformer`：嵌入模型的核心库
- 其余都是标准库（`json`、`time`、`hashlib`、`typing`），说明这个系统的外部依赖非常少

#### 3.2 数据库初始化：加载 VSS 扩展

```python
def setup_database(db_path='reddit_rag.db'):
    """Setup SQLite with VSS extension and create tables"""
    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row  # Return rows as dictionaries

    # Enable and load VSS extension
    conn.enable_load_extension(True)
    try:
        conn.load_extension("./sqlite-vss0")
        print("VSS extension loaded successfully")
    except Exception as e:
        print(f"Error loading VSS: {e}")
        print("Download from: https://github.com/asg017/sqlite-vss/releases")
        raise
```

**这段代码做了什么**：连接 SQLite 数据库，加载 VSS（Vector Similarity Search）扩展。

**关键点**：
- `conn.row_factory = sqlite3.Row`：让查询结果以字典形式返回而不是元组，代码可读性更好（用 `row['column_name']` 而不是 `row[0]`）
- **VSS 扩展是核心**：没有它，SQLite 就是个普通的关系数据库，无法做向量相似性搜索。扩展文件 `sqlite-vss0` 需要从 GitHub 单独下载
- `enable_load_extension(True)`：SQLite 默认禁止加载扩展（安全考虑），必须显式开启

> **注意**：VSS 扩展文件需要放在项目目录下（`./sqlite-vss0`），或者修改路径指向你实际的安装位置。

#### 3.3 Schema 设计：两张核心表

**posts 表**——存储原始 Reddit 帖子：

```python
    cursor = conn.cursor()

    # Create posts table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS posts (
            post_id TEXT PRIMARY KEY,
            title TEXT,
            content TEXT,
            subreddit TEXT,
            author TEXT,
            score INTEGER
        )
    """)
```

**这段代码做了什么**：创建帖子表，存放原始内容和元数据（子版块、作者、评分等）。

**content_chunks 表**——存储文本分块和向量：

```python
    # Create chunks table with vector support
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS content_chunks (
            chunk_id INTEGER PRIMARY KEY,
            post_id TEXT,
            chunk_index INTEGER,
            content TEXT,
            chunk_vector BLOB,  -- Store as BLOB for VSS
            FOREIGN KEY (post_id) REFERENCES posts(post_id)
        )
    """)
```

**这段代码做了什么**：创建分块表，每个分块都有对应的向量嵌入（存为 BLOB 格式），通过外键关联到原始帖子。

**关键点**：
- **两层设计**：`posts` 是"源数据真相"（source of truth），`content_chunks` 是"检索单元"。检索时搜 chunks，展示时追溯到原始 post
- `chunk_vector BLOB`：嵌入向量以二进制大对象（BLOB）的形式存储，这是 SQLite 存储向量的标准方式
- `chunk_index`：记录分块在原文中的顺序，方便后续重组或展示上下文
- `FOREIGN KEY`：保持了 chunk 到 post 的引用完整性

#### 3.4 创建搜索索引：VSS 虚拟表 + FTS5

```python
    # Create VSS virtual table for vector search
    cursor.execute("""
        CREATE VIRTUAL TABLE IF NOT EXISTS chunk_vss USING vss0(
            chunk_vector(384),
            chunk_id INTEGER
        )
    """)

    # Create FTS5 for keyword search
    cursor.execute("""
        CREATE VIRTUAL TABLE IF NOT EXISTS chunks_fts USING fts5(
            content,
            content='content_chunks',
            content_rowid='chunk_id'
        )
    """)

    conn.commit()
    return conn
```

**这段代码做了什么**：创建两个虚拟表——一个用于向量相似性搜索（VSS），一个用于全文关键词搜索（FTS5）。

**关键点**：
- **`chunk_vss`**：VSS 虚拟表，`chunk_vector(384)` 表示向量维度是 384——这必须和 Embedding 模型的输出维度一致（`all-MiniLM-L6-v2` 的输出就是 384 维）
- **`chunks_fts`**：FTS5（Full-Text Search 5）虚拟表，SQLite 内置的全文搜索引擎，使用 BM25 算法进行关键词排序
- `content='content_chunks'` 和 `content_rowid='chunk_id'`：告诉 FTS5 它的数据来源是 `content_chunks` 表，以 `chunk_id` 作为行标识
- **两个索引并存**是混合搜索的基础——语义搜索走 VSS，关键词搜索走 FTS5

> **为什么要两种搜索？** 语义搜索擅长理解"意思相近但词汇不同"的查询（比如"ML"和"机器学习"），但可能漏掉精确的专有名词匹配；关键词搜索（BM25）擅长精确匹配，但不理解同义词。两者互补才能达到最优检索效果。

---

### 4. 文本处理与嵌入生成（6.2 节）

#### 4.1 Embedding 模型管理：单例模式

```python
# Initialize embedding model globally
embedding_model = None

def get_embedding_model():
    """Get or initialize the embedding model"""
    global embedding_model
    if embedding_model is None:
        embedding_model = SentenceTransformer('all-MiniLM-L6-v2')
        print(f"Loaded embedding model (dimension: {embedding_model.get_sentence_embedding_dimension()})")
    return embedding_model
```

**这段代码做了什么**：用全局变量 + 懒加载实现单例模式（Singleton Pattern），确保 Embedding 模型只加载一次。

**关键点**：
- **为什么用单例？** 模型加载需要时间和内存，如果每次嵌入都重新加载，性能会很差
- **`all-MiniLM-L6-v2`** 这个模型的特点：
  - 输出 384 维向量（和前面 VSS 虚拟表的 384 对应）
  - 体积小，CPU 上也能高效运行
  - 在质量和速度之间取得了不错的平衡
  - 是 Sentence Transformers 中最常用的通用模型之一

> **坑提醒**：如果你换了 Embedding 模型（比如换成输出 768 维的模型），VSS 虚拟表的维度声明也必须同步修改为 768，否则会报错或搜索结果不正确。

#### 4.2 智能文本分块（Chunking）

```python
def chunk_text(text, chunk_size=200, overlap=50):
    """Simple text chunking by word count"""
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size - overlap):
        chunk = ' '.join(words[i:i + chunk_size])
        if chunk:
            chunks.append(chunk)
    return chunks
```

**这段代码做了什么**：将长文本按词数切分为多个重叠的小块。每块 200 个词，相邻块之间重叠 50 个词。

**关键点**：
- **为什么要分块？** LLM 的上下文窗口有限，不可能把整篇文章塞进去。而且检索时，小块更精确——如果一篇文章有 10 段，只有第 3 段和问题相关，分块后就只检索到第 3 段，不会把无关内容也塞给 LLM
- **`chunk_size=200`**：200 个词大约是一个中等段落的长度，足够包含一个完整的想法
- **`overlap=50`**：重叠是为了防止重要信息被切断在两个块的边界上。比如一句关键的话刚好跨越第 200 词和第 201 词，没有重叠就会被分到两个块里，丢失上下文
- **滑动窗口步长**：`chunk_size - overlap = 150`，即每次向前移动 150 个词

> **PDF 明确指出的权衡**：分块太小会丢失上下文（lose context），太大会稀释相关性（dilute relevance）。200 词 + 50 词重叠是一个兼顾上下文完整性和检索精度的经验值。

> **进阶思考**：这里用的是"按词数切分"（word-count chunking），是最简单的策略。后面"扩展方向"部分会提到更高级的"语义分块"（semantic chunking），即根据句子嵌入和段落边界来智能切分，让相关的内容保持在同一个块里。

> **常见误用**：把 chunk_size 设得特别大（如 1000 词）以为能"保留更多上下文"。实际上分块太大会严重稀释相关性——一个 1000 词的块里可能只有 2 句话和用户问题相关，其余 900 词都是噪音，不仅浪费 LLM 的上下文窗口，还可能让 LLM 被无关信息误导。反过来，chunk_size 设得太小（如 20 词）会丢失上下文，导致每个块都像断章取义。经验法则：200-500 词 + 10%-25% 的重叠率是合理起点。

#### 4.3 存储流程：从原始帖子到可搜索的向量

这是一个完整的数据灌入（ingestion）函数，串联了"存帖子 → 分块 → 生成嵌入 → 写入索引"的全流程：

```python
def store_post_with_chunks(conn, post_data):
    """Store a post and its chunks with embeddings"""
    cursor = conn.cursor()

    # Store post
    cursor.execute("""
        INSERT OR REPLACE INTO posts (post_id, title, content, subreddit, author, score)
        VALUES (?, ?, ?, ?, ?, ?)
    """, (
        post_data['post_id'],
        post_data['title'],
        post_data['content'],
        post_data['subreddit'],
        post_data['author'],
        post_data.get('score', 0)
    ))
```

**第一步**：存储原始帖子。`INSERT OR REPLACE` 模式能优雅地处理重复灌入（幂等操作）。

```python
    # Create chunks from title + content
    full_text = f"{post_data['title']}\n\n{post_data['content']}"
    chunks = chunk_text(full_text)

    # Get embedding model
    model = get_embedding_model()
```

**第二步**：将标题和内容拼接后再分块。

> **为什么要拼接标题？** 标题通常包含帖子的核心主题，如果只对 content 分块，丢掉标题会损失重要上下文。

```python
    # Store each chunk with embedding
    for idx, chunk_content in enumerate(chunks):
        # Generate embedding
        embedding = model.encode(chunk_content)

        # Store chunk
        cursor.execute("""
            INSERT INTO content_chunks (post_id, chunk_index, content, chunk_vector)
            VALUES (?, ?, ?, ?)
        """, (
            post_data['post_id'],
            idx,
            chunk_content,
            embedding.tobytes()
        ))

        chunk_id = cursor.lastrowid
```

**第三步**：逐块生成嵌入向量，存入 `content_chunks` 表。`embedding.tobytes()` 将 numpy 数组转为二进制格式以 BLOB 存储。

```python
        # Add to VSS index
        vector_str = f"vss_create_vector({','.join(map(str, embedding))})"
        cursor.execute(f"""
            INSERT INTO chunk_vss (chunk_id, chunk_vector)
            VALUES (?, {vector_str})
        """, (chunk_id,))

        # Add to FTS index
        cursor.execute("""
            INSERT INTO chunks_fts (rowid, content)
            VALUES (?, ?)
        """, (chunk_id, chunk_content))

    conn.commit()
    print(f"Stored post {post_data['post_id']} with {len(chunks)} chunks")
```

**第四步**：同时写入两个搜索索引——VSS（向量搜索）和 FTS5（全文搜索）。

**关键点**：
- `vss_create_vector(...)` 是 VSS 扩展提供的函数，将逗号分隔的浮点数转为 VSS 内部的向量格式
- 每个 chunk 写入三个地方：`content_chunks`（数据本体）、`chunk_vss`（向量索引）、`chunks_fts`（全文索引）
- **双索引写入**是后续混合搜索的前提

---

### 5. 混合搜索实现（6.3 节）

**定义**：混合搜索（Hybrid Search）是同时使用语义向量搜索和关键词搜索，然后融合两者的结果来获得更好的检索效果。

**核心思想**：语义搜索理解"意思"，关键词搜索匹配"词汇"，两者互补。

**直觉建立**：假设你在一个大型购物中心里找"保暖的冬季外套"。**语义搜索**就像一个懂时尚的导购——你说"保暖的冬季外套"，她会带你去看羽绒服、棉衣、甚至冲锋衣，因为她理解"保暖"和"冬季"的含义，即使商品标签上没写这些词。但她可能会漏掉一件标签上明确写着"Canada Goose 冬季外套"的商品，因为她更关注语义匹配而非品牌名。**关键词搜索**（BM25）就像一台只认字的搜索终端——你输入"冬季外套"，它精确找出所有描述里包含这四个字的商品，一个不漏，但它不会理解"羽绒服"和"冬季外套"是一回事。**混合搜索**就是同时派出导购和搜索终端，把两边的结果合在一起，导购找到的结果占 70% 权重，搜索终端的占 30%——既不漏掉语义相关的结果，也不错过精确匹配的结果。注意边界：权重 7:3 是经验值，如果你的场景需要精确匹配（如法律条款搜索），应该提高关键词权重。

#### 5.1 搜索入口与查询嵌入

```python
def hybrid_search(conn, query_text, limit=5, semantic_weight=0.7):
    """Perform hybrid search combining vector and keyword search"""
    cursor = conn.cursor()
    model = get_embedding_model()

    # Generate query embedding
    query_embedding = model.encode(query_text)
    vector_str = f"vss_create_vector({','.join(map(str, query_embedding))})"
```

**这段代码做了什么**：接收用户查询文本，将其转换为向量，准备进行双路搜索。

**关键点**：
- `semantic_weight=0.7`：语义搜索权重 70%，关键词搜索权重 30%。这意味着系统更倾向于"理解意思"而非"匹配原词"
- 查询用的是和存储时**完全相同的 Embedding 模型**——这是向量搜索的基本要求

#### 5.2 语义搜索组件（VSS cosine distance）

```python
    # Semantic search using VSS
    cursor.execute(f"""
        SELECT
            c.chunk_id,
            c.post_id,
            c.content,
            vss_cosine_distance(v.chunk_vector, {vector_str}) as distance
        FROM chunk_vss v
        JOIN content_chunks c ON v.chunk_id = c.chunk_id
        ORDER BY distance ASC
        LIMIT ?
    """, (limit * 2,))  # Get more results for merging

    semantic_results = {}
    for row in cursor.fetchall():
        semantic_results[row['chunk_id']] = {
            'chunk_id': row['chunk_id'],
            'post_id': row['post_id'],
            'content': row['content'],
            'semantic_score': 1 - row['distance']  # Convert distance to similarity
        }
```

**这段代码做了什么**：通过 VSS 扩展的 `vss_cosine_distance` 函数，计算查询向量与所有存储向量之间的余弦距离，按距离升序排列（越小越相似）。

**关键点**：
- `vss_cosine_distance`：返回的是**距离**（0 到 2 之间），不是相似度。距离越小 = 越相似
- `1 - row['distance']`：将距离转换为相似度分数（越大越好），方便后续和关键词搜索的分数融合
- `limit * 2`：多取一倍结果，因为后面要和关键词搜索结果合并再取 top-N。如果只取 limit 个，合并后可能选择范围太小
- 结果存入字典（以 `chunk_id` 为 key），方便后续按 ID 合并

#### 5.3 关键词搜索组件（FTS5 BM25）

```python
    # Keyword search using FTS5
    keyword_weight = 1 - semantic_weight
    keyword_results = {}

    try:
        cursor.execute("""
            SELECT
                c.chunk_id,
                c.post_id,
                c.content,
                bm25(chunks_fts) as score
            FROM chunks_fts f
            JOIN content_chunks c ON f.rowid = c.chunk_id
            WHERE chunks_fts MATCH ?
            ORDER BY score
            LIMIT ?
        """, (query_text, limit * 2))

        for row in cursor.fetchall():
            keyword_results[row['chunk_id']] = {
                'chunk_id': row['chunk_id'],
                'post_id': row['post_id'],
                'content': row['content'],
                'keyword_score': abs(row['score'])  # BM25 returns negative scores
            }
    except:
        # If FTS5 match fails, continue with semantic results only
        pass
```

**这段代码做了什么**：通过 FTS5 全文搜索引擎，使用 BM25 算法对查询进行关键词匹配和排序。

**关键点**：
- **BM25（Best Matching 25）**：一种经典的概率检索函数，综合考虑词频（TF）、文档长度等因素来评估相关性
- `bm25(chunks_fts)` 返回的是**负数**（SQLite FTS5 的设计），所以用 `abs()` 取绝对值转为正数
- `WHERE chunks_fts MATCH ?`：FTS5 的匹配语法，支持词组、布尔运算等
- **try-except 的设计意图**：用户输入的查询可能包含 FTS5 不支持的特殊字符，导致 MATCH 失败。此时优雅降级（graceful degradation），只用语义搜索结果

> **注意**：这种 bare `except` 会吞掉所有异常，生产环境中建议至少打个日志。

#### 5.4 分数融合与排序

```python
    # Combine results
    all_chunk_ids = set(semantic_results.keys()) | set(keyword_results.keys())
    combined_results = []

    for chunk_id in all_chunk_ids:
        sem_score = semantic_results.get(chunk_id, {}).get('semantic_score', 0)
        key_score = keyword_results.get(chunk_id, {}).get('keyword_score', 0)

        # Normalize keyword scores
        if keyword_results:
            max_keyword = max(r['keyword_score'] for r in keyword_results.values())
            if max_keyword > 0:
                key_score = key_score / max_keyword

        combined_score = (sem_score * semantic_weight) + (key_score * keyword_weight)

        # Get chunk content
        content = semantic_results.get(chunk_id, keyword_results.get(chunk_id, {})).get('content', '')
        post_id = semantic_results.get(chunk_id, keyword_results.get(chunk_id, {})).get('post_id', '')

        combined_results.append({
            'chunk_id': chunk_id,
            'post_id': post_id,
            'content': content,
            'score': combined_score
        })

    # Sort by combined score and limit
    combined_results.sort(key=lambda x: x['score'], reverse=True)
    return combined_results[:limit]
```

**这段代码做了什么**：将语义搜索和关键词搜索的结果合并，加权打分，排序后返回 top-N。

**直觉建立**：想象你在选最佳候选人，有两个面试官分别打分。面试官 A（语义搜索）打分范围是 0-1 分，面试官 B（关键词搜索）打分范围是 0-50 分。如果直接把两个分数加起来，面试官 B 的影响力会碾压面试官 A（50 >> 1）。所以必须先"归一化"——把两人的分数都映射到 0-1 范围。然后按 7:3 的权重加权合并：最终得分 = A 分数 × 0.7 + B 分数 × 0.3。归一化确保了两路搜索在"同一标尺"上比较，权重决定了谁的话语权更大。

**关键点**：
- **归一化（Normalization）**：关键词分数除以最大值，映射到 [0, 1] 范围。语义分数（1 - cosine_distance）本身就在 [0, 1] 范围内，所以不需要额外归一化
- **加权融合公式**：`combined_score = sem_score × 0.7 + key_score × 0.3`
- **集合合并**：`set(semantic_results.keys()) | set(keyword_results.keys())` 确保两路搜索的结果都被考虑。一个 chunk 可能只出现在语义结果中、或只出现在关键词结果中、或两者都有
- 如果某个 chunk 只出现在一路中，另一路的分数默认为 0

> **融合策略的选择**：这里用的是**加权线性融合（Weighted Linear Fusion）**，是最简单的方式。更高级的策略包括 Reciprocal Rank Fusion (RRF)、学习排序（Learning to Rank）等。

**适用场景**：
- `semantic_weight=0.7` 适合大多数自然语言问答场景
- 如果你的场景需要精确匹配特定术语（如法律条款、代码变量名），可以调高关键词权重
- 如果查询都是自然语言描述（如"怎么学 Python"），可以调高语义权重

---

### 6. LLM 集成：Ollama（6.4 节）

**定义**：Ollama 是一个本地运行 LLM 的工具，提供 HTTP API 接口。无需云服务，数据不出本机，零 API 费用。

#### 6.1 Ollama API 调用

```python
def call_ollama(prompt, model="llama3.1:8b", temperature=0.1):
    """Simple Ollama API call"""
    url = "http://localhost:11434/api/generate"

    payload = {
        "model": model,
        "prompt": prompt,
        "stream": False,
        "options": {
            "temperature": temperature,
            "top_p": 0.9
        }
    }

    try:
        response = requests.post(url, json=payload)
        response.raise_for_status()
        return response.json()['response']
    except Exception as e:
        print(f"Ollama error: {e}")
        return f"Error calling Ollama: {str(e)}"
```

**这段代码做了什么**：向本地 Ollama 服务发送 HTTP POST 请求，让 LLM 根据 prompt 生成回答。

**关键点**：
- **`temperature=0.1`**：非常低的温度参数。PDF 明确解释了原因——**确保一致性和事实准确性**（ensure consistent, factual responses based on the provided context）。在 RAG 场景中，我们不希望 LLM "创意发挥"，而是希望它老老实实地根据检索到的上下文回答
- **`top_p=0.9`**：核采样参数，控制生成多样性的另一个旋钮
- **`"stream": False`**：关闭流式输出，等完整结果一次性返回。简化了代码逻辑
- **模型选择 `llama3.1:8b`**：8B 参数量在质量和推理速度之间取得平衡。更大的模型（70B）质量更好但推理慢，更小的模型快但质量可能不够
- **`localhost:11434`**：Ollama 的默认端口

> **temperature 的直觉理解**：temperature=0 时模型总是选概率最高的下一个 token（最"确定"），temperature=1 时选择更随机（更"有创意"）。RAG 要的是准确，所以用接近 0 的值。

> **常见误用**：在 RAG 系统中把 temperature 设为 0.7-1.0（默认值），以为"让 LLM 更聪明"。实际上高温度会让 LLM 更倾向于"创意发挥"而非"忠实引用"，大幅增加幻觉风险。RAG 场景的本质是"基于证据回答"，不是"创意写作"。正确做法：RAG 用 0.0-0.2 的低温度；只有在需要 LLM 做改写、摘要等创造性任务时才适当提高温度。

#### 6.2 健康检查函数

```python
def test_ollama():
    """Test if Ollama is running"""
    try:
        response = requests.get("http://localhost:11434/api/tags")
        models = response.json()
        print("Available Ollama models:", [m['name'] for m in models.get('models', [])])
        return True
    except:
        print("Ollama not running. Start with: ollama serve")
        return False
```

**这段代码做了什么**：检查 Ollama 服务是否在运行，并列出可用模型。

**关键点**：
- `/api/tags` 端点返回所有已下载的模型列表
- 如果调用失败，打印启动指令 `ollama serve`，方便调试
- 这是一个很好的防御性编程实践——在执行主流程之前先验证依赖服务是否可用

---

### 7. RAG Pipeline：把一切串起来（6.5 节）

这是整个系统最核心的部分——将检索和生成编排成一个完整的问答管道。

#### 7.1 上下文格式化

```python
def format_context(chunks, conn):
    """Format retrieved chunks for the LLM"""
    cursor = conn.cursor()
    formatted_chunks = []

    for i, chunk in enumerate(chunks, 1):
        # Get post metadata
        cursor.execute("""
            SELECT title, subreddit, author, score
            FROM posts
            WHERE post_id = ?
        """, (chunk['post_id'],))

        post = cursor.fetchone()
        if post:
            formatted_chunks.append(f"""
SOURCE {i}:
From r/{post['subreddit']} by u/{post['author']} (Score: {post['score']})
Title: {post['title']}
Content:
{chunk['content']}
""")

    return "\n---\n".join(formatted_chunks)
```

**这段代码做了什么**：把检索到的 chunk 列表格式化为 LLM 能理解的结构化上下文字符串，包含来源元数据。

**关键点**：
- **为什么要附带元数据？** PDF 明确说："Including metadata helps the LLM understand the source and credibility of information, leading to better answers."（包含元数据帮助 LLM 理解信息来源和可信度，产出更好的答案）
- `SOURCE 1`, `SOURCE 2`... 编号方便 LLM 在回答中引用来源
- 子版块（subreddit）、作者（author）、评分（score）给 LLM 提供了判断信息质量的依据
- `---` 分隔符清晰地划分不同来源

#### 7.2 问答管道：4 步流程

```python
def answer_question(conn, question, num_chunks=5):
    """Complete RAG pipeline to answer a question"""
    print(f"\nQuestion: {question}")

    # Step 1: Retrieve relevant chunks
    start_time = time.time()
    chunks = hybrid_search(conn, question, limit=num_chunks)
    retrieval_time = (time.time() - start_time) * 1000

    if not chunks:
        return "No relevant information found in the database."

    print(f"Retrieved {len(chunks)} chunks in {retrieval_time:.1f}ms")
```

**Step 1 — 检索**：调用混合搜索获取最相关的 5 个 chunk，并计时。

```python
    # Step 2: Format context
    context = format_context(chunks, conn)
```

**Step 2 — 格式化上下文**：将 chunk 列表转为带元数据的结构化文本。

```python
    # Step 3: Create prompt
    prompt = f"""You are a helpful assistant that answers questions based on Reddit content.
Use ONLY the information provided below to answer the question.
If the information is insufficient, say so clearly.

QUESTION: {question}

RETRIEVED INFORMATION:
{context}

ANSWER:"""
```

**Step 3 — 构建 Prompt**：将问题和上下文组装成完整的 prompt。

```python
    # Step 4: Generate answer
    start_time = time.time()
    answer = call_ollama(prompt)
    generation_time = (time.time() - start_time) * 1000

    print(f"Generated answer in {generation_time:.1f}ms")
    print(f"Total time: {retrieval_time + generation_time:.1f}ms")
    return answer
```

**Step 4 — 生成回答**：调用 Ollama 生成最终答案，并输出性能指标。

**关键点**：

1. **Prompt 设计的三条原则**：
   - **角色设定**："You are a helpful assistant that answers questions based on Reddit content." 明确 LLM 的身份和任务
   - **约束条件**："Use ONLY the information provided below to answer the question." 这是防幻觉的关键——强制 LLM 只用检索到的内容，不要用自己的"知识"
   - **兜底机制**："If the information is insufficient, say so clearly." 当检索结果不足以回答时，让 LLM 诚实地说"不知道"，而不是编造

2. **性能特征**：PDF 指出——检索通常很快（毫秒级），生成是瓶颈（几百毫秒到秒级）。这个性能指标可以帮你判断系统瓶颈在哪，决定优化方向

3. **`num_chunks=5`**：默认检索 5 个 chunk。太少可能遗漏关键信息，太多会浪费上下文窗口且可能引入噪音

> **常见误用**：在 Prompt 中省略"Use ONLY the information provided"这类约束，或者用模糊的措辞如"参考以下内容回答"。缺少强约束时，LLM 会自由混合检索内容和自身"记忆"来回答，你完全无法分辨哪些是基于真实数据的、哪些是 LLM 编造的。更糟糕的是，编造的内容往往写得很自信，让用户误以为是事实。正确做法：Prompt 中必须明确约束"只用提供的内容"，并且加上"如果信息不足请说明"的兜底机制。

---

### 8. 演示与测试（6.6 节）

#### 8.1 示例数据

```python
def load_sample_data(conn):
    """Load some sample Reddit data for testing"""
    sample_posts = [
        {
            'post_id': 'post001',
            'title': 'ELI5: What is machine learning?',
            'content': """Machine learning is like teaching a computer to recognize patterns by showing it many examples.
Instead of programming exact rules, you feed it data and it learns the patterns itself. For example,
to teach it to recognize cats, you show it thousands of cat pictures until it learns what makes a cat a cat.
It's used in spam filters, recommendation systems, and voice assistants.""",
            'subreddit': 'explainlikeimfive',
            'author': 'curious_user',
            'score': 245
        },
        {
            'post_id': 'post002',
            'title': 'Best Python libraries for beginners?',
            'content': """I recommend starting with: NumPy for numerical computing, Pandas for data manipulation,
Matplotlib for basic plotting, Requests for HTTP requests, and Flask for simple web apps. These libraries
cover most beginner needs and have excellent documentation. Don't try to learn them all at once -
start with one based on your project needs.""",
            'subreddit': 'learnpython',
            'author': 'python_mentor',
            'score': 189
        },
        {
            'post_id': 'post003',
            'title': 'How do vector databases work?',
            'content': """Vector databases store data as high-dimensional vectors (lists of numbers) that represent
the semantic meaning of the data. When you search, your query is converted to a vector and the database
finds the most similar vectors using distance metrics like cosine similarity. This enables semantic search
where you find content by meaning rather than exact keyword matches. Popular vector databases include
Pinecone, Weaviate, and pgvector for PostgreSQL.""",
            'subreddit': 'programming',
            'author': 'db_expert',
            'score': 156
        }
    ]

    print("Loading sample data...")
    for post in sample_posts:
        store_post_with_chunks(conn, post)
    print(f"Loaded {len(sample_posts)} sample posts")
```

**关键点**：三条样例数据刻意选了不同领域（机器学习、Python 库、向量数据库），用来演示系统处理多领域问题的能力。

#### 8.2 主演示函数

```python
def main():
    """Main demonstration of the RAG system"""
    print("=== Minimal RAG System with SQLite VSS and Ollama ===\n")

    # Step 1: Setup database
    print("1. Setting up database...")
    conn = setup_database()

    # Step 2: Check Ollama
    print("\n2. Checking Ollama...")
    if not test_ollama():
        print("Please start Ollama first: ollama serve")
        print("Then pull a model: ollama pull llama3.1:8b")
        return

    # Step 3: Load sample data
    print("\n3. Loading sample data...")
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM posts")
    if cursor.fetchone()[0] == 0:
        load_sample_data(conn)
    else:
        print("Data already loaded")
```

**关键点**：
- 按步骤引导式执行：建库 → 检查 Ollama → 加载数据
- 加载数据前先检查是否已有数据（`SELECT COUNT(*)`），避免重复灌入
- 如果 Ollama 没启动，给出明确的启动指令（`ollama serve` + `ollama pull llama3.1:8b`）

#### 8.3 交互式问答

```python
    # Step 4: Demo questions
    print("\n4. Demonstrating RAG system...")
    demo_questions = [
        "What is machine learning and how does it work?",
        "What Python libraries should a beginner learn?",
        "How do vector databases enable semantic search?"
    ]

    for question in demo_questions[:1]:  # Just show one example
        answer = answer_question(conn, question)
        print(f"\nAnswer: {answer}\n")
        print("=" * 50)

    # Step 5: Interactive mode
    print("\n5. Interactive Q&A (type 'quit' to exit)")
    print("-" * 50)

    while True:
        question = input("\nYour question: ").strip()
        if question.lower() in ['quit', 'exit', 'q']:
            break
        if not question:
            continue
        answer = answer_question(conn, question)
        print(f"\nAnswer: {answer}")

    conn.close()
    print("\nGoodbye!")
```

**关键点**：
- 先用一个预设问题做 demo，让用户看到系统能正常工作
- 然后进入交互模式，用户可以自由提问
- 支持 `quit` / `exit` / `q` 退出

#### 8.4 快速测试工具

```python
def quick_start():
    """Quick start for testing individual components"""
    conn = setup_database()

    # Quick test of search
    print("Testing hybrid search for 'machine learning'...")
    results = hybrid_search(conn, "machine learning", limit=3)
    for i, result in enumerate(results, 1):
        print(f"\n{i}. Score: {result['score']:.3f}")
        print(f"   Content: {result['content'][:150]}...")

    conn.close()

if __name__ == "__main__":
    # Run main demo
    main()
    # Or run quick test
    # quick_start()
```

**关键点**：`quick_start()` 可以单独测试搜索组件，不需要启动 Ollama。这在调试 Embedding 和搜索逻辑时非常有用。

---

### 9. 扩展方向汇总

PDF 最后系统梳理了五大扩展方向，这些是把"教学 demo"变成"生产系统"的路线图：

#### 9.1 Reddit 数据完善（Missing Reddit Data Features）

| 扩展项 | 说明 |
|--------|------|
| **Comment threads（评论线程）** | 最有价值的洞见往往在评论区，需要通过 PRAW 获取评论树，处理父子关系，并决定如何分块线程对话同时保留上下文 |
| **Post metadata（帖子元数据）** | 时间戳、奖项、转发信息、编辑历史等元数据可以提升检索相关性和回答质量 |
| **Media content（媒体内容）** | 帖子常包含图片、视频、外链，当前纯文本系统无法处理 |
| **Live data ingestion（实时数据灌入）** | 系统用的是静态样例数据，真实场景需要 API 认证和速率限制 |

#### 9.2 检索增强（Retrieval Enhancements）

| 扩展项 | 说明 |
|--------|------|
| **Semantic chunking（语义分块）** | 根据句子嵌入和段落边界智能切分，让相关内容保持在同一块中，取代简单的按词数切分 |
| **Query expansion（查询扩展）** | 用 LLM 生成查询的多种替代表述，然后分别搜索，提升复杂问题的召回率 |
| **Cross-encoder reranking（交叉编码器重排序）** | 初始检索后，用 cross-encoder 模型对结果进行精排，显著提升精确度 |
| **Metadata filtering（元数据过滤）** | 搜索前按子版块、日期范围、作者 karma、帖子评分等条件预过滤 |

#### 9.3 性能优化（Performance Optimizations）

| 扩展项 | 说明 |
|--------|------|
| **Embedding cache（嵌入缓存）** | 用文本哈希→嵌入向量的映射避免重复计算 |
| **Connection pooling（连接池）** | 单连接是并发瓶颈，连接池支持并行操作 |
| **Batch processing（批处理）** | 逐条处理效率低，批量嵌入和批量写入可大幅提升吞吐量 |
| **Asynchronous pipelines（异步管道）** | 串行的"检索→生成"流程可以对多查询并行化 |

#### 9.4 生产化考量（Production Considerations）

| 扩展项 | 说明 |
|--------|------|
| **Incremental updates（增量更新）** | 当前系统无法在不重新处理全部数据的情况下添加新内容 |
| **Version management（版本管理）** | Embedding 模型升级时，需要策略来迁移旧向量或维护多版本 |
| **Monitoring and analytics（监控与分析）** | 生产系统需要查询日志、性能指标、用户反馈闭环来发现检索失败并持续改进 |
| **Scale considerations（规模考虑）** | SQLite + 本地 Ollama 适合 demo，大规模场景建议 PostgreSQL + pgvector 做数据库、API-based LLM 支持并发用户 |

#### 9.5 高级 RAG 模式（Advanced RAG Patterns）

| 扩展项 | 说明 |
|--------|------|
| **Multi-hop reasoning（多跳推理）** | 复杂问题需要多次迭代检索，LLM 可以请求额外搜索 |
| **Source diversity（来源多样性）** | 确保检索结果来自不同帖子/作者，避免回声室效应 |
| **Confidence scoring（置信度评分）** | 让 LLM 根据检索内容的质量和相关性标注答案的置信度 |
| **Fact verification（事实核查）** | 增加验证步骤，将生成答案中的声明与源材料逐一核对 |

---

### 10. 总结：RAG 系统的核心价值

PDF 在结论部分总结了 RAG 系统实现的四大能力：

1. **理解查询的含义**（Understand the meaning behind queries），而不仅仅是关键词
2. **从大规模文档集合中检索最相关的信息**（Retrieve the most relevant information from large document collections）
3. **生成准确的、有上下文的答案**，答案锚定在真实数据上（Generate accurate, contextual answers grounded in real data）
4. **避免幻觉**，通过将回答约束在检索到的信息范围内（Avoid hallucinations by constraining responses to retrieved information）

混合搜索让系统兼得语义理解和精确匹配两方面的优势。SQLite + Ollama 的本地部署提供了隐私保护和成本效益，同时保持了不错的性能。

这个基础可以通过更高级的分块策略、更好的排序算法、查询扩展技术和多步推理链来扩展——所有这些都建立在本章演示的核心 RAG 模式之上。

---

## 重点标记

1. **RAG 的核心目标**：解决 LLM 知识冻结问题，让它能基于私有/最新数据回答问题，同时防止幻觉
2. **五组件架构**：Vector DB Layer / Embedding Engine / Hybrid Search / LLM Integration / RAG Pipeline Orchestrator，缺一不可
3. **双索引策略**：VSS（向量相似性搜索）+ FTS5（BM25 全文搜索），两者互补实现混合搜索
4. **Embedding 模型一致性**：存储和查询必须使用同一个 Embedding 模型（all-MiniLM-L6-v2，384 维），否则向量空间不匹配
5. **分块参数**：chunk_size=200, overlap=50 是兼顾上下文完整性和检索精度的经验值
6. **混合搜索权重**：semantic_weight=0.7 意味着 70% 语义 + 30% 关键词，可根据场景调整
7. **temperature=0.1**：RAG 场景用低温度确保事实准确性，不要让 LLM "创意发挥"
8. **Prompt 防幻觉三板斧**：角色设定 + "Use ONLY the information provided" + 不确定时诚实说不知道
9. **性能瓶颈分布**：检索毫秒级，生成秒级——优化重点在 LLM 推理速度
10. **从 demo 到生产的关键差距**：增量更新、Embedding 版本管理、监控分析、规模化（SQLite→PostgreSQL，本地 Ollama→API LLM）

---

## 自测：你真的理解了吗？

> 以下问题考的是理解，不是记忆。试着先思考再看答案。

**Q1**：你的 RAG 系统用 `all-MiniLM-L6-v2`（384 维）生成嵌入并存储了 10 万条数据。现在你想换成效果更好的 `all-mpnet-base-v2`（768 维）模型。你能只把新查询用新模型生成嵌入，而存储的旧数据保持不变吗？为什么？

**Q2**：用户反馈 RAG 系统回答"Python 中的 GIL 是什么"时，返回的上下文全是关于"Python 蟒蛇"和"吉尔伯特（Gilbert）"的内容，完全不相关。但数据库里确实存有关于 Python GIL 的技术帖子。你会先排查哪一路搜索（语义搜索还是关键词搜索）的问题？可能的原因是什么？

**Q3**：你的团队在争论混合搜索的权重配置。场景是一个法律法规问答系统，用户经常搜索具体的法条编号（如"第 42 条"、"民法典 1032 条"）。同事 A 建议 `semantic_weight=0.9`，同事 B 建议 `semantic_weight=0.3`。你支持谁？请说明理由。

**Q4**：你的 RAG 系统检索到了 5 个 chunk，其中 3 个来自同一个帖子的相邻分块（chunk_index 为 2、3、4），另外 2 个来自不同帖子。这种情况下，你觉得 `num_chunks=5` 的设置合理吗？这暴露了当前系统的什么局限？你会怎么改进？

**Q5**：一个同事提议把 Ollama 的 temperature 从 0.1 提高到 0.8，理由是"回答太生硬了，用户体验不好"。从 RAG 系统防幻觉的角度，你怎么看这个提议？有没有两全其美的方案？
