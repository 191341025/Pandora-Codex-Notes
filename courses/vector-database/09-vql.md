# 模块九：VQL 向量查询语言（草案规范）

> 对应 PDF: 09_Vector_Query_Language_VQL_Draft_Spec.pdf，第 1-3 页
> 这是 O'Reilly Early Release 的第 9 章草案，尚未正式出版

## 概念地图

- **核心概念** (必须内化): VQL 统一抽象层的设计哲学、VECTOR_OPERATION 语法扩展模式、双曲空间操作集（Poincare 投影/指数映射/对数映射）
- **实操要点** (动手时需要): 四种搜索操作的语法与适用场景（相似性/混合/范围/批量）、索引管理（HNSW vs IVFFlat 选型）、链式变换的构建与验证
- **背景知识** (扩展理解): 线性代数操作（PCA/SVD/特征分解的关系）、流形学习三件套（ISOMAP/UMAP/t-SNE）、类型系统与错误处理机制

---

## 概念讲解

### 1. VQL 设计动机（Motivation）

**定义**：VQL（Vector Query Language）是一门受 SQL 启发的查询语言，专为向量数据库设计。它把 SQL 的经典语法和向量操作融合在一起，让开发者能用熟悉的方式操作向量数据。

**核心思想**：一句话概括——**把 SQL 对关系数据库做的事情，搬到向量数据库上来做**。SQL 为关系数据库提供了统一的抽象和跨厂商的访问层；VQL 要为向量数据库提供同样的东西。

**直觉建立**：想象你在一个欧洲城市旅行，每个国家的插座形状不同——英国三孔方插、德国双孔圆插、意大利三孔直线插。你带了一堆设备，每到一个国家就要买一个新转接头。VQL 就像一个**万能转接头**：你的设备（应用程序）只需要一种插头形状（VQL 语法），转接头内部负责适配各国插座（Pinecone、Milvus、Weaviate 等不同厂商的 API）。没有 VQL 时，你的应用代码里充满了 `if vendor == 'pinecone': ...` 这样的条件分支；有了 VQL，你只写一套查询语句，底层的厂商差异被抽象层吞掉了。不过要注意，VQL 目前还是草案，这个"万能转接头"还没正式生产——它代表的是行业的方向，而非今天就能用的工具。

**为什么重要**：截至 2025 年，向量数据库生态有一个突出痛点——**没有统一的查询语言**。每个厂商都有自己的 API 和语法，应用程序被紧紧绑定在特定的实现上，数据库的内部隐喻渗透到了应用架构中。VQL 试图解决这个问题。

**三类目标用户**：

| 用户群体 | 痛点 | VQL 如何帮助 |
|---------|------|-------------|
| **应用开发者（Application Developers）** | 各家向量数据库 API 不统一，应用与实现紧耦合，缺少关系数据库那样的抽象层 | 提供统一的抽象和跨厂商访问层，让开发者面向领域对象编程，而非底层数据结构 |
| **数据库运维（DevOps / SREs / 数据工程师）** | 管理多家厂商的向量数据库时，需要为每家定制 DDL/DML 脚本，即便有 AI 辅助也需要人工检查（因为这些操作有高破坏性） | 统一的 DDL/DML 语法，一套脚本适配多厂商 |
| **AI/ML 研究者** | 在 GenAI、信号处理、拓扑数据分析、向量嵌入等领域，研究者不得不写大量程序来操纵数据 | VQL 充当"宏语言"，用来组合变换、跑实验算法——这些操作直接在数据库中执行。把写程序变成写 VQL 脚本，更容易读、调试和维护，缩短实验周期 |

**适用场景**：
- 需要跨多个向量数据库厂商的统一访问层
- AI/ML 实验中需要快速组合和测试向量变换
- DevOps 团队管理多套向量数据库基础设施

---

### 2. 核心数据模型（Core Data Model）

**定义**：VQL 定义了五个核心概念，构成其数据模型的基础。

| 概念 | 类比 | 说明 |
|------|------|------|
| **Collection** | 数据库 schema | 顶层容器，用来组织向量数据 |
| **Table** | 数据库表 | 结构化的向量集合，附带元数据 |
| **Vector** | 数据列 | 固定维度的数值数组 |
| **Embedding** | 特殊的列 | 特殊类型的向量，代表某个被编码的实体 |
| **Metadata** | 普通列 | 以原生列（native columns）形式存储的附加结构化信息 |

**核心思想**：VQL 复用了关系数据库的层级概念（schema → table → column），但把核心数据类型换成了向量。这样 SQL 老手可以无缝迁移认知模型。

**为什么重要**：
- **Collection.Table** 的二级命名空间让数据组织方式与关系数据库一致
- Metadata 作为"原生列"而非 JSON blob，意味着可以做高效过滤
- Vector 和 Embedding 做了区分——不是所有向量都是嵌入，但所有嵌入都是向量

---

### 3. 基本语法结构（Basic Syntax Structure）

**定义**：VQL 的查询语句沿用了 SQL 的骨架，但插入了一个全新的子句 `VECTOR_OPERATION`。

```sql
SELECT fields
FROM collection_name.table_name
VECTOR_OPERATION operation_parameters
WHERE metadata_filters
ORDER BY distance_metric
LIMIT top_k;
```

**核心思想**：保持 SQL 的 `SELECT / FROM / WHERE / ORDER BY / LIMIT` 不变，新增 `VECTOR_OPERATION` 子句来承载向量特有操作。

**各子句的职责**：

| 子句 | 来源 | 在 VQL 中的角色 |
|------|------|----------------|
| `SELECT` | SQL | 选择返回的字段（含向量字段和元数据字段） |
| `FROM` | SQL | 指定 Collection.Table |
| `VECTOR_OPERATION` | **VQL 新增** | 指定向量操作（相似性搜索、混合搜索、范围搜索等） |
| `WHERE` | SQL | 元数据过滤条件 |
| `ORDER BY` | SQL | 按距离度量排序 |
| `LIMIT` / `TOP K` | SQL | 限制返回结果数量 |

> **注意**：VQL 同时使用 `LIMIT` 和 `TOP K` 两种写法，在不同操作类型中混用。

---

### 4. 向量操作——相似性搜索（Similarity Search）

**定义**：最基础的向量操作——给定一个查询向量，在表中找到最相似的 K 个向量。

#### 4.1 基本相似性搜索

```sql
-- 基本相似性搜索
SELECT *
FROM ecommerce.product_vectors
SIMILARITY SEARCH [1.2, 0.8, -0.2, 0.5]
USING METRIC cosine
TOP K 10;
```

**解释**：在 `ecommerce` 集合的 `product_vectors` 表中，用余弦相似度（cosine）找出与查询向量 `[1.2, 0.8, -0.2, 0.5]` 最相似的 10 条记录。

**关键元素**：
- `SIMILARITY SEARCH [向量]`：指定查询向量
- `USING METRIC cosine`：指定距离度量（cosine / euclidean 等）
- `TOP K 10`：返回最相似的 10 条

#### 4.2 带元数据过滤的相似性搜索

