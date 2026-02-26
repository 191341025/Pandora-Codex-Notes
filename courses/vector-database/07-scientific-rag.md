# 模块七：科学文献 RAG（PostgreSQL + pgvector）

> 对应 PDF: 07_Scientific_RAG_with_PostgreSQL_pgvector.pdf，第 1-3 页

---

## 概念地图

- **核心概念** (必须内化): 双层 Schema 设计（papers + paper_sections）、多级语义搜索（摘要级 + 章节级）、科学 RAG 的五大特殊挑战
- **实操要点** (动手时需要): pgvector HNSW 索引配置、章节检测 + 二次分块策略、科学 Prompt 工程四指令
- **背景知识** (扩展理解): ArXiv API 集成与速率限制、本地 LLM 隐私推理、从教学原型到生产系统的七大扩展方向

---

## 概念讲解

### 1. 科学文献 RAG 的特殊挑战

**定义**：科学文献 RAG（Retrieval-Augmented Generation）是专门针对学术论文设计的"检索增强生成"系统。和通用 RAG 不同，它要解决学术论文带来的一系列独有难题。

**核心思想**：通用 RAG 处理的是"普通文本"——博客、文档、对话等。但学术论文不是普通文本，它有自己的结构、术语、引用网络和质量层级。如果不针对性处理这些特点，RAG 的检索质量会大打折扣。

**直觉建立**：想象你是一个图书馆员，面对的不是普通小说，而是堆满了医学论文的房间。普通小说你翻几页就知道讲什么，但医学论文里满是专业术语（"myocardial infarction"其实就是心梗），有些关键结论藏在公式和数据表里，而且一篇论文引用了另外 30 篇——不读那 30 篇就看不懂这一篇。更麻烦的是，Nature 上发表的和某人博客上写的"医学发现"可信度天差地别。这就是科学 RAG 面对的世界：不是"找到包含关键词的文档"那么简单，而是要理解术语、结构、引用网络和证据质量，才能给出可靠答案。

**为什么重要/有效**：ArXiv 每月发表超过 15,000 篇论文，涵盖物理、数学、计算机科学等领域。传统关键词搜索根本抓不住科学话语的语义丰富性——同一个概念在不同领域、不同研究社区中可能有完全不同的表述方式。

**五大核心挑战**：

| 挑战 | 说明 | 举例 |
|------|------|------|
| **Technical Terminology（技术术语）** | 论文使用精确的、领域特定的语言，需要超越简单关键词的语义理解 | "attention mechanism" 和 "self-attention" 在不同论文中的表述差异 |
| **Structured Content（结构化内容）** | 论文遵循固定格式（摘要、方法、结果、结论），这些结构可以用来指导检索策略 | 只想看"实验结果"而不是整篇论文 |
| **Citation Networks（引用网络）** | 论文存在于引用关系网中，引用关系提供额外上下文 | A 引用了 B，B 的内容可能是理解 A 的前提 |
| **Mathematical Notation（数学符号）** | 公式和方程承载了纯文本系统无法捕捉的含义 | 损失函数的数学定义 vs 文字描述 |
| **Evidence Quality（证据质量）** | 不是所有来源都等价——同行评审、发表平台、被引次数都很重要 | Nature 上的论文 vs 未经审稿的 preprint |

**适用场景**：
- 学术研究人员需要从海量文献中发现相关工作
- 需要跨论文综合回答复杂科学问题
- 需要精确检索某篇论文的特定章节（方法论、实验结果）
- 需要基于真实研究证据生成回答，并附带引用

> **常见误用**：直接把通用 RAG 方案（如 Ch.6 的 SQLite RAG）套用到科学文献上，不做任何领域适配。这样做会导致：(1) 检索质量差——通用分块策略丢失论文的章节结构信息；(2) 回答缺乏学术严谨性——没有引用源的 Prompt 设计会让 LLM 编造不存在的论文；(3) 证据质量无法区分——把未审稿 preprint 和顶刊论文同等对待。科学 RAG 必须针对论文结构、术语、引用和证据质量做专门设计。

---

### 2. 系统目标与能力设计

**定义**：在动手写代码之前，先明确这个科学 RAG 系统到底要能做什么。

**五大核心能力**：

1. **Semantic Discovery（语义发现）**：根据概念相似性而非关键词匹配来发现论文
2. **Cross-Paper Synthesis（跨论文综合）**：回答需要多篇论文信息才能回答的问题
3. **Contextual Understanding（上下文理解）**：检索与查询相关的特定章节（方法论、结果等）
4. **Evidence-Based Responses（循证回答）**：生成基于实际研究的答案，带有规范引用
5. **Technical Depth（技术深度）**：处理需要领域专业知识的复杂科学查询

**实现这些目标的技术手段**：
- 捕获科学语义的向量嵌入（Vector Embeddings）
- 论文的分层索引（Hierarchical Indexing）——摘要级别 + 章节级别
- PostgreSQL + pgvector 提供可扩展的向量操作
- 与 ArXiv 集成实现实时论文访问
- 本地 LLM 推理保护隐私和控制权

---

### 3. 系统架构总览（六层架构）

**定义**：整个系统由六个互相连接的组件组成，形成一个从数据获取到用户交互的完整流水线。

**核心思想**：关注点分离（Separation of Concerns）。每一层独立负责一个环节，可以单独优化和测试。

