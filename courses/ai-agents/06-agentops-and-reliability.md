# 模块六：AgentOps 与可靠性

> 对应 PDF 第 50-58 页

---

## 概念讲解

### 核心问题：为什么 Agent 的质量保证特别难？

传统软件：输入 A → 总是输出 B（确定性的）
AI Agent：输入 A → 可能输出 B、C 或 D（非确定性的）

这意味着：
- 传统单元测试**不够用**
- 你不仅要测"答案对不对"，还要测"推理过程对不对"
- 即使是最好的模型也可能幻觉或陷入推理循环

> LangChain CEO Harrison Chase："Agent 是生产力的关键，但成败取决于我们的指导。"

---

### 1. AgentOps 是什么？

**定义**：一套运维方法论，把 DevOps / MLOps / DataOps 的原则**适配**到 AI Agent 的独特挑战上。

**覆盖三个关键领域**：

| 领域 | 关注点 |
|------|--------|
| 正确性与可靠性 | 最终输出准确吗？推理过程合理吗？ |
| 性能与可扩展性 | 延迟、吞吐量够不够？ |
| 安全与负责任 | 有安全防护吗？行为在边界内吗？ |

---

### 2. 四层评估框架（核心内容）

这是本模块最重要的模型——从底层到顶层的系统化评估：

```
Layer 4: System-level Monitoring（生产监控）
         ↑
Layer 3: Outcome Evaluation（语义正确性）
         ↑
Layer 2: Trajectory Evaluation（推理正确性）★ 最关键
         ↑
Layer 1: Component-level Evaluation（组件单元测试）
```

#### Layer 1：组件级评估（确定性单元测试）

**目标**：验证非 LLM 组件的代码正确性。

**测什么**：
- **工具**：合法/非法/边界输入的行为
- **数据处理**：解析和序列化函数的健壮性
- **API 集成**：成功、失败、超时的处理

**怎么做**：
- ADK 中工具就是 Python 函数 → 直接写 pytest
- Agent Starter Pack 自动生成 `tests/unit/` 目录和 pytest 环境
- 运行：`make test`

#### Layer 2：轨迹评估（过程正确性）— 最关键的一层

**目标**：验证 Agent 的**推理过程**在 ReAct 循环中每一步都是正确的。

**测什么**：

| ReAct 阶段 | 验证内容 |
|-----------|---------|
| **Reason** | Agent 是否正确评估了目标和当前状态？假设合理吗？ |
| **Act** | 是否选了对的工具（Tool Selection）？参数是否正确（Parameter Generation）？ |
| **Observe** | 是否正确理解了工具输出，并正确地传递到下一轮 Reason？ |

**怎么做**：
- ADK 运行时与 Cloud Trace 集成，记录每个 Reason/Act/Observe 步骤
- Agent Starter Pack 在 `tests/integration/` 建"黄金测试集"
- CI/CD 管道在每次 PR 时自动运行轨迹评估，防止回归

#### Layer 3：结果评估（语义正确性）

**目标**：评估 Agent 最终给用户的回答是否正确。

**测什么**：
- **事实准确性**：答案是否基于 Observe 步骤获取的信息？
- **有用性和语气**：是否完整回答了用户需求？风格合适吗？
- **完整性**：是否包含了所有必要信息？

**怎么做**：
- ADK 工具或 API 做"grounding verification"（事实核查）
- Agent Starter Pack 集成 Vertex AI Gen AI evaluation service，支持 LLM-as-judge 评分
- 内置 UI playground 支持人工反馈，评分数据写入 BigQuery

#### Layer 4：系统级监控（生产环境）

**目标**：上线后持续跟踪 Agent 的真实表现。

**监什么**：
- 工具调用失败率
- 用户反馈评分
- 轨迹指标（如每个任务的 ReAct 循环次数）
- 端到端延迟
- CPU 和内存使用（**不要只看应用指标**）