```sql
-- 带元数据过滤
SELECT product_name, category, price
FROM ecommerce.product_vectors
SIMILARITY SEARCH [1.2, 0.8, -0.2, 0.5]
USING METRIC euclidean
WHERE category = 'electronics' AND price < 1000
TOP K 5
THRESHOLD 0.8;
```

**解释**：在相似性搜索的基础上增加了两个过滤维度：
1. **元数据过滤**（`WHERE`）：只看电子产品且价格低于 1000
2. **相似度阈值**（`THRESHOLD 0.8`）：只返回相似度高于 0.8 的结果

**为什么重要**：纯粹的向量搜索往往不够用。实际业务中几乎总是需要结合结构化条件来过滤。`THRESHOLD` 是一个质量门控——确保返回的结果在语义上确实相关，而不仅仅是"K 个中相对最近的"。

**适用场景**：
- 电商推荐中限定品类和价格区间
- 文档搜索中限定时间范围和作者
- 任何需要"语义相似 + 业务约束"的场景

---

### 5. 混合搜索（Hybrid Search）

**定义**：同时结合向量相似性搜索和传统文本搜索（关键词搜索），通过权重分配来融合两种搜索的结果。

```sql
-- 向量相似性 + 文本搜索
HYBRID SEARCH (
    VECTOR [1.2, 0.8, -0.2] WEIGHT 0.7,
    TEXT "machine learning" WEIGHT 0.3
)
FROM research.paper_vectors
WHERE publication_year > 2020;
```

**核心思想**：向量搜索擅长捕捉语义相似性，文本搜索擅长精确匹配关键词——两者结合才是最实用的检索方式。

**关键元素**：
- `VECTOR [...] WEIGHT 0.7`：向量相似性搜索占 70% 权重
- `TEXT "..." WEIGHT 0.3`：文本匹配占 30% 权重
- 权重之和通常为 1.0

**为什么重要**：纯向量搜索可能会错过精确的关键词匹配；纯关键词搜索又无法理解语义。混合搜索是 RAG 系统中公认的最佳实践。

**适用场景**：
- RAG 系统中的文档检索（结合语义理解和关键词精确匹配）
- 学术论文搜索（语义相似 + 特定术语精确匹配）
- 电商搜索（"蓝色无线耳机"既要语义理解也要关键词匹配）

---

### 6. 范围搜索（Range Search）

**定义**：不限制返回数量（不是 Top K），而是返回距离在某个阈值以内的**全部**向量。

```sql
-- 在阈值范围内找所有向量
SELECT *
FROM user_data.behavior_vectors
RANGE SEARCH [user_vector]
USING METRIC cosine
THRESHOLD 0.5;
```

**核心思想**：Top K 搜索是"给我最相似的 K 个"，范围搜索是"给我所有足够相似的"——结果数量不固定。

**与 Top K 搜索的对比**：

| 维度 | Top K 搜索 | 范围搜索 |
|------|-----------|---------|
| 返回数量 | 固定 K 个 | 不固定，取决于阈值 |
| 保证 | 一定返回 K 个（即使不太相似） | 可能返回 0 个（没有满足阈值的） |
| 适用场景 | 推荐系统（总得推荐点什么） | 异常检测、相似用户发现（只要真正相似的） |

**适用场景**：
- 发现所有行为模式相似的用户
- 找出某个商品的所有近似品
- 质量控制中的相似缺陷检测

---

### 7. 批量操作（Batch Operations）

**定义**：一次提交多个查询向量，为每个查询向量分别执行相似性搜索。

```sql
-- 批量相似性搜索
BATCH SIMILARITY SEARCH (
    SELECT query_vectors FROM user_queries.batch_requests
)
FROM ecommerce.product_vectors
TOP K 5;
```

**核心思想**：从 `user_queries.batch_requests` 中取出一批查询向量，对每个向量在 `product_vectors` 中找 Top 5 相似结果。一次请求处理多个查询，减少网络往返开销。

**为什么重要**：在生产环境中，逐个执行相似性搜索效率太低。批量操作可以利用 GPU 并行计算和批处理优化来大幅提升吞吐量。

**适用场景**：
- 批量推荐生成（一次为多个用户生成推荐）
- 离线数据处理管道
- 大规模向量比对任务

---

### 8. 线性变换——矩阵操作（Linear Transformations: Matrix Operations）

**定义**：VQL 支持定义和应用矩阵变换，对向量进行旋转、缩放、投影等线性操作。

#### 8.1 定义变换矩阵

```sql
-- 定义旋转矩阵
CREATE TRANSFORM rotation_matrix AS MATRIX [
    [cos(theta), -sin(theta)],
    [sin(theta), cos(theta)]
];
```

**核心思想**：用 `CREATE TRANSFORM` 创建一个具名的变换矩阵，后续可以反复引用。这与 SQL 中的视图（VIEW）概念类似——把复杂的计算封装成一个名称。

#### 8.2 应用变换

```sql
-- 应用变换
SELECT TRANSFORM(vector_data, rotation_matrix) AS rotated_vector
FROM machine_learning.feature_vectors;
```

**解释**：对表中每个向量应用 `rotation_matrix`，输出旋转后的向量。

#### 8.3 链式变换

```sql
-- 链式变换（依次应用多个变换）
SELECT TRANSFORM_CHAIN(
    vector_data,
    [scaling_matrix, rotation_matrix, projection_matrix]
) AS transformed_vector
FROM machine_learning.feature_vectors;
```

**核心思想**：`TRANSFORM_CHAIN` 按数组顺序依次应用变换——先缩放、再旋转、最后投影。矩阵变换的顺序很重要，A * B != B * A。

> **注意**：链式变换的执行顺序是从左到右（scaling → rotation → projection）。在线性代数中，这等价于矩阵乘法 `projection * rotation * scaling * vector`（从右向左读），这是因为矩阵乘法的结合律。

**适用场景**：
- 特征工程中的向量变换
- 数据预处理管道
- 实验不同的变换组合对搜索质量的影响

> **常见误用**：忽视链式变换的顺序。`TRANSFORM_CHAIN(data, [A, B])` 和 `TRANSFORM_CHAIN(data, [B, A])` 结果通常不同，因为矩阵乘法不满足交换律。典型错误是"先投影再旋转"和"先旋转再投影"搞反，导致降维后的数据分布畸变，搜索质量莫名下降却很难定位原因。调试建议：每一步变换后都做 `VALIDATE TRANSFORM` 检查矩阵性质。

---

### 9. 内建线性变换（Built-in Linear Transformations）

VQL 内建了几个常用的降维和标准化变换，不需要手动定义矩阵。

#### 9.1 PCA（主成分分析）

```sql
SELECT PCA(vector_data, dimensions=50) AS reduced_vector
FROM analytics.high_dimensional_data;
```

**定义**：将高维向量降维到 50 维，保留方差最大的方向。PCA 是最经典的线性降维方法。

**直觉建立**：想象你站在一座山顶俯瞰一片城市。城市是三维的（有高楼、有平地），但你从正上方往下看，得到的是一个二维投影——一张地图。PCA 做的就是找到**最佳俯瞰角度**：让投影后的地图尽可能多地保留城市的结构差异。如果所有高楼都沿着东西方向排列，PCA 会优先保留东西方向（方差最大的方向），因为这个方向上的信息最丰富。保留的前 N 个方向就是"前 N 个主成分"，它们依次捕获数据中从多到少的变异信息。关键边界：PCA 只能找到线性方向（直的投影轴），如果数据的结构是弯曲的（比如一条螺旋），PCA 会表现得很糟——这时候需要流形学习方法（ISOMAP/UMAP/t-SNE）。

