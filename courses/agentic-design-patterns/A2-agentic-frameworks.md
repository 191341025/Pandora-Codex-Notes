# 附录模块 A2：Agentic 框架概览

> 对应 PDF Appendix C（第 385-392 页）

---

## 概念地图

- **核心概念**（必须内化）：框架的抽象层级光谱（低级构建块 → 高级编排平台）、选型的核心权衡（控制粒度 vs 开发效率）
- **实操要点**（动手时需要）：LangChain / LangGraph / Google ADK / CrewAI 四大框架的定位与核心 API、框架选型决策指南
- **背景知识**（扩展理解）：AutoGen / LlamaIndex / Haystack / MetaGPT / SuperAGI / Semantic Kernel / Strands Agents 的定位与适用场景

---

## 概念讲解

### 1. 框架全景：从"砖块"到"预制房"

在正式介绍各个框架之前，先建立一个整体心智模型。Agentic 框架不是一堆并列的选项，而是**沿着抽象层级排列的一条光谱**：

```
低级（精细控制）                                      高级（快速搭建）
    ↓                                                      ↓
LangChain → LangGraph → Google ADK / CrewAI → MetaGPT / SuperAGI
  链式管道     有状态图       团队编排          领域专用平台
```

**直觉建立**：

想象你要建一栋房子：

| 层级 | 建筑类比 | 框架对应 |
|------|----------|----------|
| **砖块和水泥** | 你自己砌墙、接水管、布电线——完全控制每个细节 | LangChain（LCEL 管道） |
| **预制模块** | 用预制的墙板、管道模块组装——你设计布局，模块帮你省掉底层工作 | LangGraph（有状态图） |
| **精装交付** | 买精装房——户型、装修风格已定好，你选套餐就行 | Google ADK / CrewAI（团队编排） |
| **一站式物业** | 拎包入住的酒店式公寓——连物业管理都包了 | SuperAGI（全生命周期管理） |

> **类比边界**：与建房不同，框架之间不是完全递进的替代关系——LangGraph 建在 LangChain 之上，但 Google ADK 和 CrewAI 是独立的框架，不依赖 LangChain。选择哪个取决于你的具体需求，不是"越高级越好"。

**核心权衡**：

| 你想要... | 选低级框架 | 选高级框架 |
|-----------|-----------|-----------|
| **控制粒度** | 逐节点定义逻辑、显式管理状态 | 框架自动处理流程和状态 |
| **开发速度** | 慢——但完全按需定制 | 快——但受限于框架预设的模式 |
| **调试难度** | 透明——每一步都可观测 | 黑盒——需要了解框架内部行为 |
| **适用复杂度** | 高——可以表达任意复杂的逻辑 | 中——适合框架已支持的模式 |

---

### 2. LangChain

**定义**：LangChain 是一个用于构建 LLM 应用的基础框架，核心能力是通过 LangChain Expression Language（LCEL）将组件"管道化"连接——上一步的输出直接流入下一步的输入，形成线性的 DAG（有向无环图）工作流。

**核心机制**：LCEL 的 `|` 管道运算符

```python
# LCEL 链式组合（概念示意）
chain = prompt | model | output_parser
```

一行代码就把"构造 prompt → 调用 LLM → 解析输出"三步串起来了。`|` 运算符的本质是函数组合——每个组件是一个 Runnable，输出类型与下一个组件的输入类型匹配。

**适用场景**：

| 场景 | 说明 |
|------|------|
| **简单 RAG** | 检索文档 → 构造 prompt → LLM 生成答案 |
| **文本摘要** | 用户输入文本 → 摘要 prompt → 返回摘要 |
| **结构化提取** | 非结构化文本 → 提取为 JSON 等结构化数据 |

> **回顾**：Module 01 中我们已经用 LangChain LCEL 实现了 Prompt Chaining 的完整链条（extraction_chain → analysis_chain），Module 02 中用 `RunnableParallel` 实现了并行化模式。

**局限**：LangChain 构建的是 DAG——有向无环图，意味着**不能有循环**。如果你的 Agent 需要"试一次 → 检查结果 → 不满意就再试"这样的循环推理，LangChain 原生做不到，这正是 LangGraph 出现的原因。

