# 模块四上：构建工具 - ADK 与编排

> 对应 PDF 第 29-35 页

---

## 概念讲解

### 1. ADK（Agent Development Kit）的定位

创业公司构建 Agent 面临一个核心**权衡**：

```
开发速度 ←──────────────────→ 灵活性
  ↑                              ↑
低代码平台                    从零开始构建
（快但控制力低）              （慢但完全自定义）

              ↑
            ADK
        （居中平衡点）
```

**ADK 的四大核心价值**：

1. **构建复杂协作系统**：多 Agent 协同，灵活编排
2. **集成现有工具**：兼容 LangChain、CrewAI、Notion、Slack 等
3. **从第一天保证质量**：内置测试、调试、评估工具
4. **自信地扩展**：FastAPI 暴露为标准 Web 服务，可容器化部署

> Google CEO Sundar Pichai："开源 ADK 在不到四个月内已有超过一百万次下载。"

---

### 2. ADK Agent 类型体系

ADK 的 Agent 分三大类：

```
BaseAgent
├── LlmAgent         ← 用 LLM 推理（非确定性）
├── Workflow Agents   ← 用预定义逻辑（确定性）
│   ├── SequentialAgent
│   ├── ParallelAgent
│   └── LoopAgent
└── CustomAgent       ← 完全自定义 Python 逻辑
```

#### A. LlmAgent — 最常用的 Agent 类型

| 属性 | 值 |
|------|---|
| 核心引擎 | LLM（如 Gemini） |
| 确定性 | 非确定性（灵活） |
| 用途 | 复杂推理、动态决策、自然语言理解 |

这就是最常见的"AI Agent"——用 LLM 来思考和决策。

#### B. Workflow Agents — 确定性编排

这三种 Agent 用**预定义逻辑**控制其他 Agent 的执行方式：

##### SequentialAgent（顺序执行）

```
Input → Agent A → Agent B → Output
```

**用例**：先获取网页内容，再总结。必须先拿到内容才能总结，所以必须顺序执行。

##### ParallelAgent（并行执行）

```
        ┌→ Agent A → Output 1
Input ──┼→ Agent B → Output 2
        └→ Agent C → Output 3
```

**用例**：多源数据采集、互不依赖的计算任务。**前提**：各 Agent 之间不需要共享状态。

##### LoopAgent（循环执行）

```
        ┌───────────────────────┐
        ↓                       │
Input → Agent A → Agent B → 检查退出条件？
                               │ 否 → 继续循环
                               │ 是 → Output
```

**用例**：生成图片直到内容正确（如生成包含5根香蕉的图片，用 Generate Image + Count Food Items 循环检查）。

#### C. CustomAgent — 完全自定义

继承 `BaseAgent`，实现 `_run_async_impl` 方法，写你自己的控制逻辑。适合标准推理循环无法满足的特殊需求。

---

### 3. ADK 如何实现 ReAct 循环

ADK 的 LlmAgent 原生实现了 ReAct 模式：

| ReAct 阶段 | ADK 实现 |
|-----------|---------|
| **Reason（推理）** | LlmAgent 内部管理：读取用户 prompt 和当前状态，调用 LLM 决定下一步 |
| **Act（行动）** | 通过工具系统：调用 Python 函数，或用 Agent-as-a-Tool 模式委托给子 Agent |
| **Observe（观察）** | 自动捕获工具返回的字典，注入到下一轮 Reason 的上下文中 |

---

### 4. ADK 工具体系

#### 设计有效工具的关键——"API 契约"

模型要正确使用工具，工具的定义必须像一份清晰的 API 文档：

| 要素 | 说明 | 重要性 |
|------|------|--------|
| **函数签名** | 描述性的名字和参数，Python 类型注解**必须**写 | 模型依赖类型注解生成调用参数 |
| **Docstring** | 精确定义工具的用途、使用条件、参数含义、返回格式 | 模型用 docstring 来**理解**工具 |
| **返回字典** | 建议包含 `status` 字段（success / error） | 让 Agent 在 Observe 阶段判断成功还是失败 |
| **ToolContext** | 可选参数，访问 session 级别的状态字典 | 需要读写持久状态时使用 |

#### 工具分类（Taxonomy）

```
ADK Tools
├── Toolsets（工具集）
│   └── 打包相关工具为一个对象（如 BigQueryToolset, MCPToolset）
│
├── Custom Function Tools（自定义函数工具）
│   ├── FunctionTool：同步 Python 函数的包装器
│   └── LongRunningFunctionTool：异步任务或 Human-in-the-Loop
│
├── Hierarchical & Remote Tools（层级和远程工具）
│   ├── Agent-as-a-Tool：父 Agent 把子 Agent 当工具调用
│   └── RemoteA2aAgent：通过 A2A 协议与远程 Agent 通信
│
└── Pre-built & Integrated Tools（预建工具）
    ├── Built-in：Google Search、Code Execution
    ├── Google Cloud Toolsets：Vertex AI Search、BigQuery
    └── 第三方兼容：LangchainTool、CrewaiTool 包装器
```

**Agent-as-a-Tool vs Sub-agent 的区别**：

| 模式 | 控制权 |
|------|--------|
| Agent-as-a-Tool | 父 Agent 调用子 Agent，拿到结果后**自己继续处理** |
| Sub-agent delegation | 父 Agent 把控制权**完全交给**子 Agent |

---

## 问答记录

> 待补充

---

## 重点标记

1. **ADK 居于"低代码"和"从零构建"之间**：兼顾开发速度和灵活性
2. **三种 Agent 类型要记住**：LlmAgent（推理）、Workflow Agents（编排）、Custom（自定义）
3. **Workflow Agent 是确定性的**：Sequential/Parallel/Loop 执行路径是预定义的
4. **工具的 Docstring 是"语义核心"**：模型靠 docstring 理解工具用途，写不好会导致工具调用错误
5. **工具名不要有歧义**：多个工具时，名称和描述要"简洁且独特"，否则模型会困惑
6. **Agent-as-a-Tool 是多 Agent 协作的关键模式**：父 Agent 保留控制权
