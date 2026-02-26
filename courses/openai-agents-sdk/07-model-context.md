# 模块七：Model 与 Context 管理

> 对应 PDF 第 159-167 页（Chapter 7: Model and Context Management）

---

## 概念讲解

### 1. 模型管理（Model Management）

**定义**：OpenAI Agents SDK 的一个核心设计特点是**模型无关（Model-agnostic）**。每个 Agent 可以使用不同的底层模型——不同的提供商、不同的配置参数——这让你能按需搭配，而不是被锁定在一个模型上。

**核心思想**：就像一支球队里，不同位置需要不同特长的球员。Triage agent 做简单路由，用便宜快速的小模型就够了；研究型 Agent 需要深度推理，那就上重量级模型。这种"对症下药"的模型策略能在**成本、延迟、准确度**之间取得最佳平衡。

**为什么重要/有效**：想象一个多 Agent 系统——triage agent 用 GPT-4o 做路由其实太奢侈了，用 GPT-4.1-mini 甚至开源的 LLaMA 就行；但你的数学推理 Agent 可能需要 o3-pro 才能答对。OpenAI Agents SDK 支持这种**每个 Agent 用不同模型**的架构。

#### 1.1 调整底层模型

通过 `model` 参数在创建 Agent 时指定模型：

```python
from agents import Agent, Runner

agent = Agent(
    name="SampleAgent",
    instructions="You are an AI agent",
    model="gpt-4o"  # 指定模型
)
```

**实际对比示例**：

```python
from agents import Agent, Runner
import time

gpt4o_agent = Agent(name="GPT4o Agent", instructions="You are an AI Agent", model="gpt-4o")
o3pro_agent = Agent(name="o3-pro Agent", instructions="You are an AI Agent", model="o3-pro")

prompt = "How many integers from 1 to 10000 are divisible by 3 or by 5 but not by both?"

# GPT-4o: 1.18 秒，但答案错了 (3334)
# o3-pro: 8.41 秒，答案对了 (4001)
```

| 模型 | 速度 | 准确度 | 成本 |
|------|------|--------|------|
| **GPT-4o** | ~1.2 秒 | 复杂推理可能出错 | 低 |
| **o3-pro** | ~8.4 秒 | 复杂推理准确 | 高（约 10x） |

> **关键取舍**：准确度 vs 延迟/成本。简单任务没必要上最贵的模型，复杂推理别省这个钱。给每个 Agent 选对模型，才是真正的成本优化。

---

### 2. ModelSettings：精细调控模型行为

**定义**：除了选择用哪个模型，你还可以通过 `ModelSettings` 调整模型生成文本的方式——温度、最大 token 数等参数。同一个模型，不同设置可以产生截然不同的输出风格。

**核心思想**：不换模型也能改变 Agent 的"性格"。把 temperature 调高，Agent 就变成天马行空的创意型；调低就变成严谨保守型。

**常用设置**：

| 参数 | 作用 | 典型值 |
|------|------|--------|
| `temperature` | 控制输出随机性。低 = 确定性，高 = 创意 | 0.0 - 1.0 |
| `max_tokens` | 限制回复最大 token 数，控制篇幅 | 50 - 4096 |
| `top_p` | 核采样，和 temperature 类似但算法不同 | 0.0 - 1.0 |
| `tool_choice` | 控制模型是否/如何调用工具 | "auto", "required", "none" |
| `parallel_tool_calls` | 是否允许并行调用多个工具 | True / False |

