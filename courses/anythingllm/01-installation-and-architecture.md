# 模块 01：安装部署与核心配置架构

> 对应源文件：installation-desktop/overview+system-requirements, installation-docker/overview+quickstart+system-requirements, setup/llm-configuration/overview, setup/embedder-configuration/overview, setup/vector-database-configuration/overview

---

## 概念地图

- **核心概念** (必须内化): 三大可插拔组件（LLM × Embedder × VectorDB）、三层 LLM 配置（System → Workspace → Agent）、组件作用域差异
- **实操要点** (动手时需要): Desktop vs Docker 选型、Docker 快速部署命令、Provider 选型决策
- **背景知识** (扩展理解): 系统资源需求、各 Provider 生态、镜像版本选择

---

## 概念讲解

### 1. Desktop vs Docker：两种部署形态

**定义**：AnythingLLM 提供两种部署方式——Desktop（桌面应用）和 Docker（容器化服务），功能集有明确差异。

**核心对比**：

| 维度 | Desktop | Docker |
|------|---------|--------|
| 安装方式 | 一键安装包（Win/Mac/Linux） | `docker run` 命令 |
| 使用人数 | **单人** | 单人或**多人** |
| 多用户权限 | 不支持 | 支持（Admin/Manager/Default） |
| 嵌入式聊天组件 | 不支持 | 支持 |
| 白标定制 | 不支持 | 支持（自定义 Logo、欢迎语） |
| 内置 LLM | 支持（可下载模型直接运行） | 不支持（需接入外部 LLM） |
| 密码保护 | 不支持 | 支持 |
| 访问方式 | 本地桌面应用 | 浏览器访问 `http://localhost:3001` |

**选型决策树**：

```
你需要多人同时使用吗？
├── 是 → Docker
└── 否 → 你需要对外发布聊天组件或 API 吗？
    ├── 是 → Docker
    └── 否 → 你想最简单地开始吗？
        ├── 是 → Desktop（一键安装，内置 LLM）
        └── 否 → Docker（更灵活、可远程访问）
```

> **关键差异**：Desktop 自带内置 LLM（可以下载模型直接本地运行），Docker 不带——但 Docker 有多用户、权限管理和嵌入组件等企业级功能。

### 2. 系统资源需求

**定义**：AnythingLLM 本身非常轻量，资源消耗主要取决于你选什么组件。

**最低配置**（仅 AnythingLLM 平台自身）：

| 资源 | 最低值 |
|------|--------|
| RAM | 2 GB |
| CPU | 2 核（任意架构） |
| 存储 | 5 GB |

**为什么这么轻量**：AnythingLLM 本质上是一个**编排层**——它不运行 LLM，不做向量计算（除非你选本地方案）。如果你把 LLM 指向 OpenAI、Embedding 指向 OpenAI、VectorDB 用内置 LanceDB，那 AnythingLLM 自身的资源消耗接近于零。

**资源消耗由组件决定**：

| 组件 | 云端方案资源影响 | 本地方案资源影响 |
|------|---------------|---------------|
| LLM | 零（API 调用） | 高（需要 GPU 或大量 CPU/RAM） |
| Embedding 模型 | 零（API 调用） | 中（CPU 向量化，大量文档时慢） |
| 向量数据库 | 零（托管服务） | 低（LanceDB 内置，几乎无额外消耗） |

> **实用建议**：如果你的机器没有 GPU，可以在另一台有 GPU 的机器上运行 Ollama，AnythingLLM 通过 API 远程连接——这样 AnythingLLM 本身只需要一台轻量机器甚至树莓派。

### 3. Docker 快速部署

**Linux / macOS**：

```bash
export STORAGE_LOCATION=$HOME/anythingllm && \
mkdir -p $STORAGE_LOCATION && \
touch "$STORAGE_LOCATION/.env" && \
docker run -d -p 3001:3001 \
--cap-add SYS_ADMIN \
-v ${STORAGE_LOCATION}:/app/server/storage \
-v ${STORAGE_LOCATION}/.env:/app/server/.env \
-e STORAGE_DIR="/app/server/storage" \
mintplexlabs/anythingllm:latest
```

**Windows (PowerShell)**：