**直觉建立**：把这个六层架构想象成一条寿司传送带餐厅的运作流程。第一层（数据摄入）是后厨从鱼市场进货、清洗、切片——对应从 ArXiv 获取论文和提取文本。第二层（向量处理）是寿司师傅把食材变成标准化的寿司——对应把文本变成 384 维向量。第三层（存储）是冷藏柜，分类存放半成品——对应 PostgreSQL 存储元数据和向量。第四层（检索）是传送带系统，客人说"我想吃三文鱼"，传送带把最相关的盘子送过来——对应语义搜索。第五层（生成）是服务员根据客人的口味偏好，把几盘寿司组合成推荐套餐——对应 LLM 综合上下文生成回答。第六层（界面）是点餐台和菜单——对应用户交互。关键在于：每一层可以独立升级（换更好的鱼市场、换更快的传送带），不影响其他层。但需要注意，真实系统中各层之间的依赖关系比传送带更紧密，比如向量维度的变化会同时影响处理层和存储层。

**架构详解**：

```
┌────────────────────────────────────────────────────────────────┐
│  1. Data Ingestion Layer（数据摄入层）                          │
│     ArXiv API → PDF Processing → Section Detection             │
├────────────────────────────────────────────────────────────────┤
│  2. Vector Processing Pipeline（向量处理流水线）                │
│     SentenceTransformers → 分层嵌入 → 384维向量                │
├────────────────────────────────────────────────────────────────┤
│  3. Storage Layer（存储层）: PostgreSQL + pgvector              │
│     Papers 表 + Sections 表 + HNSW 索引                        │
├────────────────────────────────────────────────────────────────┤
│  4. Retrieval System（检索系统）                                │
│     Multi-Level Search → Similarity Threshold → Context Assembly│
├────────────────────────────────────────────────────────────────┤
│  5. Generation Layer（生成层）: Ollama                          │
│     Local LLM + Scientific Prompts + Citation Handling          │
├────────────────────────────────────────────────────────────────┤
│  6. User Interface（用户界面）                                  │
│     Query Processing + Interactive Search + Performance Monitor │
└────────────────────────────────────────────────────────────────┘
```

**各层职责**：

| 层 | 核心职责 | 关键技术选择 |
|----|----------|-------------|
| Data Ingestion | 从 ArXiv 获取论文元数据和 PDF，提取结构化文本，识别章节 | ArXiv API + PyMuPDF |
| Vector Processing | 将文本转为密集向量，为摘要和章节分别生成嵌入 | SentenceTransformers, all-MiniLM-L6-v2, 384维 |
| Storage | 存储元数据、嵌入向量，提供高性能近似最近邻搜索 | PostgreSQL + pgvector + HNSW 索引 |
| Retrieval | 同时查询摘要和章节，按相关度过滤，组装上下文 | 多级搜索 + 阈值过滤 |
| Generation | 本地隐私推理，学术风格 Prompt，维护论文引用 | Ollama + llama3.1:8b |
| User Interface | 自然语言查询，不同搜索模式，性能监控 | 交互式命令行界面 |

---

### 4. 数据库基础：pgvector 设计

**定义**：使用 PostgreSQL + pgvector 扩展作为向量存储层，同时提供关系数据完整性和向量相似性搜索。

**核心思想**：与纯向量数据库不同，这种方案两全其美——既有关系型数据库的 ACID 事务保障，又有原生向量操作能力。向量操作直接在数据库内执行，不需要把数据传到应用层。

**依赖安装**：

```bash
pip install psycopg2-binary arxiv sentence-transformers requests PyMuPDF
```

#### 4.1 数据库配置与初始化

```python
def setup_database():
    """Setup PostgreSQL with pgvector for scientific papers"""
    DB_CONFIG = {
        'host': 'localhost',
        'port': '5432',
        'database': 'scientific_rag',
        'user': 'postgres',
        'password': 'password'
    }
    conn = psycopg2.connect(**DB_CONFIG)
    cursor = conn.cursor()

    # 启用 pgvector 扩展
    cursor.execute("CREATE EXTENSION IF NOT EXISTS vector;")
```

> **要点**：`CREATE EXTENSION IF NOT EXISTS vector` 这一行给 PostgreSQL 加上了 `vector` 数据类型，让向量操作可以在数据库内原生执行，这对性能至关重要。

#### 4.2 Schema 设计——反映论文的层次结构

**论文表（papers）**——存储论文级别的元数据和摘要嵌入：

```python
cursor.execute("""
    CREATE TABLE IF NOT EXISTS papers (
        paper_id TEXT PRIMARY KEY,
        title TEXT NOT NULL,
        abstract TEXT,
        authors TEXT[],           -- PostgreSQL 数组类型，免去单独建表
        published_date DATE,
        pdf_url TEXT,
        abstract_embedding vector(384)  -- 摘要级嵌入
    );
""")
```

**设计决策**：
- `authors TEXT[]`：使用 PostgreSQL 原生数组类型存储作者列表，保持数据结构性又不需要额外的关联表
- `abstract_embedding vector(384)`：摘要嵌入支持论文级别的相似性搜索

**章节表（paper_sections）**——存储分块内容和章节嵌入：

```python
cursor.execute("""
    CREATE TABLE IF NOT EXISTS paper_sections (
        section_id SERIAL PRIMARY KEY,
        paper_id TEXT REFERENCES papers(paper_id),  -- 外键关联
        section_number INTEGER,
        content TEXT NOT NULL,
        section_embedding vector(384)  -- 章节级嵌入
    );
""")
```

**直觉建立**：想象一个图书馆的索引系统。传统做法是每本书一张索引卡（书名、作者、摘要）——你能找到"哪本书可能相关"，但翻开 300 页的书后还得自己找。现在这个系统做了两级索引：第一级是书的索引卡（papers 表，存摘要嵌入），帮你快速锁定相关的书；第二级是每本书内部的章节索引（paper_sections 表，存章节嵌入），帮你直接翻到最相关的那一页。查询时，系统同时搜索两级索引——如果你问"transformer 的注意力机制怎么训练"，摘要级搜索告诉你"论文 X 整体上讨论了这个话题"，章节级搜索则精确定位到"论文 X 第 3 节的训练方法详述"。两级协作，既有宏观视野又有微观精度。

