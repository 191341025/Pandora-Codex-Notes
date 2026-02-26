# 模块四下：构建工具 - 协议与部署

> 对应 PDF 第 36-41 页

---

## 概念讲解

### 1. MCP（Model Context Protocol）— 工具连接的通用标准

**一句话**：MCP 让 Agent 能像插 USB 一样接入各种外部数据源和工具，不需要为每个都写定制集成。

#### MCP 的双向作用

| 方向 | 说明 | 在 ADK 中的实现 |
|------|------|----------------|
| **消费外部工具** | Agent 作为 MCP Client，使用第三方 MCP Server 暴露的工具 | ADK agent 可直接调用任何 MCP Server |
| **暴露本地工具** | 把你的 ADK 工具包装成 MCP Server，供外部使用 | 其他 MCP 兼容的 agent/应用可以安全调用你的工具 |

#### MCP 支持的数据源

```
MCP 就像一个通用适配器：

Agents ←→ MCP ←→ AlloyDB / MySQL / Postgres / BigQuery
                    Bigtable / Cloud SQL / Spanner / Dgraph...
```

> **Pro tip**：使用开源的 MCP Toolbox for Databases，可以轻松安全地连接 Agent 到各种数据库。

---

### 2. A2A（Agent2Agent Protocol）— Agent 间的通信标准

**一句话**：A2A 让不同的 Agent 能互相发现、通信、协作——不管它们是谁构建的、用什么框架。

#### A2A 的三个核心概念

| 概念 | 说明 | 类比 |
|------|------|------|
| **Agent Card** | Agent 的"数字名片"（JSON 文件），公布自己的能力、端点、认证要求 | LinkedIn 个人简介 |
| **Task-oriented** | 交互以"任务"为核心：Client Agent 发任务请求，Server Agent 处理并返回 | 工作委派 |
| **Modality agnostic** | 支持文本、音频、视频通信 | 多语言支持 |

#### A2A 的工作流

```
End User → Client Agent → 发现 Remote Agent（通过 Agent Card）
                        → 发送 Task 请求
                        → Remote Agent 处理
                        → 返回结果
```

**ADK Agent 原生支持 A2A**：自动暴露 HTTP 端点和 `agent.json` 文件。

#### 实际案例：BioCorteX 药物研发

BioCorteX 用 A2A 协议编排多 Agent 系统：
- 使用 Gemini 驱动的 Agent + ADK + GraphRAG
- 在 440 亿条连接的知识图谱上导航
- 从三个维度验证假设：生物学可行性 / 临床相关性 / 商业重要性
- **效果**：原来需要数年的工作，现在只需几天

---

### 3. 数据管理：三层记忆架构的 ADK 实现

模块二讲了概念，这里是具体实现：

| 记忆层 | ADK 中怎么用 | Google Cloud 服务 |
|--------|------------|-----------------|
| **长期记忆** | VertexAISearchToolset 做知识检索；Firestore 存对话历史；Cloud Storage 存原始文件；BigQueryToolset 做分析查询 | Vertex AI Search, Firestore, Cloud Storage, BigQuery |
| **工作记忆** | Memorystore 做高速缓存，避免重复昂贵的工具调用 | Memorystore |
| **事务记忆** | 工具执行关键操作后写 Cloud SQL 审计日志 | Cloud SQL, Cloud Spanner |

#### 前沿：记忆蒸馏（Memory Distillation）

**问题**：Agent 和用户的对话越来越长，全部作为上下文传给模型既贵又容易混乱。

**解决方案**：用 LLM 把长对话历史"蒸馏"成紧凑的关键事实集合。

```
原始对话（几千轮）→ 蒸馏 → 关键记忆（几十条事实）
"用户喜欢直飞航班"
"用户的预算上限是5000"
"用户的狗叫 Fido"
```

**Vertex AI Memory Bank 提供两种方式**：
- **自动蒸馏**：异步处理对话历史，自动提取关键事实（`GenerateMemories`）
- **Agent 主动蒸馏**：Agent 自己决定什么信息值得记住（`CreateMemory`）

---

### 4. 部署：从开发到生产

ADK 设计上是**部署无关的**——核心逻辑和部署环境解耦。

#### 部署流程

```
定义 Agent（Python）→ adk api_server 包装为 FastAPI → 容器化 → 部署
```

#### 三个主要部署目标

| 目标 | 特点 | 适合场景 |
|------|------|---------|
| **Vertex AI Agent Engine** | 全托管、自动扩缩、专为 Agent 优化 | ADK 用户的**推荐首选** |
| **Cloud Run** | Serverless、按需付费、适合微服务架构 | 融入现有微服务体系 |
| **GKE** | 最大控制力、GPU/TPU 支持、开源 K8s 生态 | 有平台工程团队的成熟公司 |

#### Vertex AI Agent Engine 的核心能力

**基础能力**：
- 自动扩缩容
- 安全和身份认证
- 框架无关（不限 ADK）
- Agent 生命周期管理（CRUD API）

**专用 Agent 功能**：
- **Memory Bank**：动态生成和检索长期个性化记忆
- **Example Store**：管理 few-shot 示例来改善 Agent 表现

> **注意区分**：Vertex AI Agent Builder 是**整套工具**（从发现到部署的全生命周期），Vertex AI Agent Engine 是其中**专门做部署运行**的托管服务。

---

## 问答记录

> 待补充

---

## 重点标记

1. **MCP = Agent 的"USB 接口"**：标准化工具连接，双向通信
2. **A2A = Agent 的"通信协议"**：让不同来源的 Agent 互相发现和协作
3. **Agent Card 是 A2A 的关键**：Agent 通过 JSON "名片" 公布自己的能力
4. **记忆蒸馏是前沿方向**：对话越来越长时，用 LLM 提取关键事实而非传全文
5. **ADK 部署推荐用 Vertex AI Agent Engine**：全托管、成本效益最高
6. **MCP Toolbox for Databases 是实用工具**：快速连接 Agent 到各种数据库