**适用场景**：高维嵌入的降维（如 1536 维 → 50 维），减少存储和计算开销，同时保留主要信息。

#### 9.2 随机投影（Random Projection）

```sql
SELECT RANDOM_PROJECTION(vector_data, output_dim=100, seed=42) AS projected
FROM analytics.sparse_vectors;
```

**定义**：利用随机矩阵将向量投影到低维空间。理论基础是 Johnson-Lindenstrauss 引理——随机投影近似保持向量间的距离。

**与 PCA 的对比**：

| 维度 | PCA | 随机投影 |
|------|-----|---------|
| 计算成本 | 高（需要计算协方差矩阵） | 低（只需随机矩阵乘法） |
| 质量 | 最优（保留最大方差） | 近似（保持距离关系） |
| 适用场景 | 数据量不太大时的最优降维 | 超大规模数据的快速降维 |
| 可复现性 | 确定性结果 | 需要固定 `seed` 来复现 |

#### 9.3 白化变换（Whitening Transform）

```sql
SELECT WHITEN(vector_data) AS whitened_vector
FROM analytics.correlated_data;
```

**定义**：去除向量各分量之间的相关性，并将方差归一化为 1。白化后的数据各维度独立且方差相等。

**为什么重要**：很多机器学习算法假设输入特征是不相关的。白化变换是满足这一假设的标准预处理步骤。

#### 9.4 归一化（Normalization）

```sql
SELECT NORMALIZE(vector_data) AS unit_vector
FROM analytics.unnormalized_data;
```

**定义**：将向量归一化为单位向量（长度为 1）。通常使用 L2 归一化。

**为什么重要**：余弦相似度要求向量是归一化的；很多索引算法（如 HNSW）在归一化向量上表现更好。

---

### 10. 线性代数操作（Linear Algebra Operations）

VQL 直接内建了三个核心线性代数运算。

#### 10.1 矩阵乘法

```sql
SELECT MATMUL(matrix1, matrix2) AS product;
```

**用途**：两个矩阵相乘。在向量变换中，矩阵乘法是所有线性变换的基础。

#### 10.2 特征分解（Eigendecomposition）

```sql
SELECT EIGENDECOMPOSE(correlation_matrix) AS (
    eigenvalues,
    eigenvectors
);
```

**用途**：将方阵分解为特征值和特征向量。PCA 的底层实现就是对协方差矩阵做特征分解。

**返回值**：
- `eigenvalues`：特征值向量
- `eigenvectors`：特征向量矩阵（列为特征向量）

#### 10.3 奇异值分解（SVD）

```sql
SELECT SVD(data_matrix) AS (U, S, V);
```

**用途**：将任意矩阵分解为 U * S * V^T。SVD 是比特征分解更通用的矩阵分解方法（适用于非方阵）。

**直觉建立**：SVD 可以理解为把任何线性变换拆解成三步基本操作：先**旋转**（V^T，调整坐标系的朝向）→ 再**拉伸/压缩**（S，沿各轴缩放，奇异值就是每个轴的缩放倍数）→ 最后再**旋转**（U，调整到输出空间的朝向）。就像把一个复杂的舞蹈动作拆解成"转身→伸展→再转身"三个基本步骤。奇异值 S 从大到小排列，最大的几个代表数据最核心的变化方向，最小的接近零的可以丢掉——这就是 SVD 做降维的原理。与特征分解的关系：特征分解只能用于方阵，SVD 对任意矩阵都有效，所以 PCA 的实现通常优先选 SVD（数值更稳定）。

**返回值**：
- `U`：左奇异向量矩阵
- `S`：奇异值（通常是对角矩阵）
- `V`：右奇异向量矩阵

> **PCA vs 特征分解 vs SVD 的关系**：PCA 的数学实现可以通过对协方差矩阵做特征分解，也可以直接对数据矩阵做 SVD（通常更数值稳定）。VQL 同时提供这三者，让研究者可以选择最适合的抽象层级。

---

### 11. 非线性变换——双曲空间操作（Hyperbolic Space Operations）

**定义**：双曲空间是一种常曲率为负的非欧几何空间，天然适合表示层次结构（hierarchical data）。VQL 提供了完整的双曲空间操作集。

**直觉建立**：想象一个**漏斗形的地图**。漏斗的窄口处（中心）代表树的根节点，空间很紧凑——根节点只有一个，不需要太多"面积"。但越往漏斗的宽口处（边缘）走，可用空间**指数级膨胀**——恰好对应树结构中叶节点数量随层级指数增长的特性。在普通的平面地图（欧氏空间）上，如果你要画一棵有 1000 个叶节点的树，你需要非常大的画布才能让节点不重叠。但在漏斗形地图上，边缘自然有足够的空间容纳海量叶节点，而且从根到叶的距离由漏斗的"深度"自然编码。这就是为什么双曲空间能用低维度（比如 50 维甚至 10 维）高效表示在欧氏空间中需要数百维才能编码的层次结构。关键边界：双曲空间专门适合**树形/层次结构**数据；如果数据没有层次关系（比如均匀分布的点云），用双曲空间反而会增加不必要的复杂性。

**为什么需要双曲空间**：欧氏空间中，圆的周长与半径成正比（C = 2πr）。但双曲空间中，圆的周长随半径**指数增长**。这意味着双曲空间可以用较低维度容纳更多的"距离"，非常适合表示树形结构——因为树的节点数也随深度指数增长。

#### 11.1 Poincare 球投影

```sql
SELECT POINCARE_PROJECTION(vector_data, radius=1.0) AS hyperbolic_vector
FROM hierarchy.tree_structures;
```

**定义**：将欧氏空间中的向量投影到 Poincare 球模型中。Poincare 球是单位球内部（|x| < 1），球心附近代表层次结构的根部，越靠近球面代表越底层的叶节点。

**适用场景**：分类法（taxonomy）嵌入、组织架构嵌入、知识图谱的层次关系建模。

#### 11.2 双曲距离计算

```sql
SELECT HYPERBOLIC_DISTANCE(
    vector1,
    vector2,
    model='poincare'
) AS h_distance;
```

**定义**：在指定的双曲模型中计算两点之间的距离。双曲距离与欧氏距离不同——越靠近球面边缘，同样的坐标差异对应的双曲距离越大。

#### 11.3 Mobius 加法

```sql
SELECT MOBIUS_ADD(
    hyperbolic_vec1,
    hyperbolic_vec2,
    curvature=-1.0
) AS combined_vector;
```

**定义**：双曲空间中的"加法"操作。在欧氏空间中，向量加法是直截了当的；但在双曲空间中，由于空间弯曲，"加法"需要用 Mobius 变换来定义。