---

### 3. LangGraph

**定义**：LangGraph 是构建在 LangChain 之上的库，专门处理需要**有状态、可循环**的高级 Agent 系统。它将工作流定义为一个图（Graph），图中的节点（Node）是函数或 LCEL 链，边（Edge）是条件逻辑，支持循环，并且**显式管理应用状态**。

**解决什么问题**：

LangChain 只能走"A → B → C"的直线。但真实的 Agent 经常需要：
- 执行一步 → 检查结果 → 不满意就回去重试（循环）
- 根据中间结果决定下一步走哪条路（条件分支）
- 在多步执行中记住之前的所有结果（持久状态）

LangGraph 用图结构解决了这些问题——你可以画出任意拓扑的工作流，包括环。

**三大典型用例**：Multi-Agent 协作、Plan-and-Execute Agents（规划后执行）、Human-in-the-Loop 工作流。

#### LangChain vs LangGraph 对比

| 维度 | LangChain | LangGraph |
|------|-----------|-----------|
| **核心抽象** | Chain（LCEL 管道） | Graph of Nodes（节点图） |
| **工作流类型** | 线性 DAG（有向无环图） | 可循环图（支持环） |
| **状态管理** | 基本无状态（每次运行独立） | 显式的持久化状态对象 |
| **适用场景** | 简单、可预测的线性序列 | 复杂、动态、有状态的 Agent |

**选型建议**：
- **选 LangChain**：流程清晰、可预测，A → B → C 不需要回头
- **选 LangGraph**：Agent 需要推理、规划、循环重试，需要在多步间维护状态

**代码示例：并行工作流（笑话 + 故事 + 诗歌生成器）**

```python
# 定义图的状态结构
class State(TypedDict):
    topic: str
    joke: str
    story: str
    poem: str
    combined_output: str

# 定义节点——每个节点是一个函数
def call_llm_1(state: State):
    """生成笑话"""
    msg = llm.invoke(f"Write a joke about {state['topic']}")
    return {"joke": msg.content}

def call_llm_2(state: State):
    """生成故事"""
    msg = llm.invoke(f"Write a story about {state['topic']}")
    return {"story": msg.content}

def call_llm_3(state: State):
    """生成诗歌"""
    msg = llm.invoke(f"Write a poem about {state['topic']}")
    return {"poem": msg.content}

def aggregator(state: State):
    """聚合所有输出"""
    combined = f"Here's a story, joke, and poem about {state['topic']}!\n\n"
    combined += f"STORY:\n{state['story']}\n\n"
    combined += f"JOKE:\n{state['joke']}\n\n"
    combined += f"POEM:\n{state['poem']}"
    return {"combined_output": combined}

# 构建工作流图
parallel_builder = StateGraph(State)

# 添加节点
parallel_builder.add_node("call_llm_1", call_llm_1)
parallel_builder.add_node("call_llm_2", call_llm_2)
parallel_builder.add_node("call_llm_3", call_llm_3)
parallel_builder.add_node("aggregator", aggregator)

# 添加边——从 START 到三个节点（并行扇出），再汇聚到 aggregator
parallel_builder.add_edge(START, "call_llm_1")
parallel_builder.add_edge(START, "call_llm_2")
parallel_builder.add_edge(START, "call_llm_3")
parallel_builder.add_edge("call_llm_1", "aggregator")
parallel_builder.add_edge("call_llm_2", "aggregator")
parallel_builder.add_edge("call_llm_3", "aggregator")
parallel_builder.add_edge("aggregator", END)

# 编译并运行
parallel_workflow = parallel_builder.compile()
state = parallel_workflow.invoke({"topic": "cats"})
print(state["combined_output"])
```

> **代码解读**：这段代码展示了 LangGraph 的核心操作——定义 `State` 类型 → 写节点函数（每个函数读取 state、返回更新）→ 用 `add_node` / `add_edge` 构建图 → `compile()` + `invoke()`。从 `START` 同时出发到三个节点实现了并行扇出，三个节点都连向 `aggregator` 实现了扇入聚合。

---

### 4. Google ADK

