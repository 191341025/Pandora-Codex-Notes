# 模块二：Weather Bot 基础 — 单 Agent 与多模型

> 对应 PDF 第 15-28 页（Step 0 ~ Step 2）

---

## 概念讲解

### 1. 渐进式教程总览

这个教程从一个简单的 Weather Bot 出发，逐步叠加 6 层能力：

```
Step 0: 安装和配置
Step 1: 单 Agent + 单工具（基础 Weather Lookup）
Step 2: LiteLLM 多模型切换（Gemini / GPT / Claude）[可选]
Step 3: 多 Agent Team + 任务委派
Step 4: Session State 记忆
Step 5: before_model_callback 输入护栏
Step 6: before_tool_callback 工具参数护栏
```

**最终产物**：一个多 Agent Weather Bot 系统，能处理天气查询、打招呼、告别，还能记住上次查的城市，并有安全边界保护。

**运行环境**：推荐在 Google Colab / Jupyter Notebook 中运行（因为用了大量 `await` 异步代码）。如果在 `.py` 脚本中运行，需要用 `asyncio.run()`。

> **替代方案**：如果不想手动管理 Runner/Session，可以用 `adk web` / `adk run` / `adk api_server`，它们会自动处理。本教程手动管理是为了让你理解底层机制。

---

### 2. Step 0：安装与配置

#### 安装依赖

```bash
pip install google-adk -q
pip install litellm -q     # 多模型支持
```

#### 核心 import

```python
import os
import asyncio
from google.adk.agents import Agent
from google.adk.models.lite_llm import LiteLlm       # 多模型支持
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.genai import types                        # 消息格式
```

#### 模型常量定义

```python
MODEL_GEMINI_2_5_FLASH = "gemini-2.5-flash"
MODEL_GPT_4O = "openai/gpt-4.1"                             # LiteLLM 格式：provider/model
MODEL_CLAUDE_SONNET = "anthropic/claude-sonnet-4-20250514"   # 同上
```

> **注意**：Gemini 模型直接写字符串（如 `"gemini-2.5-flash"`），其他模型需要用 `LiteLlm(model="provider/model_name")` 包装。

#### API Key 配置

```python
os.environ["GOOGLE_API_KEY"] = "你的 Key"
os.environ['OPENAI_API_KEY'] = '你的 Key'        # 可选
os.environ['ANTHROPIC_API_KEY'] = '你的 Key'     # 可选
os.environ["GOOGLE_GENAI_USE_VERTEXAI"] = "False"
```

> **安全提示**：正式项目中不要硬编码 API Key，用 Colab Secrets 或环境变量管理。

---

### 3. Step 1：定义 Tool（工具）

**定义**：Tool 就是普通的 Python 函数，赋予 Agent "做事"的能力——调 API、查数据库、算数……不写 Tool，Agent 只会聊天。

```python
def get_weather(city: str) -> dict:
    """Retrieves the current weather report for a specified city.

    Args:
        city (str): The name of the city (e.g., "New York", "London", "Tokyo").

    Returns:
        dict: A dictionary containing the weather information.
              Includes a 'status' key ('success' or 'error').
    """
    print(f"--- Tool: get_weather called for city: {city} ---")
    city_normalized = city.lower().replace(" ", "")

    mock_weather_db = {
        "newyork": {"status": "success", "report": "The weather in New York is sunny with a temperature of 25°C."},
        "london":  {"status": "success", "report": "It's cloudy in London with a temperature of 15°C."},
        "tokyo":   {"status": "success", "report": "Tokyo is experiencing light rain and a temperature of 18°C."},
    }

    if city_normalized in mock_weather_db:
        return mock_weather_db[city_normalized]
    else:
        return {"status": "error", "error_message": f"Sorry, I don't have weather information for '{city}'."}
```

**关键点**：