**为什么需要两级存储**：科学论文内容多样——对结果感兴趣的人可能不关心方法论。章节级索引允许精确检索特定部分。这是科学 RAG 与普通 RAG 的关键区别之一。

#### 4.3 HNSW 高性能向量索引

```python
# 摘要嵌入索引
cursor.execute("""
    CREATE INDEX IF NOT EXISTS idx_abstract_embeddings
    ON papers USING hnsw (abstract_embedding vector_cosine_ops);
""")

# 章节嵌入索引
cursor.execute("""
    CREATE INDEX IF NOT EXISTS idx_section_embeddings
    ON paper_sections USING hnsw (section_embedding vector_cosine_ops);
""")
```

**关键参数解释**：
- `USING hnsw`：使用 HNSW（Hierarchical Navigable Small World）索引算法，构建多层图结构，即使面对百万级向量也能实现对数级搜索复杂度
- `vector_cosine_ops`：指定使用余弦相似度（Cosine Similarity），适合归一化后的嵌入向量

> **对比注意**：Ch.5 中也使用了 pgvector 的 HNSW 索引。这里的索引策略完全一致，但应用场景不同——Ch.5 是单级搜索，Ch.7 是分层搜索（摘要 + 章节两个索引）。

---

### 5. 嵌入生成策略（Embedding Generation）

**定义**：嵌入是人类语言和数学相似性之间的桥梁。嵌入模型和策略的选择直接影响检索质量。

**核心思想**：用单例模式加载模型，避免重复加载带来的性能损耗。

```python
# 全局嵌入模型
embedding_model = None

def get_embedding_model():
    """Get or initialize the embedding model"""
    global embedding_model
    if embedding_model is None:
        embedding_model = SentenceTransformer('all-MiniLM-L6-v2')
        print(f"Loaded embedding model (dimension: {embedding_model.get_sentence_embedding_dimension()})")
    return embedding_model

def generate_embedding(text):
    """Generate embedding for text"""
    model = get_embedding_model()
    return model.encode(text, convert_to_numpy=True)
```

**模型选择要点**：
- **all-MiniLM-L6-v2**：专门在科学文本对上训练过，适合学术内容
- **384 维输出**：在语义丰富度和计算效率之间取得平衡
- **Singleton 模式**：`get_embedding_model()` 确保模型只加载一次，后续调用直接复用

---

### 6. ArXiv 集成与 PDF 处理

**定义**：科学 RAG 需要获取论文全文内容，不仅仅是摘要。这部分处理论文发现和内容提取。

#### 6.1 论文发现——ArXiv API

```python
@dataclass
class Paper:
    paper_id: str
    title: str
    abstract: str
    authors: List[str]
    published_date: str
    pdf_url: str

def fetch_arxiv_papers(query, max_results=5):
    """Fetch papers from ArXiv"""
    client = arxiv.Client()
    search = arxiv.Search(
        query=query,
        max_results=max_results,
        sort_by=arxiv.SortCriterion.SubmittedDate
    )

    papers = []
    for result in client.results(search):
        paper = Paper(
            paper_id=result.entry_id.split('/')[-1],
            title=result.title,
            abstract=result.summary,
            authors=[author.name for author in result.authors],
            published_date=result.published.date(),
            pdf_url=result.pdf_url
        )
        papers.append(paper)
        time.sleep(0.5)  # Rate limiting，尊重 ArXiv 服务器

    return papers
```

**设计细节**：
- **dataclass** 提供类型安全和干净的数据结构
- ArXiv 查询支持复杂布尔逻辑和字段搜索：`cat:cs.LG` 用于机器学习论文，`abs:"machine learning"` 用于摘要搜索
- **rate limiting**（`time.sleep(0.5)`）：尊重 ArXiv 服务器，防止被封禁

#### 6.2 智能 PDF 文本提取

PDF 处理是出了名的难，特别是科学论文——复杂排版、公式、图表。这里的方案聚焦于鲁棒的文本提取 + 章节检测：

```python
def download_pdf_text(pdf_url):
    """Download and extract text from PDF using PyMuPDF"""
    try:
        import fitz  # PyMuPDF

        # 下载 PDF
        headers = {
            'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36'
        }
        response = requests.get(pdf_url, headers=headers, timeout=30)
        if response.status_code != 200:
            return []

        # 从字节流打开 PDF
        doc = fitz.open(stream=response.content, filetype="pdf")

        sections = []
        current_section = ""

        for page_num in range(len(doc)):
            page = doc[page_num]
            text = page.get_text()

            lines = text.split('\n')
            # 常见学术论文章节标题
            section_headers = [
                'introduction', 'abstract', 'methodology', 'method',
                'results', 'discussion', 'conclusion', 'references',
                'related work', 'experiments', 'background', 'approach',
                'evaluation', 'analysis'
            ]

            for line in lines:
                line_lower = line.lower().strip()
                is_header = False
                for header in section_headers:
                    if (line_lower == header or
                        (len(line.split()) <= 3 and header in line_lower) or
                        line_lower.startswith(f"{header}.")):
                        is_header = True
                        break

                if is_header and current_section.strip():
                    sections.append(current_section.strip())
                    current_section = line + "\n"
                else:
                    current_section += line + "\n"

        if current_section.strip():
            sections.append(current_section.strip())

        # 如果没检测到章节，按页面分割（fallback 策略）
        if len(sections) <= 1:
            sections = []
            for page_num in range(min(10, len(doc))):
                page = doc[page_num]
                text = page.get_text()
                if text.strip():
                    sections.append(text.strip())

        doc.close()

        # 过滤掉太短的段落（可能是页眉/页脚）
        filtered_sections = [s for s in sections if len(s) > 100]
        return filtered_sections[:20]  # 限制最多 20 个章节

    except ImportError:
        print("PyMuPDF not installed. Install with: pip install PyMuPDF")
        return []
    except Exception as e:
        print(f"Error extracting PDF text: {e}")
        return []
```