**直觉建立**：在平面上，你从 A 点向东走 3 步到 B，再从 B 向北走 4 步到 C，最终位置很好算（直角三角形）。但如果你在地球表面做同样的事——从赤道上某点向东走 3000 公里再向北走 4000 公里——由于地球是弯曲的，最终位置不能用简单的平面向量加法算出来，你需要球面几何的公式。Mobius 加法就是双曲空间版本的"考虑了弯曲的加法"。越靠近 Poincare 球的边缘（球面），弯曲效应越强烈，同样坐标长度的位移对应越远的"真实距离"。

**参数说明**：`curvature=-1.0` 指定双曲空间的曲率。曲率越负，空间弯曲得越厉害。

#### 11.4 指数映射与对数映射

```sql
-- 指数映射：切空间 → 双曲空间
SELECT EXPMAP(tangent_vector, base_point, model='poincare') AS hyperbolic_point;

-- 对数映射：双曲空间 → 切空间
SELECT LOGMAP(hyperbolic_point, base_point, model='poincare') AS tangent_vector;
```

**定义**：
- **指数映射（Exponential Map）**：将切空间（tangent space）中的向量映射到双曲空间上的点。切空间是流形上某点处的局部欧氏近似。
- **对数映射（Logarithmic Map）**：指数映射的逆操作，将双曲空间上的点映射回切空间。

**直觉建立**：想象你站在一个弯曲的山坡上（双曲空间），脚下放一块小平板（切空间）。平板贴着山坡与地面相切，在你脚下的一小片区域内，平板和山坡几乎完全重合——这就是"局部欧氏近似"。**指数映射**就像是把平板上画的一个方向箭头"贴"到山坡上，沿着山坡的曲面走出去，最终到达山坡上的一个实际位置。**对数映射**则是反过来，给你山坡上的两个点，告诉你"如果站在第一个点往平板上画箭头，应该画多长、指向哪里，才能沿着山坡走到第二个点"。为什么需要这对操作？因为梯度下降等优化算法是在平面（欧氏空间）上定义的，你必须先"展平"到切空间算梯度，再"贴回"弯曲空间更新位置。

**为什么重要**：很多优化算法（如梯度下降）在欧氏空间中工作。要在双曲空间中做优化，通常需要：先用对数映射回到切空间 → 在切空间中计算梯度 → 用指数映射回到双曲空间。

---

### 12. 非线性变换——流形操作（Manifold Operations）

#### 12.1 黎曼优化（Riemannian Optimization）

```sql
OPTIMIZE ON MANIFOLD 'hyperbolic'
OBJECTIVE minimize_hyperbolic_distance(source_vectors, target_vectors)
USING riemannian_gradient_descent(learning_rate=0.01, max_iterations=1000);
```

**定义**：在指定的流形上直接做优化，而不是在欧氏空间中。黎曼梯度下降是经典梯度下降在弯曲空间中的推广。

**直觉建立**：普通梯度下降就像在平地上找最低点——每一步朝"最陡下坡方向"走一步。但如果"地面"本身是弯曲的（比如你在一个碗的内壁上滚球），你不能假装地面是平的然后走直线——球会飞离碗壁，到达一个不在碗上的位置。黎曼梯度下降就是让球"贴着碗壁"走：每一步先在当前位置的切平面上计算下坡方向（黎曼梯度），然后沿着碗壁的曲面（测地线方向）移动到新位置（通过指数映射），保证始终留在碗壁（流形）上。

**为什么重要**：如果数据天然存在于弯曲空间（如层次关系、社交网络），直接在流形上优化比先映射到欧氏空间再优化效果更好。

#### 12.2 平行传输（Parallel Transport）

```sql
SELECT PARALLEL_TRANSPORT(
    vector,
    start_point,
    end_point,
    manifold='poincare'
) AS transported_vector;
```

**定义**：沿着测地线（manifold 上两点间的"最短路径"），将一个切向量从起点"搬运"到终点，同时保持向量与测地线的相对关系不变。

**直觉理解**：想象你站在地球北极，手里拿一支水平箭头指向右边。你沿着经线走到赤道，如果箭头始终保持与你的行进方向的相对角度不变——这就是平行传输。在弯曲空间中，平行传输后的向量方向可能发生变化（这就是曲率的体现）。

#### 12.3 测地线插值（Geodesic Interpolation）

```sql
SELECT GEODESIC_INTERPOLATE(
    point1,
    point2,
    t=0.5,
    manifold='poincare'
) AS midpoint;
```

**定义**：沿着两点之间的测地线（最短路径）做插值。`t=0.5` 表示找中点，`t=0` 是 point1，`t=1` 是 point2。

**与欧氏线性插值的对比**：在欧氏空间中，线性插值走的是直线；在弯曲流形上，测地线插值走的是"弯曲的最短路径"。

**适用场景**：
- 嵌入空间中的平滑过渡（如在两个概念之间生成中间概念）
- 层次结构中两个节点之间的路径分析

---

### 13. 高级非线性变换（Advanced Nonlinear Transforms）

#### 13.1 核变换（Kernel Transformations）

```sql
SELECT KERNEL_TRANSFORM(
    vector_data,
    kernel='rbf',
    gamma=0.1
) AS kernel_space;
```

**定义**：使用核函数（如 RBF 径向基函数）将向量映射到高维核空间。核心思路是"核技巧"（kernel trick）——在低维空间中线性不可分的数据，映射到高维空间后可能变得线性可分。

**直觉建立**：想象一张桌子上混杂着红色和蓝色的弹珠，红的在中间围成一圈，蓝的在外圈。你无论怎么在桌面上画一条直线，都不可能把红蓝完美分开——这就是"线性不可分"。但如果你猛拍桌子，弹珠弹到空中——中间的红色弹珠弹得更高，外圈的蓝色弹得低。现在在三维空间里，你可以用一个水平面完美切开红蓝两组！"拍桌子"就是核变换——把二维数据映射到三维（或更高维），让原本缠绕在一起的数据变得可分。RBF 核就是一种特定的"拍法"，`gamma` 控制拍的力度——gamma 越大，弹珠弹起的高度差异越集中在近邻之间。核技巧的妙处在于你甚至不需要真的计算高维坐标，只需计算点对之间的核函数值就够了。

**参数说明**：
- `kernel='rbf'`：使用 RBF（高斯）核
- `gamma=0.1`：RBF 核的宽度参数，gamma 越大，影响范围越小

#### 13.2 流形学习（Manifold Learning）

VQL 内建了三种主流的流形学习（非线性降维）算法：

**直觉建立**：PCA 降维就像把一张摊开的报纸从三维压到二维——报纸本来就是平的，投影很自然。但如果数据不是"平的"而是"卷的"呢？想象一卷被展开到一半的瑞士卷蛋糕（Swiss Roll），数据点分布在这个卷曲的曲面上。如果你用 PCA 从上往下拍平，原本在蛋糕不同层上的点会被压叠在一起——距离关系全乱了。流形学习的思路是：先沿着蛋糕的表面测量点与点之间的"沿面距离"（而非直线穿透距离），然后找到一种二维展开方式，使得展开后的平面距离尽量保持原来的沿面距离。三种方法的区别在于"怎么测量沿面距离"和"保持哪种距离关系"。

**ISOMAP**：
```sql
SELECT ISOMAP(vector_data, n_neighbors=5, n_components=2) AS manifold_vector;
```
保持测地线距离的非线性降维。先构建近邻图，再用最短路径近似测地线距离。