- 函数签名有类型注解（`city: str`）→ LLM 知道该传什么参数
- Docstring 描述清楚"做什么、参数、返回值" → LLM 知道什么时候用
- 返回 dict 包含 `status` 字段 → 方便 Agent 判断成功/失败
- `print()` 日志 → 调试时能看到 Tool 是否被调用

> **最佳实践**：Docstring 写得越清晰，LLM 用工具就越准。这不是给人看的注释，是给 LLM 看的"使用说明书"。

---

### 4. Step 1：定义 Agent

```python
weather_agent = Agent(
    name="weather_agent_v1",
    model=MODEL_GEMINI_2_5_FLASH,
    description="Provides weather information for specific cities.",
    instruction="You are a helpful weather assistant. "
                "When the user asks for the weather in a specific city, "
                "use the 'get_weather' tool to find the information. "
                "If the tool returns an error, inform the user politely. "
                "If the tool is successful, present the weather report clearly.",
    tools=[get_weather],
)
```

**参数详解**：

| 参数 | 给谁看 | 作用 |
|------|--------|------|
| `name` | ADK 内部 | 唯一标识，用于路由和日志 |
| `model` | ADK | 指定用哪个 LLM |
| `description` | 其他 Agent | 后续做任务委派时，root agent 靠这个决定把任务分给谁 |
| `instruction` | LLM | Agent 的"人设"和行为指南，越详细越好 |
| `tools` | LLM + ADK | 可用工具列表，直接传函数引用 |

> **`description` vs `instruction` 的区别**：`description` 是一句话概括（给"上级"看），`instruction` 是详细的行为指南（给"自己"看）。

---

### 5. Step 1：Runner 与 Session 执行机制

这是 ADK 最核心的运行模型：**Agent + Runner + SessionService 三件套**。

#### SessionService（会话管理）

```python
session_service = InMemorySessionService()

APP_NAME = "weather_tutorial_app"
USER_ID = "user_1"
SESSION_ID = "session_001"

session = await session_service.create_session(
    app_name=APP_NAME,
    user_id=USER_ID,
    session_id=SESSION_ID
)
```

**作用**：管理对话历史和状态。`InMemorySessionService` 是最简单的实现（内存存储，重启就没了），适合开发测试。

**三个标识符**：
- `app_name`：应用名
- `user_id`：用户 ID
- `session_id`：会话 ID

> 这三个组合唯一确定一个对话上下文。

#### Runner（执行引擎）

```python
runner = Runner(
    agent=weather_agent,
    app_name=APP_NAME,
    session_service=session_service
)
```

**作用**：编排整个交互流程——接收用户输入 → 路由到 Agent → Agent 调 LLM → LLM 决定调 Tool → Tool 返回结果 → LLM 生成回复 → 更新 Session。

#### 交互函数

```python
async def call_agent_async(query: str, runner, user_id, session_id):
    """Sends a query to the agent and prints the final response."""
    print(f"\n>>> User Query: {query}")

    content = types.Content(role='user', parts=[types.Part(text=query)])
    final_response_text = "Agent did not produce a final response."

    async for event in runner.run_async(user_id=user_id, session_id=session_id, new_message=content):
        if event.is_final_response():
            if event.content and event.content.parts:
                final_response_text = event.content.parts[0].text
            elif event.actions and event.actions.escalate:
                final_response_text = f"Agent escalated: {event.error_message or 'No specific message.'}"
            break

    print(f"<<< Agent Response: {final_response_text}")
```

**执行流程拆解**：

```
用户输入 "What is the weather in London?"
    ↓
types.Content(role='user', parts=[Part(text=...)])   ← 打包成 ADK 消息格式
    ↓
runner.run_async(...)                                 ← 提交给 Runner
    ↓
Runner 把消息发给 Agent 的 LLM
    ↓
LLM 看到 instruction + tools + 用户消息，决定调用 get_weather("London")
    ↓
ADK 自动执行 get_weather("London")，拿到 {"status": "success", "report": "..."}
    ↓
LLM 拿到 Tool 结果，生成最终回复
    ↓
Event(is_final_response=True) 返回给用户
```