**章节检测逻辑的三个匹配规则**：
1. 行内容完全等于标题名（如单独一行 "Introduction"）
2. 行很短（<=3 词）且包含标题关键词（如 "1. Introduction"）
3. 行以标题关键词开头加点号（如 "Introduction."）

**容错设计**：
- **User-Agent header**：防止 CDN 拦截自动化请求
- **timeout=30**：防止慢连接导致挂起
- **fallback 到按页分割**：格式再差的 PDF 也能提取出内容
- **100 字符最小长度**：过滤页眉、页脚、页码等噪音
- **最多 20 个章节**：防止超长论文导致内存问题

---

### 7. 科学论文分块策略（Advanced Text Chunking）

**定义**：分块策略对检索质量有决定性影响。块太小会丢失上下文，块太大会稀释相关性。

**核心思想**：500 字符的块大致对应一个段落，在科学文本中段落边界通常和概念边界对齐。加上重叠（overlap）确保边界处的信息不会丢失。

```python
def simple_chunk_text(text, chunk_size=500, overlap=50):
    """Simple text chunking by character count"""
    chunks = []
    words = text.split()

    current_chunk = []
    current_length = 0

    for word in words:
        word_length = len(word) + 1  # +1 for space
        if current_length + word_length > chunk_size and current_chunk:
            # 保存当前块
            chunks.append(' '.join(current_chunk))
            # 带 overlap 开始新块
            if overlap > 0:
                overlap_words = current_chunk[-overlap//10:]  # 粗略 overlap
                current_chunk = overlap_words
                current_length = sum(len(w) + 1 for w in overlap_words)
            else:
                current_chunk = []
                current_length = 0

        current_chunk.append(word)
        current_length += word_length

    if current_chunk:
        chunks.append(' '.join(current_chunk))

    return chunks
```

**与普通文本分块的区别**：

| 维度 | 普通文本分块 | 科学论文分块 |
|------|-------------|-------------|
| 分块依据 | 固定字符数或按段落 | 先按章节检测分割，再在章节内按字符分块 |
| 上下文 | 分块即可 | 需要保留章节归属信息（来自哪个 section） |
| 边界处理 | 简单 overlap | 需要 overlap，因为公式推导等常跨段落 |
| 大小建议 | 通常 1000+ 字符 | 500 字符左右（科学段落通常较短且信息密集） |
| 预处理 | 去除多余空白 | 需要过滤掉页眉、页脚、参考文献列表等噪音 |

> **注意**：这里的分块是"二次分块"——先用章节检测把论文切成大块（section），再在每个 section 内用 `simple_chunk_text` 切成 500 字符的小块。两层切割保证了语义连贯性。

> **常见误用**：用过大的块（如 2000+ 字符）处理科学论文，认为"块越大上下文越丰富"。实际上科学文本信息密度极高，过大的块会导致嵌入向量被无关内容"稀释"——一个块里同时包含方法论和结论，向量就无法精确表示任何一个主题，检索时两种查询都匹配不好。500 字符左右是科学文本的甜点区间。

---

### 8. 存储流水线与批量处理

**定义**：存储流水线协调 PDF 处理、分块、嵌入生成和数据库插入的全过程。

#### 8.1 单篇论文存储

```python
def store_paper_with_embeddings(db_config, paper, sections):
    """Store paper and sections with embeddings"""
    conn = psycopg2.connect(**db_config)
    cursor = conn.cursor()

    try:
        # 生成摘要嵌入
        abstract_embedding = generate_embedding(paper.abstract)

        # 存储论文（幂等操作）
        cursor.execute("""
            INSERT INTO papers (paper_id, title, abstract, authors,
                              published_date, pdf_url, abstract_embedding)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (paper_id) DO UPDATE SET
                title = EXCLUDED.title,
                abstract = EXCLUDED.abstract,
                abstract_embedding = EXCLUDED.abstract_embedding
        """, (paper.paper_id, paper.title, paper.abstract,
              paper.authors, paper.published_date, paper.pdf_url,
              abstract_embedding.tolist()))

        # 存储章节及其嵌入
        for i, section_text in enumerate(sections):
            if section_text.strip():
                chunks = simple_chunk_text(section_text)
                for j, chunk in enumerate(chunks):
                    section_embedding = generate_embedding(chunk)
                    cursor.execute("""
                        INSERT INTO paper_sections
                            (paper_id, section_number, content, section_embedding)
                        VALUES (%s, %s, %s, %s)
                    """, (paper.paper_id, i * 100 + j, chunk,
                          section_embedding.tolist()))

        conn.commit()
    except Exception as e:
        conn.rollback()
        raise
    finally:
        cursor.close()
        conn.close()
```

**关键设计决策**：
- **`ON CONFLICT ... DO UPDATE`**：幂等操作，重新处理同一篇论文时更新而非重复插入。恢复失败或刷新论文时保持数据一致性
- **章节编号 `i * 100 + j`**：同时保留了章节顺序（i）和章节内分块顺序（j），需要时可以还原文档原始结构
- **事务处理**：要么整篇论文全部存入，要么一条都不存（原子性）

#### 8.2 批量处理流水线

```python
def process_and_store_papers(db_config, papers):
    """Process and store multiple papers"""
    for paper in papers:
        sections = download_pdf_text(paper.pdf_url)
        if sections:
            store_paper_with_embeddings(db_config, paper, sections)
        else:
            print(f" Skipping - no content extracted")
        time.sleep(1)  # Rate limiting
```