**UMAP**：
```sql
SELECT UMAP(vector_data, n_neighbors=15, min_dist=0.1) AS umap_vector;
```
基于拓扑数据分析的降维方法。在保持全局结构和局部结构之间取得平衡。

**t-SNE**：
```sql
SELECT TSNE(vector_data, perplexity=30, n_components=2) AS tsne_vector;
```
专为可视化设计的降维方法。擅长在 2D/3D 中展示高维数据的聚类结构。

**三种流形学习方法的对比**：

| 方法 | 保持什么 | 适合什么 | 参数关键点 |
|------|---------|---------|-----------|
| **ISOMAP** | 测地线距离（全局结构） | 数据位于低维流形上的情况 | `n_neighbors` 控制局部图构建 |
| **UMAP** | 全局 + 局部结构的平衡 | 通用降维和可视化 | `n_neighbors`（局部性）、`min_dist`（点间最小距离） |
| **t-SNE** | 局部邻域结构 | 聚类可视化（2D/3D） | `perplexity`（有效邻居数） |

> **注意**：t-SNE 的结果中，不同聚类之间的距离没有意义——只有同一聚类内部的相对位置有意义。不要用 t-SNE 的结果来判断两个聚类是否"相近"。

> **常见误用**：把 t-SNE 的降维结果当作下游任务（如聚类、分类）的特征输入。t-SNE 设计目标是可视化，不保持全局距离关系，且结果受 `perplexity` 参数影响极大——换个 perplexity 值，同一数据可以画出完全不同的"聚类"形态。如果需要降维后的特征做后续计算，应该用 UMAP（更稳定且保持全局结构）或 PCA（线性但可复现）。

---

### 14. 空间转换（Space Conversions）

#### 14.1 双曲模型间转换

```sql
SELECT CONVERT_SPACE(
    hyperbolic_vector,
    from='poincare',
    to='lorentz'
) AS lorentz_vector;
```

**定义**：在不同的双曲空间模型之间转换。Poincare 球模型和 Lorentz（双曲面）模型是同一双曲空间的两种不同数学表示，各有计算优势。

**为什么需要转换**：Poincare 模型直观适合可视化，Lorentz 模型数值更稳定适合训练。实际工作中经常需要在两者之间切换。

#### 14.2 不同曲率间投影

```sql
SELECT CURVATURE_PROJECT(
    vector_data,
    source_curvature=-1.0,
    target_curvature=-0.5
) AS projected;
```

**定义**：将向量从一个曲率的双曲空间投影到另一个曲率的双曲空间。

**为什么需要**：不同的数据集或模型可能使用不同的曲率参数。曲率是双曲嵌入中的超参数，跨模型迁移时需要曲率适配。

---

### 15. 向量函数与聚合（Vector Functions and Aggregations）

#### 15.1 基本向量函数

| 函数 | 语法 | 说明 |
|------|------|------|
| `DIMENSION()` | `SELECT DIMENSION(embedding) AS vector_dim;` | 获取向量的维度数 |
| `DISTANCE()` | `SELECT DISTANCE(vector1, vector2, 'cosine') AS similarity;` | 计算两个向量之间的距离 |
| `CONCAT()` | `SELECT CONCAT(vector1, vector2) AS combined_vector;` | 拼接两个向量为一个更长的向量 |
| `DOT()` | `SELECT DOT(vector1, vector2) AS dot_product;` | 计算两个向量的点积（标量结果） |

#### 15.2 向量算术

```sql
-- 向量加法
SELECT vector_data + influence_vector AS adjusted_vector;

-- 标量乘法
SELECT vector_data * scalar_value AS scaled_vector;
```

**核心思想**：VQL 让向量算术像普通数字运算一样直观——直接用 `+` 和 `*` 操作符。

**适用场景**：
- 向量加法：给用户向量加入偏好偏移量（如推荐中的"喜欢"信号）
- 标量乘法：调整向量的强度/权重

#### 15.3 向量聚合

**质心计算（Centroid）**：

```sql
SELECT AVG(user_vector) AS centroid
FROM analytics.user_profiles
GROUP BY user_category;
```

**定义**：对每个用户类别计算向量平均值（质心）。质心是一组向量的"重心"，代表该组的典型方向。

**中位数向量（Median Vector）**：

```sql
SELECT MEDIAN_VECTOR(feature_vector) AS representative
FROM analytics.product_features
GROUP BY product_category;
```

**定义**：计算中位数向量作为代表向量。中位数向量比均值向量更鲁棒——不受离群点影响。

**质心 vs 中位数向量**：

| 方法 | 受离群点影响 | 计算成本 | 适用场景 |
|------|------------|---------|---------|
| `AVG()`（质心） | 大 | 低 | 数据分布均匀时 |
| `MEDIAN_VECTOR()` | 小 | 高 | 数据有噪声或离群点时 |

---

### 16. 聚类与分析（Clustering and Analytics）

#### 16.1 K-means 聚类

```sql
CLUSTER customer_data.behavior_vectors
USING KMEANS
CLUSTERS 5
OUTPUT AS customer_segments;
```

**定义**：对行为向量执行 K-means 聚类，将数据划分为 5 个簇。结果输出为 `customer_segments`（包含聚类分配或质心）。

**适用场景**：用户分群、商品分类、行为模式发现。

#### 16.2 层次聚类（Hierarchical Clustering）

```sql
CLUSTER content.document_vectors
USING HIERARCHICAL
LINKAGE 'ward'
OUTPUT AS doc_hierarchy;
```

**定义**：对文档向量执行层次聚类，使用 Ward 链接方法。输出一个树状图（dendrogram）或聚类分配。

**K-means vs 层次聚类**：

| 维度 | K-means | 层次聚类 |
|------|---------|---------|
| 需要预设 K | 是 | 否（可以后期选择切分层级） |
| 输出结构 | 扁平的 K 个簇 | 树状层次结构 |
| 可解释性 | 中等 | 高（可以看到合并过程） |
| 计算成本 | O(nKt) | O(n^2 log n) 或更高 |
| 适用场景 | 大规模数据快速分群 | 需要理解层次关系的场景 |

#### 16.3 多样性采样（Diversity Sampling）

```sql
SELECT *
FROM search.result_vectors
SIMILARITY SEARCH [query_vector]
WITH DIVERSITY FACTOR 0.3
TOP K 10;
```

**定义**：在相似性搜索中引入多样性因子。`DIVERSITY FACTOR 0.3` 意味着在选择 Top K 结果时，不仅考虑相似度，还要保证结果之间的多样性。

**为什么重要**：纯相似性搜索可能返回一堆高度相似的结果（如 10 个几乎一样的红色运动鞋）。多样性采样确保结果覆盖更广的子空间。

**适用场景**：
- 搜索结果去重/去相似
- 推荐系统中避免"信息茧房"
- 探索性搜索

> **常见误用**：盲目设置过高的 `DIVERSITY FACTOR`（如 0.8 以上）。多样性因子越高，结果偏离纯相似性排序越远——极端情况下返回的结果与查询几乎无关，只是彼此不同。正确做法：从 0.1-0.3 的保守值开始，通过 A/B 测试评估用户满意度后再逐步调高。