**Event（事件）是什么**：Runner 在执行过程中会 yield 多个 Event，代表每一步：
- Tool call requested（LLM 请求调用工具）
- Tool result received（工具返回结果）
- Intermediate LLM thought（中间推理）
- Final response（最终回复）→ 用 `event.is_final_response()` 识别

> **为什么是 async？** LLM 调用和工具执行都是 I/O 密集型操作，用 asyncio 可以高效处理而不阻塞。

---

### 6. Step 2：LiteLLM 多模型切换

**定义**：LiteLLM 是一个统一接口库，支持 100+ 种 LLM。ADK 通过 `LiteLlm` wrapper 集成它，让你一行代码换模型。

**为什么需要多模型**：

| 考量 | 说明 |
|------|------|
| 性能 | 某些模型在特定任务上更强（编码、推理、创意写作） |
| 成本 | 不同模型价格差异很大 |
| 能力 | 上下文窗口大小、微调选项不同 |
| 可用性 | 多模型 = 冗余备份 |

#### 用法：Gemini vs 非 Gemini

```python
# Gemini 模型：直接传字符串
agent_gemini = Agent(model="gemini-2.5-flash", ...)

# 非 Gemini 模型：用 LiteLlm 包装
agent_gpt = Agent(model=LiteLlm(model="openai/gpt-4.1"), ...)
agent_claude = Agent(model=LiteLlm(model="anthropic/claude-sonnet-4-20250514"), ...)
```

#### GPT Agent 示例

```python
weather_agent_gpt = Agent(
    name="weather_agent_gpt",
    model=LiteLlm(model=MODEL_GPT_4O),
    description="Provides weather information (using GPT-4o).",
    instruction="You are a helpful weather assistant powered by GPT-4o. "
                "Use the 'get_weather' tool for city weather requests. "
                "Clearly present successful reports or polite error messages.",
    tools=[get_weather],
)
```

> **注意**：每个 Agent 需要独立的 SessionService + Session + Runner，对话历史互不干扰。

#### Claude Agent 示例

```python
weather_agent_claude = Agent(
    name="weather_agent_claude",
    model=LiteLlm(model=MODEL_CLAUDE_SONNET),
    description="Provides weather information (using Claude Sonnet).",
    instruction="You are a helpful weather assistant powered by Claude Sonnet. "
                "Use the 'get_weather' tool for city weather requests. "
                "Analyze the tool's dictionary output ('status', 'report'/'error_message'). "
                "Clearly present successful reports or polite error messages.",
    tools=[get_weather],
)
```

#### 多模型对比观察

用同一个 Tool、同一个 query 测试三个 Agent，你会发现：
- Tool 执行结果完全一致（因为 Tool 是你写的固定逻辑）
- **最终回复的措辞、风格、格式可能不同**（因为每个 LLM 解读 instruction 的方式不同）

> **最佳实践**：用常量定义模型名（如 `MODEL_GPT_4O`），避免拼写错误。用 `try...except` 包裹 Agent 定义，某个模型的 Key 失效不会导致整个程序崩溃。

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **三件套架构**：Agent（决策） + Runner（编排） + SessionService（记忆），这是 ADK 的核心运行模型
2. **Event 驱动**：Runner 通过 yield Events 来报告执行进度，用 `is_final_response()` 找最终回复
3. **Tool 的 Docstring 是关键**：LLM 完全依赖 docstring 理解工具，类型注解也很重要
4. **LiteLlm 一行换模型**：Gemini 直接传字符串，其他模型用 `LiteLlm(model="provider/model_name")`
5. **description vs instruction**：description 给其他 Agent 看（用于委派决策），instruction 给自身 LLM 看（行为指南）
6. **Session 隔离**：不同 Agent 测试用不同的 SessionService/Session，避免历史污染