**怎么做**：
- ADK Agent 在生产中持续发射事件和追踪数据
- Agent Starter Pack 自动配置 OpenTelemetry、Log Router → BigQuery、Looker Studio 仪表盘

---

### 3. AgentOps 工具链：ADK + Agent Starter Pack

#### Agent Starter Pack 的组成

| 组件 | 说明 |
|------|------|
| **Infrastructure as Code (Terraform)** | 可重现的模板，配置 Cloud Run、IAM、网络 |
| **CI/CD (Cloud Build)** | 自动化构建 → 单元测试 → 定量评估 → 部署 |
| **可观测性 (Cloud Trace + Logging)** | Agent 执行追踪 + 集中日志管理 |
| **数据集成 (BigQuery)** | 分析结构化企业数据 |
| **持续评估 (Vertex AI evaluation)** | 对每次代码变更运行评估数据集 |

**一条命令启动新项目**：
```bash
uvx agent-starter-pack create my-agent -a adk@gemini-fullstack
```

#### 五步工作流（开发到部署）

```
1. Bootstrap → Agent Starter Pack 生成项目（Terraform、CI/CD、评估骨架）
2. Develop → 用 ADK 写 Agent 逻辑（工具、编排、指令）
3. Commit → 代码提交触发 CI/CD 管道
4. Evaluate → 管道自动运行定量评估（验证性能和安全）
5. Deploy → 评估通过后自动部署到生产环境
```

---

### 4. 负责任 AI 与安全

**核心原则**：安全是"不可谈判的"（non-negotiable）。

#### 风险类型和防护手段

| 风险 | 防护 |
|------|------|
| 性能不达标（安全、质量、准确性） | Safety attributes、Recitation checks、客户反馈 |
| 被滥用 | 服务条款、可接受使用政策、内容审核 |
| 创造虚假能力印象 | UI 免责声明、模型卡 |
| 放大负面偏见 | 模型评估、偏见评估工具 |
| 加剧不平等 | 负责任 AI 指南、模型监控 |
| 不安全部署 | 模型评估、偏见评估 |
| 信息风险（幻觉、确认偏见等） | Recitation checks、偏见评估 |

#### ADK + Agent Starter Pack 的纵深防御

| 安全层面 | 怎么做 |
|---------|--------|
| **基础设施安全** | Terraform 配置 Cloud Run + IAM 最小权限 |
| **输入/输出防护** | ADK 中实现 prompt 注入检测 + 输出过滤 |
| **自动化安全测试** | CI/CD 管道在每次变更时运行安全测试 |
| **审计和监控** | ADK 追踪每个 thought 和 tool call，Agent Starter Pack 路由到 BigQuery |

---

### 5. Section 3 核心要点速查

| 你的目标 | 最佳方案 |
|---------|---------|
| 专业管理 Agent 生命周期 | 采用 AgentOps 方法论 |
| 上线前确保准确和安全 | CI/CD 管道中自动评估质量、grounding、安全 |
| 跟踪生产环境表现 | 用可观测性工具监控延迟、token 用量、工具成功率 |
| 排查 Agent 决策原因 | 用 logging/tracing 检查 Agent 的推理轨迹 |
| 保护 Agent、数据、工具访问 | 应用 AgentOps 安全原则：基础设施安全 + 数据治理 + 合规 |
| 快速启动 AgentOps | 用 Agent Starter Pack 的预配置模板 |

---

## 问答记录

> 待补充

---

## 重点标记

1. **四层评估是核心框架**：组件 → 轨迹 → 结果 → 系统监控
2. **Layer 2（轨迹评估）最关键**：验证 Agent "怎么想的"比"答了什么"更重要
3. **Agent Starter Pack 一条命令起步**：不要从零搭基础设施
4. **"Vibe testing" 不够**：必须用自动化、定量的评估替代感觉式测试
5. **安全是不可谈判的**：从第一天就设计安全防护，不是事后补
6. **持续评估**：每次 PR 都触发评估，防止质量回归