#### 16.4 异常检测（Outlier Detection）

```sql
SELECT *, OUTLIER_SCORE(behavior_vector) AS anomaly_score
FROM analytics.user_behavior
WHERE OUTLIER_SCORE(behavior_vector) > 0.95;
```

**定义**：计算每个向量的异常分数。分数越高，该向量与其他向量越不同。阈值 0.95 意味着只标记前 5% 最异常的数据点。

**适用场景**：
- 欺诈检测
- 网络入侵检测
- 产品质量异常识别
- 用户行为异常监控

---

### 17. 索引管理（Index Management）

#### 17.1 创建向量索引

**HNSW 索引**：

```sql
CREATE INDEX product_vectors_idx
ON ecommerce.product_vectors
USING ALGORITHM 'hnsw'
WITH (
    M = 16,
    ef_construction = 200,
    distance_metric = 'cosine'
);
```

**参数说明**：
- `M = 16`：每个节点的最大连接数。M 越大，搜索精度越高，但内存消耗也越大
- `ef_construction = 200`：构建索引时的动态候选列表大小。越大构建越慢但索引质量越高
- `distance_metric = 'cosine'`：距离度量

**复合索引（IVFFlat）**：

```sql
CREATE INDEX category_vector_idx
ON ecommerce.product_vectors (category, vector_data)
USING ALGORITHM 'ivfflat'
WITH (lists = 1000);
```

**定义**：同时在元数据列（`category`）和向量列上创建索引。IVFFlat 将向量空间划分为 1000 个倒排列表（lists），搜索时先根据 category 过滤，再在对应的分区中做向量搜索。

**HNSW vs IVFFlat**：

| 维度 | HNSW | IVFFlat |
|------|------|---------|
| 搜索速度 | 更快（对数级） | 较快（线性于探测的 list 数） |
| 索引构建速度 | 较慢 | 较快 |
| 内存消耗 | 较高 | 较低 |
| 动态更新 | 支持 | 需要重建 |
| 适合场景 | 高频搜索、需要动态更新 | 大数据量、可以离线构建 |

#### 17.2 索引优化

```sql
-- 重建索引（使用新参数）
REBUILD INDEX product_vectors_idx
WITH (M = 32, ef_construction = 400);

-- 分析索引性能
ANALYZE INDEX product_vectors_idx;
```

**核心思想**：
- `REBUILD INDEX`：用新参数重建现有索引（如提高 M 和 ef_construction 来换取更高精度）
- `ANALYZE INDEX`：收集索引的统计信息和性能指标，帮助决定是否需要调优

---

### 18. 高级特性（Advanced Features）

#### 18.1 自定义变换函数

```sql
CREATE TRANSFORM normalize_and_scale AS (
    SELECT NORMALIZE(vector_data) * scale_factor AS result
    WHERE scale_factor = 1.5
);
```

**定义**：创建一个可复用的自定义变换——先归一化再乘以 1.5 的缩放因子。

#### 18.2 复合变换

```sql
CREATE TRANSFORM feature_extractor AS
CHAIN(
    NORMALIZE(),
    PCA(dimensions=50),
    POINCARE_PROJECTION(radius=1.0)
);
```

**定义**：用 `CHAIN()` 将多个变换串成一个命名管道。这个 `feature_extractor` 依次执行：归一化 → PCA 降维到 50 维 → 投影到 Poincare 球。

**为什么重要**：复杂的变换管道在实验中会反复使用。将其封装为命名变换后，可以像调用一个函数一样使用，大幅简化查询语句。

#### 18.3 学习最优变换（Learn Optimal Transformations）

```sql
OPTIMIZE TRANSFORM learning_projection
OBJECTIVE minimize_distance(source_vectors, target_vectors)
USING gradient_descent(learning_rate=0.01, max_iterations=1000);
```

**定义**：通过优化算法自动学习一个变换矩阵，使得变换后的 source_vectors 尽量接近 target_vectors。这本质上是在 VQL 中做**度量学习（metric learning）**。

**适用场景**：
- 跨模态对齐（如文本嵌入和图像嵌入映射到同一空间）
- 领域自适应（将通用嵌入适配到特定领域）

#### 18.4 验证变换（Validate Transformations）

```sql
VALIDATE TRANSFORM my_transform
CHECK (
    orthogonality = TRUE,
    determinant != 0,
    condition_number < 100
);
```

**定义**：对变换矩阵做数学性质验证。

**检查项说明**：
- `orthogonality = TRUE`：正交矩阵保持向量长度和角度不变
- `determinant != 0`：行列式非零确保变换可逆
- `condition_number < 100`：条件数衡量矩阵的数值稳定性，越小越好。条件数过大意味着微小的输入变化会导致输出剧烈变化

---

### 19. 类型系统（Type System）

VQL 定义了一套专用的类型系统来确保类型安全。

#### 19.1 向量类型

| 类型 | 说明 |
|------|------|
| `VECTOR(dim)` | 固定维度的向量（如 `VECTOR(768)`） |
| `DYNAMIC_VECTOR` | 可变维度的向量 |
| `EMBEDDING` | 带元数据的特殊向量 |
| `HYPERBOLIC_VECTOR(model, dim)` | 双曲空间中的向量（指定模型和维度） |
| `MANIFOLD_POINT(type, coords)` | 特定流形上的点 |

#### 19.2 变换类型

| 类型 | 说明 |
|------|------|
| `MATRIX(rows, cols)` | 固定维度的矩阵 |
| `TRANSFORM` | 变换函数类型 |
| `TRANSFORM_CHAIN` | 复合变换类型 |
| `DISTANCE` | 相似度/距离的数值类型 |

#### 19.3 自定义空间定义

```sql
-- 定义自定义空间
CREATE SPACE hyperbolic_space (
    model = 'poincare',
    curvature = -1.0,
    dimension = 50
);

-- 在自定义空间中验证约束
VALIDATE IN hyperbolic_space
CHECK (
    point_norm < 1.0,
    satisfies_hyperbolic_constraints = TRUE
);
```

**定义**：`CREATE SPACE` 定义一个具名的数学空间（带模型、曲率、维度等属性）。`VALIDATE IN` 检查向量是否满足该空间的约束。

**为什么重要**：Poincare 球模型要求所有点的范数 < 1.0。如果某个操作产生了范数 >= 1 的点，说明计算有误。类型系统和约束验证让这类错误在 VQL 层面就被捕获，而非在应用层面引发莫名其妙的 bug。

---

### 20. 错误处理与验证（Error Handling and Validation）

#### 20.1 维度不匹配处理

```sql
ON DIMENSION MISMATCH
THEN PAD_ZEROS | TRUNCATE | REJECT;
```

**定义**：当向量维度不匹配时的处理策略：

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| `PAD_ZEROS` | 用零填充到目标维度 | 旧版本嵌入迁移到新维度时 |
| `TRUNCATE` | 截断到目标维度 | 降维场景（但可能损失信息） |
| `REJECT` | 抛出错误，拒绝操作 | 严格模式，不容忍任何不匹配 |