> 完整参数列表参见：[ModelSettings 参考文档](https://openai.github.io/openai-agents-python/ref/model_settings/#agents.model_settings.ModelSettings)

**示例**——创意型 vs 精确型 Agent：

```python
from agents import Agent, Runner
from agents.model_settings import ModelSettings

creative_agent = Agent(
    name="CreativeAgent",
    instructions="You are an AI agent that answers questions.",
    model="gpt-4o",
    model_settings=ModelSettings(
        temperature=1.0,    # 高随机性
        max_tokens=300      # 允许长回复
    )
)

precise_agent = Agent(
    name="PreciseAgent",
    instructions="You are an AI agent that answers questions.",
    model="gpt-4o",
    model_settings=ModelSettings(
        temperature=0.2,    # 低随机性
        max_tokens=50       # 强制短回复
    )
)

prompt = "Describe the future of AI in customer service."
```

**效果对比**：
- **Creative Agent**：长篇大论、措辞丰富、有时甚至带有推测性语言
- **Precise Agent**：简短精炼、措辞保守，甚至因为 `max_tokens=50` 被截断

> **实践建议**：客服 Agent 建议 temperature 设低（0.2-0.3），确保回复稳定可靠；头脑风暴 Agent 可以设高（0.8-1.0），鼓励多样化输出。`max_tokens` 不宜设太低，否则回复会被截断导致用户体验差。

---

### 3. 第三方模型集成

**定义**：OpenAI Agents SDK 不限于 OpenAI 自己的模型。通过 LiteLLM（一个轻量级 Python 库），你可以无缝切换到 Anthropic Claude、Google Gemini、Meta LLaMA 等任何主流模型提供商。

**核心思想**：LiteLLM 提供了一层统一的 API 抽象——无论底层用什么模型，你的 Agent 代码不需要改。只需改 `model` 参数的字符串就行。

**安装**：

```bash
pip install "openai-agents[litellm]"
```

**配置 API Key**（以 Anthropic 为例）：

在 `.env` 文件中添加：
```
ANTHROPIC_API_KEY=sk-ant-api03-[your-key]
```

**使用示例**：

```python
from agents import Agent, Runner

agent = Agent(
    name="Claude Agent",
    instructions="You are an AI Agent",
    model="litellm/anthropic/claude-opus-4-20250514"  # 用 Anthropic Claude
)

response = Runner.run_sync(agent, "How do I restart my computer? Answer in a few words.")
print(response.final_output)
```

**常用 LiteLLM 模型字符串**：

| 提供商 | 模型字符串示例 |
|--------|---------------|
| Anthropic Claude | `litellm/anthropic/claude-opus-4-20250514` |
| Google Gemini | `litellm/gemini/gemini-pro` |
| Meta LLaMA | `litellm/meta_llama/Llama-3.3-70B-Instruct` |

**适用场景**：
- **Benchmark 对比**：同一个 Agent，分别用 GPT-4o 和 Claude 跑，看哪个效果更好
- **分环境部署**：开发用 GPT-4o，生产用 Claude（比如它在某些总结任务上更强）
- **成本优化**：非关键 Agent 用开源模型（LLaMA），省钱
- **合规要求**：某些行业要求数据不出境，必须用本地部署的模型

> **核心价值**：模型提供商是可替换的，你的 Agent 逻辑不需要跟着改。这是架构解耦的好处。

---

### 4. Context 管理：RunContext 与依赖注入

**定义**：Context（上下文）指 Agent 运行时能够访问到的所有信息。之前我们讨论过 system prompt、对话历史、RAG 检索等方式给 Agent 喂信息。这里要讲的是 **Local Context（本地上下文）**，也叫 **Run Context**——一种在 Agent 初始化时注入的、对 LLM 不可见但对 Tool 可见的数据。

**核心思想**：有些数据你希望 Tool 能用到（比如当前用户的 ID、订单号），但又不希望这些数据出现在发给 LLM 的 prompt 里（安全性考虑、隐私保护）。Local Context 就是为此设计的。

**为什么重要/有效**：这是一种**依赖注入（Dependency Injection）**模式。Agent 的 Tool 不需要自己去查"当前用户是谁"——这个信息在 Agent 启动时就被注入了。Tool 只管做自己的事，用户身份信息作为上下文自动传进来。

#### 4.1 基本用法

三步走：定义 Context 对象 → Agent 声明 Context 类型 → Runner 传入 Context 实例。

```python
from dataclasses import dataclass
from agents import Agent, Runner, RunContextWrapper, function_tool

# 第一步：定义 Context 数据结构
@dataclass
class OrderContext:
    customer_name: str
    order_id: str
    shipping_status: str

# 创建 Context 实例
order_context = OrderContext(
    customer_name="Henry Habib",
    order_id="123",
    shipping_status="Delayed"
)

# 第二步：Tool 函数通过 RunContextWrapper 接收 Context
@function_tool
def get_shipping_status(wrapper: RunContextWrapper[OrderContext]) -> str:
    """Provide the shipping status for the current order."""
    ctx = wrapper.context
    return (
        f"Hi {ctx.customer_name}, your order {ctx.order_id} is currently: "
        f"{ctx.shipping_status}."
    )

# 第三步：Agent 声明 Context 类型 + Runner 传入 Context
agent = Agent[OrderContext](
    name="Shipping Support Agent",
    instructions="You are a helpful support agent who can check shipping status.",
    tools=[get_shipping_status]
)

result = Runner.run_sync(agent, input="Where is my order?", context=order_context)
print(result.final_output)
# 输出：Hi Henry Habib, your order (123) is currently delayed.
```

**关键细节**：

| 要素 | 说明 |
|------|------|
| `@dataclass` 定义 Context | 普通 Python dataclass 即可，定义你需要传递的字段 |
| `Agent[OrderContext]` | 用泛型语法声明这个 Agent 需要什么类型的 Context |
| `RunContextWrapper[OrderContext]` | Tool 函数的参数类型，SDK 自动注入 |
| `context=order_context` | 在 `Runner.run_sync()` 中传入 Context 实例 |

**数据流向**：

```
用户提问 "Where is my order?"
    ↓
Agent 决定调用 get_shipping_status Tool
    ↓
SDK 自动把 order_context 注入到 Tool 的 wrapper 参数
    ↓
Tool 用 ctx.customer_name、ctx.order_id 组装回答
    ↓
Agent 返回给用户
```

> **核心优势**：用户的个人信息（姓名、订单号）**从未出现在 LLM 的 prompt 中**。LLM 只知道"有个 Tool 可以查订单状态"，不知道具体的用户数据。这对数据隐私至关重要。

#### 4.2 RunContext 的实际应用场景

| 场景 | Context 内容 |
|------|-------------|
| **客服系统** | 用户 ID、历史订单、VIP 等级 |
| **内部助手** | 员工工号、部门、权限等级 |
| **金融 Agent** | 账户余额、持仓信息、风险偏好 |
| **医疗 Agent** | 患者 ID、过敏信息、用药历史 |

> **设计原则**：Context 中放的是**会话级别的元数据**——跟当前用户/会话绑定，但不应该放到 prompt 里让 LLM 看到的数据。如果某个信息需要 LLM 推理时用到，那就放 system prompt 或 instructions 里。如果只是 Tool 执行时需要，就放 Context 里。

---

### 5. RunConfig 全局配置

**定义**：除了 per-agent 的 model 和 model_settings，SDK 还提供了 `RunConfig` 用于设置 Runner 级别的全局配置，比如 tracing 是否开启、超时时间等。

> **简单理解**：`model` 和 `model_settings` 是给单个 Agent 配参数，`RunConfig` 是给整个 Runner（一次完整运行）配参数。

![Tracing with Models](images/sdk_p171_1.png)

> **图说**：Traces 模块中可以看到不同 Agent 使用的模型信息和调用链路，方便调试和成本追踪。

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **每个 Agent 可以用不同模型**：通过 `model` 参数指定，简单 Agent 用便宜模型，复杂任务用强模型
2. **准确度 vs 成本/延迟的取舍**：GPT-4o 快但可能错，o3-pro 准但慢且贵——按任务复杂度选模型
3. **ModelSettings 调节模型行为**：`temperature` 控制创意度，`max_tokens` 控制篇幅，不换模型也能改 Agent "性格"
4. **LiteLLM 实现模型无关**：一个 `model` 字符串就能切换到 Claude、Gemini、LLaMA，Agent 代码不用改
5. **Local Context（RunContext）是依赖注入模式**：Tool 需要但 LLM 不该看到的数据，通过 Context 注入
6. **`Agent[ContextType]` + `RunContextWrapper[ContextType]`**：泛型语法声明 Context 类型，SDK 自动注入
7. **Context 中的数据不进 prompt**：用户个人信息、身份 Token 等敏感数据通过 Context 传递，保护隐私
8. **RunConfig 管全局配置**：Runner 级别的设置，如 tracing 开关、超时等
