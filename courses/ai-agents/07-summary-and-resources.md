# 模块七：总结与资源

> 对应 PDF 第 59-64 页

---

## 白皮书完整知识图谱

```
                        AI Agents 技术指南
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
      Section 1          Section 2          Section 3
      核心概念            如何构建            可靠与负责
            │                 │                 │
    ┌───────┼───────┐    ┌───┼────┐       ┌────┼────┐
    │       │       │    │   │    │       │    │    │
  Agent   核心    Grounding ADK  设计    AgentOps 安全
  生态    组件    & RAG    工具链 & 规模   评估    & 负责
    │       │       │      │      │       │      │
    │    ┌──┴──┐    │   ┌──┴──┐   │    ┌──┴──┐   │
    │    │     │    │   │     │   │    │     │   │
  Build Model Tool  RAG ADK  MCP  Agent 4层   纵深
  Use   Orch  Mem  Graph     A2A Space 评估   防御
  Partner Runtime   Agentic  Deploy     框架
                    RAG      Engine
```

---

## 三大 Section 核心要点速查

### Section 1：核心概念

| 组件 | 一句话 |
|------|--------|
| Model | 按场景选模型，不是越强越好 |
| Tools | Agent 超越 LLM 原生能力的"手脚" |
| Orchestration | ReAct 循环：Reason → Act → Observe |
| Runtime | 三选一：Agent Engine / Cloud Run / GKE |
| Grounding | RAG → GraphRAG → Agentic RAG 三级演进 |

### Section 2：如何构建

| 工具 | 一句话 |
|------|--------|
| ADK | 代码优先的 Agent 开发框架，多 Agent 设计 |
| MCP | Agent 连接外部工具的通用标准 |
| A2A | Agent 之间通信协作的开放协议 |
| Agent Engine | ADK Agent 的推荐部署目标 |
| Agentspace | 管理 Agent 舰队 + 无代码构建 |
| Firebase Studio | 全栈应用开发平台 |
| Gemini CLI | 终端里免费实验 |

### Section 3：可靠与负责

| 实践 | 一句话 |
|------|--------|
| 四层评估 | 组件 → 轨迹 → 结果 → 系统监控 |
| Agent Starter Pack | 一条命令生成生产级基础设施 |
| 持续评估 | 每次 PR 自动评估，防止回归 |
| 纵深防御 | 基础设施安全 + IO 防护 + 自动测试 + 审计 |

---

## 关键术语中英对照表

| 英文 | 中文 | 模块 |
|------|------|------|
| AI Agent | AI 代理 | 全文 |
| ADK (Agent Development Kit) | Agent 开发工具包 | 2 |
| MCP (Model Context Protocol) | 模型上下文协议 | 2 |
| A2A (Agent2Agent) | Agent 间通信协议 | 2 |
| RAG (Retrieval-Augmented Generation) | 检索增强生成 | 1 |
| GraphRAG | 图谱增强 RAG | 1 |
| Agentic RAG | Agent 主动检索 RAG | 1 |
| Grounding | 事实落地/接地 | 1 |
| ReAct (Reason + Act) | 推理与行动 | 1 |
| Orchestration | 编排 | 1 |
| Fine-tuning | 微调 | 1 |
| Vector Embedding | 向量嵌入 | 1 |
| AgentOps | Agent 运维 | 3 |
| Trajectory Evaluation | 轨迹评估 | 3 |
| Memory Distillation | 记忆蒸馏 | 2 |
| Context Poisoning | 上下文中毒 | 2 |
| Agent Card | Agent 数字名片 | 2 |
| HITL (Human in the Loop) | 人在回路 | 3 |
| SAIF (Secure AI Framework) | 安全 AI 框架 | 3 |

---

## Google Cloud Agent 工具全景

| 工具/服务 | 用途 |
|----------|------|
| ADK | 构建 Agent |
| Vertex AI Agent Engine | 部署和管理 Agent |
| Agent Starter Pack | 快速搭建生产基础设施 |
| Google Agentspace | 管理 Agent 舰队 + 无代码构建 |
| Firebase Studio | 全栈应用开发 |
| Gemini CLI | 终端实验 |
| Vertex AI Search | 语义搜索（RAG） |
| Vertex AI RAG Engine | RAG 数据框架 |
| Memory Bank | Agent 长期记忆 |
| Example Store | few-shot 示例管理 |
| Model Garden | 200+ 模型市场 |
| Gemini Code Assist | 编程助手 |
| Gemini Cloud Assist | 云基础设施助手 |

---

## 延伸学习资源

白皮书推荐的关键资源：

| 资源 | 说明 |
|------|------|
| ADK Documentation | ADK 官方文档（代码示例和 API） |
| ADK Samples Repository | 现成的 Agent 示例库 |
| A2A Protocol Docs | A2A 协议规范 |
| Agent Starter Pack | 生产级模板 |
| Google's SAIF | 安全 AI 框架指南 |
| Startup School | Google Cloud 创业课程 |
| Google for Startups Cloud Program | 最高 $350k 云计算积分 |

---

## 学完之后的建议

1. **先理解概念再动手**：Section 1 的概念是基础，不要跳过
2. **ADK 是核心工具**：值得深入学习文档和示例
3. **MCP + A2A 是趋势**：标准化协议会成为行业标准
4. **AgentOps 决定产品质量**：不是"能跑就行"，要"可靠地跑"
5. **从简单开始**：先用 Gemini CLI 实验，再用 ADK 构建，最后用 Agent Starter Pack 部署

---

## 问答记录

> 待补充