> **常见误用**：在生产环境中默认使用 `PAD_ZEROS` 处理维度不匹配，认为"补零总比报错好"。实际上，零填充会让高维空间中的向量偏向原点方向，严重扭曲相似度计算结果——两个本来语义无关的向量，补零后可能因为零分量大量重叠而被判为"相似"。正确做法：开发阶段用 `REJECT` 尽早暴露问题，只在明确理解后果的迁移场景中才用 `PAD_ZEROS`，并且迁移后应尽快重新编码。

#### 20.2 无效向量处理

```sql
ON INVALID VECTOR
THEN NULL | REJECT | DEFAULT [default_vector];
```

**定义**：当向量包含 NaN 或无穷大值时的处理策略：

| 策略 | 行为 |
|------|------|
| `NULL` | 返回 NULL |
| `REJECT` | 拒绝操作 |
| `DEFAULT [default_vector]` | 替换为预设的默认向量 |

#### 20.3 质量保证函数

```sql
-- 向量质量评分
SELECT *
FROM analytics.user_data
WHERE VECTOR_QUALITY(user_vector) > 0.9;

-- 检查向量有效性
SELECT *
FROM analytics.processed_data
WHERE IS_VALID_VECTOR(vector_data) = TRUE;
```

- `VECTOR_QUALITY()`：返回向量的质量评分（0-1），评分标准包括是否包含异常值、分布是否合理等
- `IS_VALID_VECTOR()`：布尔函数，检查向量是否有效（无 NaN、无无穷大）

---

### 21. 性能优化考量（Performance Considerations）

#### 21.1 查询优化

- **惰性求值（Lazy Evaluation）**：向量操作不会立即执行，而是构建执行计划后统一优化执行
- **自动批处理（Automatic Batching）**：大规模操作自动拆分为批次
- **索引感知查询规划（Index-Aware Query Planning）**：查询优化器根据可用索引选择最优执行路径
- **并行处理能力（Parallel Processing）**：利用多核/GPU 并行执行向量运算

#### 21.2 缓存策略

- **高频向量缓存**：频繁访问的向量保留在内存中
- **变换结果缓存**：重复的变换操作结果被缓存
- **查询计划缓存**：相同结构的查询复用已优化的执行计划

#### 21.3 内存管理

- **流式处理（Streaming Processing）**：大数据集不全部加载到内存，而是流式处理
- **高效向量存储格式**：使用紧凑的二进制格式存储向量
- **临时向量垃圾回收**：计算过程中产生的临时向量在不需要后自动回收

---

### 22. 实现指南（Implementation Guidelines）

#### 22.1 后端集成

- 支持多种向量数据库后端（如 Pinecone、Milvus、Weaviate、pgvector 等）
- 可插拔的距离度量实现（允许用户自定义距离函数）
- 可扩展的变换框架（允许注册自定义变换）

#### 22.2 API 设计

- **RESTful API**：支持远程查询执行
- **流式结果返回**：大结果集分批流式返回，避免客户端内存溢出
- **异步查询执行**：长时间运行的查询（如聚类、优化）可以异步提交，后台执行

#### 22.3 安全与访问控制

- **基于角色的访问控制（RBAC）**：对 Collection 和 Table 级别做权限管理
- **查询级安全策略**：可以对特定类型的查询设置安全限制
- **向量数据加密**：支持存储和传输加密

---

### 23. 综合用例示例（Example Use Cases）

#### 23.1 推荐系统

```sql
SELECT product_id, product_name, similarity_score
FROM ecommerce.product_vectors
SIMILARITY SEARCH (
    SELECT user_vector FROM analytics.user_profiles WHERE user_id = 12345
)
WHERE category IN ('electronics', 'books')
TOP K 20;
```

**解析**：这是一个嵌套查询——内层查询获取用户 12345 的向量，外层用这个向量在商品向量表中搜索，限定品类为电子产品或书籍，返回最相似的 20 个商品。这展示了 VQL 支持子查询的能力。

#### 23.2 语义搜索

```sql
SELECT document_id, title, relevance
FROM content.document_vectors
HYBRID SEARCH (
    VECTOR [text_vector] WEIGHT 0.8,
    TEXT "artificial intelligence" WEIGHT 0.2
)
WHERE publication_date > '2020-01-01'
TOP K 50;
```

**解析**：混合搜索的实战用法——向量语义相似度占 80%，关键词精确匹配占 20%，再叠加时间过滤。这是构建 RAG 系统检索模块的典型模式。

#### 23.3 异常检测

```sql
SELECT user_id, behavior_vector, anomaly_score
FROM analytics.user_behavior
WHERE OUTLIER_SCORE(behavior_vector) > 0.95
ORDER BY anomaly_score DESC;
```

**解析**：对用户行为向量计算异常分数，筛选出最异常的 5%，按异常程度降序排列。可以直接用于欺诈检测或安全监控系统。

#### 23.4 层次聚类（双曲嵌入）

```sql
SELECT product_id, category,
    HYPERBOLIC_HIERARCHY_EMBED(feature_vector, depth=3) AS hierarchical_pos
FROM ecommerce.product_features
ORDER BY HYPERBOLIC_TREE_DISTANCE(hierarchical_pos, root_category);
```

**解析**：这是最能体现 VQL 独特价值的用例——将商品特征向量嵌入到双曲空间的层次结构中（`HYPERBOLIC_HIERARCHY_EMBED`，深度 3 层），然后按照双曲树距离（`HYPERBOLIC_TREE_DISTANCE`）排序。这样可以自动发现商品的层次分类关系，从根类目到叶类目。

**为什么用双曲空间**：商品类目天然是树形层次结构，双曲空间能够以低维表示高效编码这种树结构，而欧氏空间在表示层次关系时维度效率很低。

---

### 24. VQL 的定位总结（Conclusion）

**VQL 的核心定位**：

1. **熟悉性**：保持 SQL 语法风格，降低学习门槛
2. **向量原生**：提供完整的向量操作原语（搜索、变换、聚类、索引）
3. **数学深度**：支持线性代数、双曲几何、流形操作等高级数学运算
4. **统一抽象**：跨向量数据库厂商的统一查询语言
5. **从实验到生产**：同一套语言从研究者的探索性分析到生产系统的批量处理都适用

> **这是草案（Draft Specification）**：VQL 目前还处于草案阶段，GitHub 仓库尚未公开。这意味着语法和功能可能会在正式发布前发生变化。但其设计思路——为向量数据库提供类 SQL 的统一查询语言——代表了行业的一个重要趋势。

---

## VQL 操作速查表