> **生产提示**：顺序处理防止大 PDF 导致内存问题，也尊重了速率限制。在生产环境中，可以通过合理的资源管理实现并行化。

---

### 9. 多级语义搜索（Multi-Level Semantic Search）

**定义**：科学查询需要不同粒度的搜索。有时需要论文级别的概览（摘要），有时需要具体的技术细节（章节）。

**核心思想**：分层检索——同时搜索摘要和章节，然后合并排序。

**直觉建立**：想象你在一个大型购物中心找一双特定的运动鞋。第一步（摘要搜索）相当于看每家店的招牌和橱窗——"这家是运动品牌店，可能有"；第二步（章节搜索）相当于走进店里，在货架上逐排查找——"第三排第二层就是那双鞋"。如果只做第一步，你知道去哪家店但不知道鞋在哪；如果只做第二步，你可能在不相关的店里浪费大量时间。系统的巧妙之处在于同时做两步搜索，然后把所有结果按相关度统一排序——就像同时有一个看招牌的人和一个逛货架的人，最后汇总他们的发现，把最靠谱的结果排在最前面。这里"撒大网"用低阈值 0.6 检索，相当于"宁可多走几家店也别漏掉"，最后再筛选才收紧标准。

#### 9.1 摘要级搜索

```python
def search_papers_by_abstract(db_config, query, limit=5, threshold=0.7):
    """Search papers by abstract similarity"""
    conn = psycopg2.connect(**db_config)
    cursor = conn.cursor()

    query_embedding = generate_embedding(query)

    cursor.execute("""
        SELECT
            paper_id, title, abstract, authors,
            1 - (abstract_embedding <=> %s::vector) as similarity
        FROM papers
        WHERE abstract_embedding IS NOT NULL
        AND 1 - (abstract_embedding <=> %s::vector) > %s
        ORDER BY similarity DESC
        LIMIT %s
    """, (query_embedding.tolist(), query_embedding.tolist(), threshold, limit))

    results = cursor.fetchall()
    cursor.close()
    conn.close()

    return [{'paper_id': row[0], 'title': row[1], 'abstract': row[2],
             'authors': row[3], 'similarity': row[4]} for row in results]
```

**SQL 关键点**：
- `<=>`：pgvector 的余弦距离操作符（Cosine Distance）
- `1 - (... <=>)`：将距离转为相似度——1.0 表示完全相同，0.0 表示正交
- `threshold` 参数：过滤掉边缘相关的结果，提高精确度

**适用场景**：适合发现相关论文、了解研究版图

#### 9.2 章节级搜索

```python
def search_paper_sections(db_config, query, limit=10, threshold=0.7):
    """Search within paper sections"""
    conn = psycopg2.connect(**db_config)
    cursor = conn.cursor()

    query_embedding = generate_embedding(query)

    cursor.execute("""
        SELECT
            ps.paper_id, p.title, ps.section_number,
            ps.content,
            1 - (ps.section_embedding <=> %s::vector) as similarity
        FROM paper_sections ps
        JOIN papers p ON ps.paper_id = p.paper_id
        WHERE ps.section_embedding IS NOT NULL
        AND 1 - (ps.section_embedding <=> %s::vector) > %s
        ORDER BY similarity DESC
        LIMIT %s
    """, (query_embedding.tolist(), query_embedding.tolist(), threshold, limit))

    results = cursor.fetchall()
    cursor.close()
    conn.close()

    return [{'paper_id': row[0], 'title': row[1], 'section_number': row[2],
             'content': row[3], 'similarity': row[4]} for row in results]
```

**与摘要搜索的区别**：
- JOIN papers 表获取论文标题，让用户知道这段内容来自哪篇论文
- 章节搜索通常返回更多结果（章节数远多于论文数）
- 适合找具体的技术细节、方法论描述或实验结果

---

### 10. RAG Pipeline 深度解析

**定义**：RAG Pipeline 是检索与生成的交汇点。这是决定回答质量的最关键组件，需要精心编排搜索、上下文组装和 Prompt 工程。

#### 10.1 本地 LLM 集成（Ollama）

```python
def call_ollama(prompt, model="llama3.1:8b"):
    """Call Ollama for text generation"""
    url = "http://localhost:11434/api/generate"
    payload = {
        "model": model,
        "prompt": prompt,
        "stream": False,
        "options": {
            "temperature": 0.1,   # 低温度 → 一致、事实性的回答
            "top_p": 0.9
        }
    }
    try:
        response = requests.post(url, json=payload, timeout=60)
        response.raise_for_status()
        return response.json()['response']
    except Exception as e:
        return f"Error calling Ollama: {str(e)}"
```

**参数选择理由**：
- **temperature=0.1**：极低温度确保基于检索上下文生成一致、事实性的回答，不做"创意发挥"
- **timeout=60**：为复杂科学解释的长篇生成留足时间
- **stream=False**：简化实现，但生产环境中可启用流式输出提升体验

**为什么用本地 LLM**：科学研究可能涉及敏感内容或未发表的数据，本地推理保护隐私。

#### 10.2 健康检查与模型发现

```python
def test_ollama():
    """Test if Ollama is available"""
    try:
        response = requests.get("http://localhost:11434/api/tags", timeout=5)
        models = response.json()
        print("Available Ollama models:",
              [m['name'] for m in models.get('models', [])])
        return True
    except:
        print("Ollama not running. Start with: ollama serve")
        return False
```

> 这个工具函数帮助部署和调试。它显示可用模型，帮助用户选择适合其领域的模型（比如专门的科学模型）。

#### 10.3 智能上下文检索