**定义**：Google Agent Development Kit（ADK）是一个高层级、结构化的框架，专门用于构建和部署由多个 AI Agent 组成的应用。与 LangChain/LangGraph 提供底层构建块不同，ADK 提供**预制的架构模式**，让开发者以"团队"的视角编排 Agent 协作。

**核心差异——ADK vs LangGraph（工厂 vs 精密布线）**：

| 维度 | LangGraph | Google ADK |
|------|-----------|------------|
| **开发比喻** | 给你导线和元器件，你自己画电路图、焊接每个连接点 | 给你一条装配产线，机器人已经知道怎么协作 |
| **节点/边定义** | 开发者显式定义每个 node 和 edge | 框架提供 SequentialAgent / ParallelAgent 等预制类型 |
| **状态管理** | 显式——开发者定义 State 对象、手动在节点间传递 | 隐式——框架管理 Session 和 State |
| **适合场景** | 需要精细控制单个 Agent 内部推理流程 | 需要快速搭建多 Agent 协作的团队 |
| **灵活性** | 高——可以表达任意图结构 | 中——受限于框架提供的编排模式 |

**ADK 的预制 Agent 类型**：

| Agent 类型 | 功能 | 类比 |
|-----------|------|------|
| `LlmAgent` | 基础 Agent——接收指令、使用工具 | 团队中的个人贡献者 |
| `SequentialAgent` | 按顺序执行子 Agent | 流水线工位 |
| `ParallelAgent` | 并行执行子 Agent | 多条并行产线 |
| `LoopAgent` | 循环执行直到条件满足 | 质检循环 |

> **回顾**：Module 04 中我们已深入使用了 ADK 的三大编排原语（SequentialAgent / ParallelAgent / LoopAgent）来实现多 Agent 系统。Module 06 中我们用 ADK 的 MCPToolset 连接了 MCP Server。

**代码示例：ADK 搜索增强 Agent**

```python
from google.adk.agents import LlmAgent
from google.adk.tools import google_search

dice_agent = LlmAgent(
    model="gemini-2.0-flash-exp",
    name="question_answer_agent",
    description="A helpful assistant agent that can answer questions.",
    instruction="""Respond to the query using google search""",
    tools=[google_search],
)
```

> **注意简洁度**：同样是创建一个能搜索的 Agent，ADK 只需要 6 行声明式代码——指定模型、名称、描述、指令、工具。不需要定义图、状态、边。这就是"工厂 vs 精密布线"的差异——ADK 把底层编排逻辑封装在了 `LlmAgent` 内部。

---

### 5. CrewAI

**定义**：CrewAI 是一个以**角色扮演和团队协作**为核心的多 Agent 编排框架。它不要求你定义图结构或管道，而是让你设计一个"团队章程"——定义角色（Agent）、任务（Task）和团队（Crew），框架自动管理交互。

**三大核心组件**：

| 组件 | 是什么 | 定义方式 |
|------|--------|---------|
| **Agent** | 有"人设"的执行者——不只有功能，还有角色（role）、目标（goal）和背景故事（backstory） | 声明 role + goal + backstory |
| **Task** | 一个离散的工作单元——有描述、期望输出，分配给特定 Agent | 声明 description + expected_output + agent |
| **Crew** | 包含 Agent 和 Task 的协作单元——执行预定义的流程 | 组合 agents + tasks + process |

**两种流程模式**：

| 模式 | 说明 | 类比 |
|------|------|------|
| **Sequential（顺序）** | 上一个 Task 的输出流入下一个 Task | 接力赛——每棒选手跑完把棒传给下一位 |
| **Hierarchical（层级）** | 一个"经理" Agent 负责分配任务和协调工作流 | 项目经理分派工作给团队成员 |

**代码示例**：

```python
@crew
def crew(self) -> Crew:
    """Creates the research crew"""
    return Crew(
        agents=self.agents,
        tasks=self.tasks,
        process=Process.sequential,
        verbose=True,
    )
```

> **回顾**：Module 04 中我们用 CrewAI 实现了多 Agent 协作（定义 Researcher + Writer Agent），Module 13 中用 CrewAI 的 Policy Enforcer Agent 实现了安全护栏。