| 操作类型 | 关键语法 | 功能 |
|---------|---------|------|
| 相似性搜索 | `SIMILARITY SEARCH [...] USING METRIC ... TOP K ...` | 找最相似的 K 个向量 |
| 混合搜索 | `HYBRID SEARCH (VECTOR ... WEIGHT ..., TEXT ... WEIGHT ...)` | 向量 + 文本联合搜索 |
| 范围搜索 | `RANGE SEARCH [...] THRESHOLD ...` | 找阈值内所有向量 |
| 批量搜索 | `BATCH SIMILARITY SEARCH (...)` | 多查询向量批量搜索 |
| 定义变换 | `CREATE TRANSFORM ... AS MATRIX [...]` | 创建命名变换矩阵 |
| 应用变换 | `TRANSFORM(data, matrix)` | 应用单个变换 |
| 链式变换 | `TRANSFORM_CHAIN(data, [t1, t2, t3])` | 依次应用多个变换 |
| PCA 降维 | `PCA(data, dimensions=N)` | 主成分分析降维 |
| 随机投影 | `RANDOM_PROJECTION(data, output_dim=N)` | 随机矩阵降维 |
| 归一化 | `NORMALIZE(data)` | L2 归一化为单位向量 |
| 白化 | `WHITEN(data)` | 去相关 + 方差归一 |
| Poincare 投影 | `POINCARE_PROJECTION(data, radius=1.0)` | 投影到双曲空间 |
| 双曲距离 | `HYPERBOLIC_DISTANCE(v1, v2, model)` | 双曲空间距离计算 |
| Mobius 加法 | `MOBIUS_ADD(v1, v2, curvature)` | 双曲空间向量加法 |
| 指数映射 | `EXPMAP(tangent, base, model)` | 切空间 → 流形 |
| 对数映射 | `LOGMAP(point, base, model)` | 流形 → 切空间 |
| 黎曼优化 | `OPTIMIZE ON MANIFOLD ...` | 流形上直接优化 |
| 平行传输 | `PARALLEL_TRANSPORT(v, start, end, manifold)` | 沿测地线传输向量 |
| 测地线插值 | `GEODESIC_INTERPOLATE(p1, p2, t, manifold)` | 流形上最短路径插值 |
| 核变换 | `KERNEL_TRANSFORM(data, kernel, gamma)` | 核技巧映射到高维 |
| ISOMAP | `ISOMAP(data, n_neighbors, n_components)` | 保测地线距离降维 |
| UMAP | `UMAP(data, n_neighbors, min_dist)` | 拓扑降维 |
| t-SNE | `TSNE(data, perplexity, n_components)` | 可视化降维 |
| 空间转换 | `CONVERT_SPACE(v, from, to)` | 双曲模型间转换 |
| 曲率投影 | `CURVATURE_PROJECT(v, source, target)` | 不同曲率间投影 |
| K-means | `CLUSTER ... USING KMEANS CLUSTERS N` | K-means 聚类 |
| 层次聚类 | `CLUSTER ... USING HIERARCHICAL LINKAGE 'ward'` | 层次聚类 |
| 多样性采样 | `WITH DIVERSITY FACTOR N` | 结果多样性控制 |
| 异常检测 | `OUTLIER_SCORE(vector)` | 计算异常分数 |
| 创建索引 | `CREATE INDEX ... USING ALGORITHM 'hnsw'/'ivfflat'` | 创建向量索引 |
| 重建索引 | `REBUILD INDEX ... WITH (...)` | 用新参数重建索引 |
| 分析索引 | `ANALYZE INDEX ...` | 收集索引性能指标 |
| 自定义空间 | `CREATE SPACE ... (model, curvature, dimension)` | 定义数学空间 |
| 维度处理 | `ON DIMENSION MISMATCH THEN ...` | 维度不匹配策略 |
| 向量验证 | `VECTOR_QUALITY()` / `IS_VALID_VECTOR()` | 质量检查 |

---

## 重点标记

1. **VQL 的核心价值**：为向量数据库提供类似 SQL 对关系数据库的统一抽象——跨厂商、对开发者友好、功能完整
2. **三类目标用户**：应用开发者（统一 API）、运维人员（统一 DDL/DML）、AI/ML 研究者（宏语言缩短实验周期）
3. **语法设计哲学**：保留 SQL 骨架（SELECT/FROM/WHERE/ORDER BY/LIMIT），新增 VECTOR_OPERATION 子句
4. **四种搜索操作**：相似性搜索（Top K）、带过滤的搜索（WHERE + THRESHOLD）、混合搜索（向量 + 文本）、范围搜索（阈值内全部）
5. **线性变换完整链条**：CREATE TRANSFORM → TRANSFORM → TRANSFORM_CHAIN + 内建的 PCA/随机投影/白化/归一化
6. **双曲空间操作是亮点**：Poincare 投影、双曲距离、Mobius 加法、指数/对数映射——专为层次结构数据设计
7. **流形操作非常前沿**：黎曼优化、平行传输、测地线插值——这些在大多数向量数据库中都没有原生支持
8. **流形学习三件套**：ISOMAP（保全局）、UMAP（平衡全局/局部）、t-SNE（可视化专用）
9. **类型系统的严谨性**：VECTOR/EMBEDDING/HYPERBOLIC_VECTOR/MANIFOLD_POINT 等专用类型 + CREATE SPACE 自定义空间 + VALIDATE 约束检查
10. **错误处理三策略**：维度不匹配（PAD_ZEROS/TRUNCATE/REJECT）、无效向量（NULL/REJECT/DEFAULT）、质量评分（VECTOR_QUALITY > 阈值）
11. **索引管理**：HNSW（搜索快、支持动态更新、内存高）vs IVFFlat（构建快、内存低、需重建）
12. **学习最优变换**：VQL 中可以直接做度量学习——OPTIMIZE TRANSFORM ... USING gradient_descent，这让跨模态对齐等任务可以在数据库层面完成
13. **草案状态**：VQL 尚未正式发布，语法可能变化，但设计理念值得关注——统一的向量查询语言是行业趋势

---

## 自测：你真的理解了吗？

> 以下问题考的是理解，不是记忆。试着先思考再看答案。

**Q1**：你的公司同时使用 Pinecone 做推荐系统、Milvus 做文档搜索、pgvector 做用户画像匹配。每次换一个厂商或新增一个向量数据库，开发团队都要重写大量代码。有人建议"写一个内部适配层统一接口"，另一人建议"等 VQL 成熟后直接采用"。结合 VQL 的设计哲学和当前状态，你会如何决策？两种方案各有什么风险？

**Q2**：一个同事在做商品分类体系的嵌入，数据结构是一棵深度为 5、叶节点约 10 万个的分类树。他打算用 1024 维的欧氏空间嵌入。你建议他考虑双曲空间嵌入，他问"有什么好处？维度选多少合适？"。你怎么回答？如果这棵分类树非常"扁平"（深度只有 2），你的建议会改变吗？

**Q3**：你需要对一个 1536 维的嵌入数据集做降维以加速搜索。数据可能存在非线性的流形结构。有三个选择：PCA 降到 256 维、UMAP 降到 256 维、随机投影降到 256 维。请分析每种方案在降维质量、计算成本、可复现性三个维度上的优劣，并说明在什么条件下你会选择每一种。

**Q4**：一个搜索系统使用 `SIMILARITY SEARCH ... TOP K 10` 返回推荐结果，但用户反馈"推荐的 10 个商品长得几乎一模一样"。你决定加入 `WITH DIVERSITY FACTOR`。如果你把 DIVERSITY FACTOR 设为 0.9，可能会出现什么问题？你会如何逐步调优这个参数？

**Q5**：你在 VQL 中写了一个链式变换 `TRANSFORM_CHAIN(data, [pca_matrix, rotation_matrix, poincare_projection])`，目的是先降维再旋转最后投影到双曲空间。执行后发现有部分向量在 Poincare 球模型中范数超过了 1.0（违反约束）。可能的原因是什么？你会用 VQL 的哪些机制来诊断和修复这个问题？