```python
def retrieve_relevant_contexts(db_config, question, max_contexts=5):
    """Retrieve relevant contexts for RAG"""
    # 同时搜索摘要和章节
    abstract_results = search_papers_by_abstract(
        db_config, question, limit=3, threshold=0.6)
    section_results = search_paper_sections(
        db_config, question, limit=max_contexts, threshold=0.6)

    contexts = []

    # 添加摘要上下文
    for result in abstract_results:
        contexts.append({
            'type': 'abstract',
            'paper_id': result['paper_id'],
            'title': result['title'],
            'content': result['abstract'],
            'similarity': result['similarity']
        })

    # 添加章节上下文
    for result in section_results:
        contexts.append({
            'type': 'section',
            'paper_id': result['paper_id'],
            'title': result['title'],
            'content': result['content'],
            'similarity': result['similarity']
        })

    # 按相似度排序并限制数量
    contexts.sort(key=lambda x: x['similarity'], reverse=True)
    return contexts[:max_contexts]
```

**双搜索策略的设计智慧**：
- 同时获取宏观上下文（摘要）和微观细节（章节）
- 检索时用**更低的阈值 0.6**（而展示时可能用 0.7），相当于"撒网更广"，让 LLM 自己判断哪些真正相关
- 按相似度排序后截断——最相关的上下文排在最前面，LLM 通常对 prompt 前部的内容给予更多权重

#### 10.4 科学 Prompt 工程

```python
def format_scientific_rag_prompt(question, contexts):
    """Format scientific contexts for RAG"""
    if not contexts:
        return f"Question: {question}\n\nI don't have relevant scientific literature to answer this question."

    context_parts = []
    for i, ctx in enumerate(contexts, 1):
        source_type = "Abstract" if ctx['type'] == 'abstract' else "Section"
        context_parts.append(
            f"Source {i} ({source_type} from '{ctx['title']}', "
            f"similarity: {ctx['similarity']:.3f}):\n"
            f"{ctx['content'][:500]}..."
        )
    context_section = "\n\n".join(context_parts)

    prompt = f"""You are a scientific assistant helping researchers understand academic literature.
Answer the question based ONLY on the provided research paper excerpts.
Cite the specific papers when referencing information.

Question: {question}

Relevant research findings:
{context_section}

Answer based on the scientific literature above:"""

    return prompt
```

**Prompt 设计的四个关键指令**：
1. **角色设定**（"scientific assistant"）：设定合适的语气和专业水平
2. **仅基于提供的上下文**（"based ONLY on"）：防止幻觉
3. **要求引用源**（"Cite the specific papers"）：维护学术严谨性
4. **附带相似度分数**：帮助用户评估相关性

**上下文截断**（500 字符/条）：在信息密度和 token 限制之间取得平衡。包含论文标题和类型帮助 LLM 理解来源的可信度和相关性。

> **常见误用**：在科学 RAG 的 Prompt 中不加"based ONLY on"约束，或者不要求 LLM 引用来源。这会导致 LLM "创意发挥"——编造不存在的论文标题、虚构实验数据、或者把自己的训练知识当作检索结果呈现。在学术场景中，这种幻觉比不回答更危险，因为用户可能会把虚构的引用写进自己的论文里。

#### 10.5 完整 RAG 执行流水线

```python
def scientific_rag_answer(db_config, question, max_contexts=5):
    """Generate scientific answer using RAG"""
    print(f"\nScientific Question: {question}")

    # Step 1: 检索相关上下文
    start_time = time.time()
    contexts = retrieve_relevant_contexts(db_config, question, max_contexts)
    retrieval_time = (time.time() - start_time) * 1000

    if not contexts:
        return "I couldn't find relevant scientific literature to answer this question."

    # 展示检索到的上下文
    for i, ctx in enumerate(contexts, 1):
        print(f"  {i}. {ctx['title'][:50]}... (similarity: {ctx['similarity']:.3f})")

    # Step 2: 格式化 Prompt
    prompt = format_scientific_rag_prompt(question, contexts)

    # Step 3: 生成回答
    start_time = time.time()
    answer = call_ollama(prompt)
    generation_time = (time.time() - start_time) * 1000

    print(f"Total time: {retrieval_time + generation_time:.1f}ms")

    return {
        'answer': answer,
        'contexts': contexts,
        'stats': {
            'retrieval_time_ms': retrieval_time,
            'generation_time_ms': generation_time,
            'num_contexts': len(contexts)
        }
    }
```

**性能基准**：
| 环节 | 典型耗时 | 说明 |
|------|----------|------|
| 检索（Retrieval） | 50-200ms | 有 HNSW 索引时 |
| 生成（Generation） | 500-5000ms | 取决于回答长度 |
| 总响应时间 | <5s | 科学查询可接受的范围 |

**返回上下文和统计信息的好处**：
- 用户可以验证回答的来源
- 方便系统性能优化
- 调试检索问题
- 通过透明性建立用户信任

---

### 11. 演示与交互界面

#### 11.1 主演示流程

```python
def main():
    """Main demonstration of scientific RAG system"""
    # 1. 初始化数据库
    db_config = setup_database()

    # 2. 检查是否已有论文（智能避免重复处理）
    conn = psycopg2.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM papers")
    paper_count = cursor.fetchone()[0]
    cursor.close()
    conn.close()

    if paper_count == 0:
        # 3. 从 ArXiv 获取论文
        papers = fetch_arxiv_papers(
            'cat:cs.LG AND (abs:"machine learning" OR abs:"neural network")',
            max_results=3)
        if papers:
            process_and_store_papers(db_config, papers)
    else:
        print(f"Found {paper_count} existing papers in database")

    # 4. 演示语义搜索
    search_queries = ["neural network architecture",
                      "machine learning optimization",
                      "deep learning applications"]

    for query in search_queries[:1]:  # 展示一个示例
        results = search_papers_by_abstract(db_config, query, limit=3)
        # ... 展示结果
        section_results = search_paper_sections(db_config, query, limit=3)
        # ... 展示结果

    # 5. 演示 RAG（如果 Ollama 可用）
    if test_ollama():
        questions = [
            "What are the main challenges in machine learning?",
            "How do neural networks learn from data?",
            "What are current trends in AI research?"
        ]
        for question in questions[:1]:
            result = scientific_rag_answer(db_config, question)
            # ... 展示结果
```