**CrewAI 的独特定位**：

与 LangGraph 的"画电路图"和 ADK 的"装配产线"不同，CrewAI 更像是"组建项目团队"——你不关心信息怎么流动、状态怎么传递，只关心"谁做什么、按什么顺序做"。这让它特别适合**模拟人类团队协作**的场景。

---

### 6. 其他框架速览

以下框架在 PDF 中有简要介绍，各有独特定位：

| 框架 | 开发方 | 核心定位 | 关键优势 | 主要局限 |
|------|--------|---------|----------|---------|
| **Microsoft AutoGen** | Microsoft | 基于**对话**的多 Agent 协作 | 灵活的对话驱动方式，支持动态交互 | 对话范式可能导致执行路径不可预测，需要精心的 prompt 设计来确保收敛 |
| **LlamaIndex** | LlamaIndex | **数据框架**——连接 LLM 与外部/私有数据 | 数据摄取和检索管道非常强大，是 RAG 场景的首选 | 复杂 Agent 控制流和多 Agent 编排能力较弱 |
| **Haystack** | deepset | 构建可扩展的**搜索系统** | 模块化节点管道，专注于大规模信息检索的性能和可扩展性 | 为搜索管道优化的设计，实现高度动态的 Agent 行为时较僵硬 |
| **MetaGPT** | — | 基于 **SOP** 的多 Agent 系统 | 用标准操作流程结构化 Agent 协作，模拟软件公司（产品经理、工程师等角色） | 高度专业化——不适合代码生成以外的通用 Agent 任务 |
| **SuperAGI** | — | Agent **全生命周期管理**平台 | 提供 Agent 创建、监控、GUI、故障处理（如循环检测），关注生产就绪性 | 全平台方案带来更多复杂度和开销 |
| **Semantic Kernel** | Microsoft | **SDK**——通过插件和规划器将 LLM 集成到传统代码中 | 与企业级代码库无缝集成（.NET / Python），将 LLM 视为推理引擎（Reasoning Engine） | 插件 + 规划器的概念架构学习曲线较陡 |
| **Strands Agents** | AWS | **轻量级 SDK**——模型驱动的 Agent 构建 | 简单灵活、模型无关、原生集成 MCP | 轻量设计意味着高级监控、生命周期管理等需要自建 |

> **Semantic Kernel 中的推理引擎概念**：原书 Appendix C 在介绍 Semantic Kernel 时提到了"Reasoning Engine"这个概念——它的核心思想是将 LLM **不当作**独立的 AI 系统，而是当作一个更大软件应用中的**推理组件**。LLM 通过 Semantic Kernel 的"插件"调用传统代码函数，通过"规划器"编排多步骤工作流。这种视角将 LLM 从"生成器"提升为"决策者"。

---

### 7. 框架选型决策指南

面对这么多框架，如何选择？关键在于**匹配你的应用需求到正确的抽象层级**：

```
你的应用需要什么？
    │
    ├─ 简单的线性流程（A→B→C）
    │   └─→ LangChain（LCEL 管道就够了）
    │
    ├─ 需要循环推理、条件分支、状态持久化
    │   └─→ LangGraph（有状态图提供精细控制）
    │
    ├─ 多 Agent 团队协作，快速搭建
    │   ├─ 偏好声明式 + Google 生态 → Google ADK
    │   └─ 偏好角色驱动 + 团队模拟 → CrewAI
    │
    ├─ 核心挑战是数据检索和知识增强
    │   └─→ LlamaIndex（数据管道专精）
    │
    ├─ 需要 Agent 全生命周期管理（创建、监控、运维）
    │   └─→ SuperAGI
    │
    └─ 需要将 LLM 嵌入现有企业应用
        └─→ Semantic Kernel（插件式集成）
```

**核心原则**（来自 PDF 结论）：

> 选择框架的关键不是"哪个最强"，而是"你的项目需要哪个层级的抽象"——需要简单序列、动态推理循环，还是受管理的专家团队？

**四大框架对比总结**：