```powershell
$env:STORAGE_LOCATION="$HOME\Documents\anythingllm"
If(!(Test-Path $env:STORAGE_LOCATION)) {New-Item $env:STORAGE_LOCATION -ItemType Directory}
If(!(Test-Path "$env:STORAGE_LOCATION\.env")) {New-Item "$env:STORAGE_LOCATION\.env" -ItemType File}
docker run -d -p 3001:3001 `
--cap-add SYS_ADMIN `
-v "$env:STORAGE_LOCATION`:/app/server/storage" `
-v "$env:STORAGE_LOCATION\.env:/app/server/.env" `
-e STORAGE_DIR="/app/server/storage" `
mintplexlabs/anythingllm:latest
```

启动后访问 `http://localhost:3001` 即可使用。数据持久化在 `$HOME/anythingllm/` 中，容器重建不会丢失。

**镜像选择**：
- `mintplexlabs/anythingllm:latest` — 标准版（内置 SQLite）
- `mintplexlabs/anythingllm:pg` — PostgreSQL 版（需自备 PG 数据库 + PGVector 扩展）

> **注意 UID/GID**：Docker 容器内默认 UID/GID 为 1000。如果你的宿主机用户 UID 不同，可能出现权限问题，需要在 `.env` 中调整。

---

### 4. 三大可插拔组件架构

**定义**：AnythingLLM 的核心架构由三大组件构成，各自独立可替换——这是理解整个系统配置的钥匙。

**直觉建立**：

把 AnythingLLM 想象成一个**万能插座面板**，上面有三个插孔：

1. **LLM 插孔**（大脑）：负责理解问题、生成回答。你可以插上 OpenAI、Anthropic、本地 Ollama，甚至不同 Workspace 插不同的"大脑"
2. **Embedding 插孔**（翻译官）：负责把文档文字翻译成向量（数字表示），让机器能"理解"语义。全系统共用一个翻译官
3. **VectorDB 插孔**（记忆库）：负责存储和检索向量化后的文档内容。全系统共用一个记忆库

三个插孔互相独立——换了 LLM 不影响已有的向量数据，换了 VectorDB 也不需要换 LLM。但 **Embedding 模型一旦选定就不建议轻易更换**，因为更换意味着所有已嵌入的文档都要重新处理。

> **类比边界**：现实中插座是标准化的，但这里的"插孔"对适配器有要求——并非所有 LLM Provider 都支持 Agent 工具调用，选 Agent LLM 时需要特别注意。

**三大组件的配置作用域**：

| 组件 | 作用域 | 能否按 Workspace 独立配置 | 更换成本 |
|------|--------|------------------------|---------|
| **LLM** | System / Workspace / Agent 三层 | **能**（每个 Workspace 可以不同） | 低（随时换） |
| **Embedding 模型** | 系统级 | **不能**（全局统一） | **高**（换了要重新嵌入所有文档） |
| **向量数据库** | 系统级 | **不能**（全局统一） | **高**（换了要重新嵌入所有文档） |

> **常见误用**：有人频繁切换 Embedding 模型或向量数据库（"想试试哪个效果好"），结果每次切换都导致已有文档变成"脏数据"——因为不同 Embedding 模型产出的向量维度和语义空间不同，新旧向量混在一起会导致搜索结果混乱。**一旦开始正式嵌入文档，尽量不要换这两个组件。**

---

### 5. LLM 三层配置：System → Workspace → Agent

**定义**：AnythingLLM 的 LLM 配置分为三层，优先级从高到低为 Agent > Workspace > System。

| 层级 | 作用 | 何时生效 |
|------|------|---------|
| **System LLM** | 全局默认的 LLM | 未配置 Workspace/Agent LLM 时使用 |
| **Workspace LLM** | 覆盖特定 Workspace 的 LLM | 仅在该 Workspace 内的对话中生效 |
| **Agent LLM** | 专用于 Agent 任务的 LLM | 仅在 `@agent` 会话中生效 |

**为什么要分三层**：

不同场景对 LLM 的需求不同：
- **日常对话**（System）：用通用且成本适中的模型（如 GPT-4o-mini）
- **法律文档 Workspace**：用推理能力更强的模型（如 Claude Sonnet）
- **Agent 任务**：用支持工具调用（Function Calling）的模型（如 GPT-4o）——不是所有模型都支持 Agent

**示例配置**：

```
System LLM: OpenAI GPT-4o-mini （便宜、日常够用）
├── Workspace "技术文档" → 继承 System LLM
├── Workspace "法律合规" → 覆盖为 Anthropic Claude Sonnet
├── Workspace "头脑风暴" → 覆盖为 Google Gemini
└── Agent LLM → OpenAI GPT-4o （支持 Function Calling）
```