**演示问题的设计意图**：
- "Challenges" 类问题：需要跨论文综合信息
- "How" 类问题：需要技术解释
- "Trends" 类问题：需要分析近期论文

#### 11.2 交互式搜索界面

```python
def interactive_search():
    """Interactive scientific search interface"""
    db_config = setup_database()
    print("Commands: 'abstract [query]', 'section [query]', 'ask [question]', 'quit'\n")

    while True:
        command = input("\n> ").strip()
        if command.lower() == 'quit':
            break

        parts = command.split(' ', 1)
        cmd, query = parts[0].lower(), parts[1]

        if cmd == 'abstract':
            results = search_papers_by_abstract(db_config, query, limit=3)
            # ... 展示结果

        elif cmd == 'section':
            results = search_paper_sections(db_config, query, limit=5)
            # ... 展示结果

        elif cmd == 'ask':
            if test_ollama():
                result = scientific_rag_answer(db_config, query)
                # ... 展示结果
```

**三种搜索模式**：
| 命令 | 用途 | 返回内容 |
|------|------|----------|
| `abstract [query]` | 发现相关论文 | 论文标题、相似度、摘要片段 |
| `section [query]` | 查找具体细节 | 章节内容、来源论文 |
| `ask [question]` | RAG 问答 | LLM 生成的带引用回答 |

**入口点支持两种模式**：

```python
if __name__ == "__main__":
    import sys
    if len(sys.argv) > 1 and sys.argv[1] == "interactive":
        interactive_search()
    else:
        main()
```

---

### 12. Next Steps：系统扩展方向

PDF 原文给出了一份详尽的扩展路线图，涵盖七个方向。这些不是"可选的锦上添花"，而是从教学原型到生产系统的必经之路：

#### 12.1 Enhanced PDF Processing（增强 PDF 处理）

| 方向 | 说明 |
|------|------|
| Formula Extraction（公式提取） | 当前系统纯文本提取会丢失数学公式。集成 LaTeX OCR 或 MathML 提取可以捕获这些关键信息 |
| Figure & Table Analysis（图表分析） | 研究成果常通过可视化传达。多模态模型可以描述图表并从表格中提取结构化数据 |
| Citation Parsing（引用解析） | 提取和跟踪引用关系，实现跨连接研究的多跳推理 |
| Structured Extraction（结构化提取） | 用训练过的 ML 模型取代简单的章节检测，更准确地识别方法论、结果、贡献 |

#### 12.2 Advanced Retrieval Techniques（高级检索技术）

| 方向 | 说明 |
|------|------|
| Hybrid Search | 向量相似度 + 关键词匹配 + 引用网络 + 元数据过滤 |
| Query Decomposition（查询分解） | 复杂科学问题拆解为多个子问题，分别检索再合并 |
| Iterative Refinement（迭代精化） | 让 LLM 基于初始结果请求额外搜索，实现更深入的主题探索 |
| Concept Linking（概念链接） | 构建科学概念知识图谱，用于查询扩展 |

#### 12.3 Improved RAG Pipeline（改进 RAG 流水线）

| 方向 | 说明 |
|------|------|
| Reranking（重排序） | 初始检索后用专门训练的 cross-encoder 模型重排序 |
| Answer Verification（答案验证） | 对比源文档验证生成回答中的声明 |
| Multi-paper Synthesis（多论文综合） | 处理来自多篇论文的信息组合、矛盾和共识 |
| Confidence Scoring（置信度评分） | 基于支持来源的数量和质量标注置信度 |

#### 12.4 Domain Specialization（领域特化）

- Field-specific Models：针对特定领域（物理、生物、CS）微调的嵌入模型
- Terminology Handling：领域特定词典和缩写展开
- Equation Understanding：符号数学处理
- Experimental Data：数值结果、统计测试、实验参数的提取与分析

#### 12.5 Scale and Performance（规模与性能）

- Distributed Processing：并行化 PDF 处理和嵌入生成
- Incremental Updates：流式摄入新论文，无需重新处理整个语料库
- Caching Strategies：缓存嵌入、搜索结果和生成的回答
- Index Optimization：根据语料库大小和查询模式调优 HNSW 参数（M, ef_construction）

#### 12.6 User Experience Enhancements（用户体验增强）

- Source Highlighting：精确展示回答中每个声明对应的源文档位置
- Interactive Refinement：用户反馈改进后续搜索
- Export Capabilities：生成格式化引用、参考文献条目、文献综述草稿
- Collaborative Features：团队间共享搜索、标注和论文集

#### 12.7 Quality Assurance（质量保障）

- Evaluation Metrics：标准 IR 指标 + RAG 特有指标（答案准确性、引用正确性）
- Test Datasets：领域特定的问答对 + ground truth
- Human-in-the-loop：专家反馈接口
- Bias Detection：监控论文选择和回答生成中的偏见

#### 12.8 Integration Capabilities（集成能力）

- API Endpoints：提供 REST/GraphQL API，与现有研究工具和工作流集成
- Reference Managers：与 Zotero、Mendeley、EndNote 等参考文献管理器对接
- Writing Assistants：与 LaTeX 编辑器和文字处理器集成，辅助论文写作
- Continuous Monitoring：设置保存的查询条件，当匹配的新论文发表时自动发送告警通知