| 维度 | LangChain | LangGraph | Google ADK | CrewAI |
|------|-----------|-----------|------------|--------|
| **抽象层级** | 低——管道组合 | 中——有状态图 | 高——预制编排 | 高——角色编排 |
| **工作流类型** | 线性 DAG | 可循环图 | 预制模式（Sequential/Parallel/Loop） | Sequential / Hierarchical |
| **状态管理** | 基本无 | 显式 State 对象 | 隐式 Session/State | 隐式（任务输出流） |
| **多 Agent** | 需自建 | 可从头构建 | 原生团队编排 | 原生角色协作 |
| **学习曲线** | 低 | 中 | 低-中 | 低 |
| **灵活性** | 中 | 高 | 中 | 中-低 |
| **本课程使用** | Module 01/02/03/05/07 | — | 贯穿全课程 | Module 04/13 |

---

## 模式关联

| 关系类型 | 相关模式 | 说明 |
|----------|---------|------|
| **实现载体** | 所有设计模式（Module 00-14）| 框架是设计模式的实现载体——Prompt Chaining 用 LangChain LCEL，Multi-Agent 用 ADK/CrewAI |
| **标准化层** | MCP（Module 06）| MCP 标准化了 Agent-工具通信，框架通过集成 MCP 获得跨平台工具互操作能力（如 ADK MCPToolset、Strands Agents 原生 MCP） |
| **协作层** | A2A（Module 10）| A2A 协议标准化了不同框架构建的 Agent 之间的通信——让 LangGraph Agent 和 ADK Agent 可以跨框架协作 |
| **互补** | Tool Use（Module 03）| 每个框架都有自己的工具定义方式（LangChain @tool、ADK 内置工具、CrewAI 工具），但核心的工具使用模式是通用的 |

---

## 重点标记

1. **框架是一条抽象光谱**：从 LangChain（砖块和水泥）到 Google ADK / CrewAI（精装交付），核心权衡是控制粒度 vs 开发效率
2. **LangChain = 线性管道，LangGraph = 有状态循环图**：需要循环推理就选 LangGraph，线性流程用 LangChain 足够
3. **ADK = 工厂装配线，LangGraph = 精密布线**：ADK 用预制 Agent 类型封装了底层图构建，换来开发速度但牺牲了灵活性
4. **CrewAI 的独特之处是"人设"**：Agent 有 role + goal + backstory，适合模拟人类团队协作的场景
5. **选框架看抽象层级**：不是"哪个最强"，而是"你的项目需要简单序列、动态推理循环，还是受管理的专家团队"
6. **MCP 和 A2A 是跨框架的连接器**：MCP 标准化 Agent-工具通信，A2A 标准化 Agent-Agent 通信，让不同框架构建的 Agent 可以互操作

---

## 自测：你真的理解了吗？

**Q1**：你正在构建一个客服 Agent，它需要：接收用户问题 → 检索知识库 → 生成回答。流程是固定的，不需要循环重试。你会选 LangChain 还是 LangGraph？为什么？如果后来产品经理要求增加"回答质量不够就自动重新生成"的功能，你的选择会变吗？

**Q2**：一个团队需要快速原型验证一个多 Agent 系统——三个 Agent 分别负责调研、写作和审校，按顺序执行。Google ADK 和 CrewAI 都能做。两者在这个场景下的核心差异是什么？你会根据什么因素做最终选择？

**Q3**：你的公司已有大量 .NET 代码和内部 API，现在想让 LLM 能调用这些内部 API 来辅助员工工作。上面哪个框架最适合这个场景？请说明理由，并解释"LLM 作为推理引擎"在这个场景中意味着什么。

**Q4**：一位同事说"LangGraph 比 LangChain 更强大，所以我们所有项目都应该用 LangGraph"。你同意吗？请从开发成本、调试复杂度和 YAGNI（You Ain't Gonna Need It）原则三个角度分析这个观点。

**Q5**：回顾本课程中使用框架的情况——LangChain 在 Module 01/02 中实现链式和并行模式，ADK 贯穿全课程做编排，CrewAI 在 Module 04/13 中做角色协作。如果你要从零开始构建一个 Agent 项目，你会选择"一个框架走天下"还是"不同场景用不同框架"？各有什么利弊？