---

### 6. Provider 选型逻辑

**定义**：三大组件各自有多个 Provider 可选，选型的核心维度是：隐私、成本、质量。

#### LLM Provider 全景

| 类别 | Provider | 适合场景 |
|------|----------|---------|
| **本地** | Built-in (Desktop 专属) | 最简单，一键下载模型 |
| **本地** | Ollama | 最流行的本地 LLM 方案，模型丰富 |
| **本地** | LM Studio, LocalAI, KobaldCPP | 更多本地选择 |
| **云端** | OpenAI, Anthropic, Google Gemini | 质量最高，但数据经过第三方 |
| **云端** | Groq, Together AI | 推理速度快、成本低 |
| **云端** | OpenRouter | 聚合器，一个 API Key 访问多家模型 |
| **云端** | Azure OpenAI, AWS Bedrock | 企业合规场景 |
| **云端** | OpenAI (Generic) | 兼容 OpenAI API 格式的任意服务 |

#### Embedding Provider 全景

| 类别 | Provider | 特点 |
|------|----------|------|
| **本地** | Built-in (默认) | 零配置、CPU 运行、不依赖外部服务 |
| **本地** | Ollama, LM Studio, LocalAI | 可用更强的 Embedding 模型 |
| **云端** | OpenAI, Azure OpenAI, Cohere | 质量高、速度快、需 API Key |

#### Vector Database Provider 全景

| 类别 | Provider | 特点 |
|------|----------|------|
| **本地** | LanceDB (默认) | 零配置、内置、性能足够 |
| **本地** | Chroma, Milvus | 更强的查询能力和扩展性 |
| **云端** | Pinecone, Zilliz, QDrant, Weaviate, AstraDB | 托管服务、自动扩展、多副本 |

**选型决策矩阵**：

| 你的优先级 | LLM 推荐 | Embedder 推荐 | VectorDB 推荐 |
|-----------|----------|--------------|--------------|
| **隐私优先** | Ollama（本地） | Built-in（本地） | LanceDB（本地） |
| **质量优先** | OpenAI / Anthropic | OpenAI | 任意（影响不大） |
| **成本优先** | Groq / 本地 Ollama | Built-in | LanceDB |
| **企业部署** | Azure OpenAI / Bedrock | Azure OpenAI | Pinecone / QDrant |
| **快速上手** | Built-in (Desktop) | Built-in | LanceDB |

> **推荐新手起步配置**：Desktop 安装 → Built-in LLM → Built-in Embedder → LanceDB。全部零配置，不需要任何 API Key，纯本地运行。等熟悉后再按需替换组件。

---

## 重点标记

1. **Desktop 是单人版，Docker 是团队版**：不需要多用户就用 Desktop，更简单
2. **AnythingLLM 本身极轻量**：2GB RAM + 2 核 CPU 即可运行，资源消耗取决于你选的组件
3. **LLM 可以三层覆盖**：System（全局默认）→ Workspace（按需覆盖）→ Agent（专用），灵活但不混乱
4. **Embedder 和 VectorDB 是系统级的**：一旦选定并开始嵌入文档，不建议更换，否则需要重新嵌入所有文档
5. **OpenAI Generic 是万能适配器**：任何兼容 OpenAI API 格式的服务都可以通过它接入

---

## 自测：你真的理解了吗？

**Q1**：你的团队有 10 个人，需要共享一个 AI 知识库，每个部门看到的文档不同，还需要在公司官网嵌入一个客服聊天窗口。应该选 Desktop 还是 Docker？为什么？

**Q2**：你已经用 OpenAI Embedding 模型嵌入了 500 个文档到 LanceDB。现在想换成 Ollama 的本地 Embedding 模型来降低成本。直接在设置里切换就行了吗？会有什么后果？

**Q3**：一个 Workspace 配置了 Workspace LLM 为 Claude Sonnet，System LLM 是 GPT-4o-mini，Agent LLM 是 GPT-4o。在这个 Workspace 中发起一个 `@agent` 会话时，用的是哪个 LLM？

**Q4**：你的笔记本没有 GPU，但你想用本地 LLM 保护隐私。有可能吗？怎么做？

**Q5**：有人说"选 Provider 最重要的是选最贵最好的"。你怎么看？在什么情况下这个说法是错的？