---

### 13. 与 Ch.6 SQLite RAG 的对比

**核心区别**：Ch.6 是入门级 RAG 原型，Ch.7 是面向科学场景的专业级系统。

| 维度 | Ch.6 SQLite RAG | Ch.7 Scientific RAG (pgvector) |
|------|-----------------|-------------------------------|
| **数据库** | SQLite + sqlite-vss 扩展 | PostgreSQL + pgvector 扩展 |
| **ACID 事务** | SQLite 事务（单用户） | PostgreSQL 完整 ACID（多用户并发） |
| **索引类型** | 基础向量索引 | HNSW 高性能近似最近邻索引 |
| **检索层级** | 单级（文本块） | 双级（摘要级 + 章节级） |
| **数据源** | 通用文本 | ArXiv 科学论文（PDF 处理） |
| **分块策略** | 通用文本分块 | 章节检测 + 二次分块 |
| **Prompt 设计** | 通用问答 | 科学助手 + 强制引用源 |
| **扩展性** | 适合原型/小规模（万级文档） | 生产级（百万级文档，原文明确说 ~1M） |
| **部署复杂度** | 极低（SQLite 无需安装） | 中等（需 PostgreSQL 服务） |
| **适用场景** | 个人知识库、小团队内部搜索 | 科研团队、学术机构、大规模文献管理 |
| **LLM** | Ollama（本地） | Ollama（本地），同样注重隐私 |

**选择建议**：
- 如果只是做个原型验证 RAG 概念，或者数据量在万级以下 → Ch.6 的 SQLite 方案足够
- 如果要处理科学论文、需要多用户并发、数据量可能到百万级 → Ch.7 的 PostgreSQL + pgvector 方案

---

### 14. 系统结论与关键收获

PDF 最后的 Conclusion 强调了几个核心要点：

**系统能力总结**：
1. 通过学术文本上训练的语义嵌入**理解科学概念**
2. 用摘要和章节的分层索引**导航文档结构**
3. 从多篇论文检索并生成连贯回答实现**跨源综合**
4. 通过来源归因和置信度评分**维护学术严谨**
5. 使用 PostgreSQL + pgvector 实现**高效扩展**——可支撑约 100 万文档

**PostgreSQL + pgvector 相比专用向量数据库的优势**：
- **事务完整性**：论文和嵌入的更新保持一致
- **关系查询**：可以在向量搜索旁边做复杂的元数据过滤
- **运维简单**：单一数据库降低部署复杂度
- **经验证的规模**：合理索引下可处理百万级论文

**架构理念**：关注点分离使每个组件可以独立优化和测试。模块化设计确保系统能在不改变架构的情况下采用未来的技术改进（更好的嵌入模型、检索算法、语言模型）。

---

## 重点标记

1. **科学 RAG 五大挑战**：技术术语、结构化内容、引用网络、数学符号、证据质量——这些决定了不能直接用通用 RAG
2. **双层 Schema 设计**：papers 表（摘要嵌入）+ paper_sections 表（章节嵌入），反映论文的层次结构
3. **HNSW + vector_cosine_ops**：pgvector 的高性能近似最近邻索引，百万级向量也能对数级搜索
4. **章节检测 + 二次分块**：先按学术标题切章节，再在章节内 500 字符分块，保证语义连贯
5. **ON CONFLICT DO UPDATE**：幂等存储操作，重复处理不会产生重复数据
6. **双搜索合并策略**：摘要搜索（宏观）+ 章节搜索（微观），检索用低阈值 0.6 撒大网，展示用高阈值筛选
7. **低温度 0.1 生成**：科学问答场景下确保事实性，不做创意发挥
8. **Prompt 四指令**：角色设定、仅用上下文、强制引用、附带相似度
9. **性能预期**：检索 50-200ms，生成 500-5000ms，总响应 <5s
10. **规模上限**：PostgreSQL + pgvector 方案可支撑约 100 万文档
11. **七大扩展方向**：PDF 处理增强、高级检索、RAG 改进、领域特化、规模与性能、用户体验、质量保障——从教学原型到生产系统的路线图
12. **与 Ch.6 的核心区别**：SQLite 适合原型/小规模，PostgreSQL + pgvector 适合生产级/百万级科学文献管理

---

## 自测：你真的理解了吗？

> 以下问题考的是理解，不是记忆。试着先思考再看答案。

**Q1**：你正在构建一个科学 RAG 系统，用户搜索"attention mechanism optimization"时，系统只从摘要级搜索返回了结果，章节级搜索一无所获。可能的原因是什么？你会怎么排查和解决？

**Q2**：一位同事建议把 papers 和 paper_sections 合并成一张表，每行存储一个章节并在该行中重复存储论文的元数据（标题、作者等），理由是"减少 JOIN 操作能提升查询速度"。你认为这个方案是否合理？分析利弊。

**Q3**：系统当前使用固定的相似度阈值 0.6 进行检索。在以下两种场景中，你会如何调整这个阈值？(a) 论文库只有 50 篇高度相关的论文；(b) 论文库有 100 万篇覆盖多个领域的论文。为什么？

**Q4**：假设你需要把这个科学 RAG 从教学原型升级为可供 50 人研究团队使用的生产系统，你认为六层架构中哪一层最先需要改造？改造的优先级如何排序？说说你的理由。

**Q5**：当前系统对每个上下文片段截断到 500 字符后传入 Prompt。如果一篇论文的关键方法论描述恰好在 500 字符处被截断（公式被切成两半），这会导致什么问题？你会如何改进分块或截断策略来避免这种情况？
