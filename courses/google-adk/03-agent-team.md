# 模块三：多 Agent 协作 — Agent Team 与任务委派

> 对应 PDF 第 28-35 页（Step 3）

---

## 概念讲解

### 1. 为什么需要 Agent Team？

**问题**：把所有功能堆到一个 Agent 里（天气、打招呼、告别、计算……），instructions 会变得又长又复杂，难以维护。

**解决方案**：Agent Team 模式——把大问题拆成小问题，每个小问题交给专门的 Agent 处理。

```
用户请求 → Root Agent（调度员） → 分析意图 → 委派给合适的 Sub-Agent → Sub-Agent 处理 → 返回结果
```

**为什么这样做更好**：

| 优势 | 说明 |
|------|------|
| 模块化 | 每个 Agent 独立开发、测试、维护 |
| 专业化 | 每个 Agent 的 instruction 和模型选择可以针对性优化 |
| 可扩展 | 加新功能 = 加新 Agent，不用改现有代码 |
| 成本优化 | 简单任务（打招呼）可以用便宜的模型，复杂任务用强模型 |

---

### 2. 定义 Sub-Agent 的工具

每个 Sub-Agent 有自己专属的工具：

```python
from typing import Optional

def say_hello(name: Optional[str] = None) -> str:
    """Provides a simple greeting. If a name is provided, it will be used.
    Args:
        name (str, optional): The name of the person to greet.
    Returns:
        str: A friendly greeting message.
    """
    if name:
        return f"Hello, {name}!"
    else:
        return "Hello there!"

def say_goodbye() -> str:
    """Provides a simple farewell message to conclude the conversation."""
    return "Goodbye! Have a great day."
```

> **注意**：`say_hello` 的 `name` 参数是 `Optional[str]`，这样 LLM 可以选择传不传名字。

---

### 3. 定义专业化的 Sub-Agent

每个 Sub-Agent 有两个关键属性：**instruction（行为指南）** 和 **description（能力描述）**。

#### Greeting Agent

```python
greeting_agent = Agent(
    model=MODEL_GEMINI_2_5_FLASH,
    name="greeting_agent",
    instruction="You are the Greeting Agent. Your ONLY task is to provide a friendly greeting. "
                "Use the 'say_hello' tool to generate the greeting. "
                "If the user provides their name, make sure to pass it to the tool. "
                "Do not engage in any other conversation or tasks.",
    description="Handles simple greetings and hellos using the 'say_hello' tool.",
    tools=[say_hello],
)
```

#### Farewell Agent

```python
farewell_agent = Agent(
    model=MODEL_GEMINI_2_5_FLASH,
    name="farewell_agent",
    instruction="You are the Farewell Agent. Your ONLY task is to provide a polite goodbye message. "
                "Use the 'say_goodbye' tool when the user indicates they are leaving "
                "(e.g., 'bye', 'goodbye', 'thanks bye', 'see you'). "
                "Do not perform any other actions.",
    description="Handles simple farewells and goodbyes using the 'say_goodbye' tool.",
    tools=[say_goodbye],
)
```

**最佳实践**：

- **`description` 要精准**：Root Agent 的 LLM 靠 description 决定委派给谁，写不好就委派错
- **`instruction` 要限定范围**：明确说"Your ONLY task is..."，避免 Sub-Agent 越权处理不该管的事

> **类比**：`description` 就像员工的"岗位描述"（给领导看的），`instruction` 就像"工作手册"（自己执行任务时看的）。

---

### 4. Root Agent 与自动委派（Auto Flow）

Root Agent 是整个 Team 的入口，它需要：
1. 保留自己的核心能力（天气查询）
2. 知道有哪些 Sub-Agent 可以委派
3. 根据用户意图决定自己处理还是委派

```python
weather_agent_team = Agent(
    name="weather_agent_v2",
    model=root_agent_model,
    description="The main coordinator agent. Handles weather requests and delegates greetings/farewells to specialists.",
    instruction="You are the main Weather Agent coordinating a team. "
                "Your primary responsibility is to provide weather information. "
                "Use the 'get_weather' tool ONLY for specific weather requests. "
                "You have specialized sub-agents: "
                "1. 'greeting_agent': Handles simple greetings like 'Hi', 'Hello'. Delegate to it for these. "
                "2. 'farewell_agent': Handles simple farewells like 'Bye', 'See you'. Delegate to it for these. "
                "Analyze the user's query. If it's a greeting, delegate to 'greeting_agent'. "
                "If it's a farewell, delegate to 'farewell_agent'. "
                "If it's a weather request, handle it yourself using 'get_weather'. "
                "For anything else, respond appropriately or state you cannot handle it.",
    tools=[get_weather],
    sub_agents=[greeting_agent, farewell_agent]   # ← 关键：注册 Sub-Agent
)
```

**`sub_agents` 参数就是魔法所在。** 只要传了 Sub-Agent 列表，ADK 就会自动启用 **Auto Flow（自动委派）**：

1. Root Agent 的 LLM 收到用户消息
2. LLM 同时看到：自己的 instruction + 自己的 tools + **每个 Sub-Agent 的 `name` 和 `description`**
3. LLM 判断：这个请求应该自己处理，还是委派给某个 Sub-Agent？
4. 如果决定委派，ADK 会生成一个内部 action，将控制权转给对应的 Sub-Agent
5. Sub-Agent 用自己的模型、instruction、tools 独立处理请求
6. 处理结果返回

> **关键洞察**：Auto Flow 的委派决策完全依赖 LLM 的判断。Root Agent 的 instruction 要写清楚什么情况委派给谁，Sub-Agent 的 description 要写清楚自己能做什么。

---

### 5. 执行流程分析

测试三条消息，观察委派行为：

```python
await call_agent_async("Hello there!", ...)           # → greeting_agent
await call_agent_async("What is the weather in New York?", ...)  # → root agent 自己处理
await call_agent_async("Thanks, bye!", ...)            # → farewell_agent
```

**执行流程图**：

```
"Hello there!"
    → Root Agent LLM 分析 → 匹配 greeting_agent 的 description
    → 委派给 greeting_agent
    → greeting_agent 调用 say_hello()
    → 返回 "Hello there!"

"What is the weather in New York?"
    → Root Agent LLM 分析 → 天气请求，自己处理
    → 调用 get_weather("New York")
    → 返回天气报告

"Thanks, bye!"
    → Root Agent LLM 分析 → 匹配 farewell_agent 的 description
    → 委派给 farewell_agent
    → farewell_agent 调用 say_goodbye()
    → 返回 "Goodbye! Have a great day."
```

**如何验证委派成功**：看日志中 `--- Tool: ... called ---` 的输出：
- `say_hello` 被调用 → 说明 greeting_agent 处理了请求
- `get_weather` 被调用 → 说明 root agent 自己处理了
- `say_goodbye` 被调用 → 说明 farewell_agent 处理了请求

---

### 6. Agent Team 架构总结

```
                    ┌──────────────────────┐
   用户请求 ──────→ │   Root Agent (v2)     │
                    │   tools: [get_weather]│
                    │   sub_agents: [...]   │
                    └───────┬──────┬────────┘
                            │      │
              ┌─────────────┘      └─────────────┐
              ↓                                   ↓
    ┌─────────────────┐                 ┌─────────────────┐
    │ greeting_agent  │                 │ farewell_agent  │
    │ tools: [say_hi] │                 │ tools: [say_bye]│
    └─────────────────┘                 └─────────────────┘
```

**委派决策依据**：

| 用户意图 | 匹配的 description | 处理者 |
|----------|-------------------|--------|
| "Hi" / "Hello" / "Hey" | "Handles simple greetings and hellos" | greeting_agent |
| 天气查询 | Root agent 自己的 instruction | Root Agent |
| "Bye" / "See you" / "Thanks bye" | "Handles simple farewells and goodbyes" | farewell_agent |
| 其他 | 无匹配 | Root Agent（兜底） |

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **`sub_agents` 开启 Auto Flow**：只需把 Sub-Agent 列表传给 Root Agent，ADK 自动处理委派逻辑
2. **`description` 是委派的关键**：Root Agent 的 LLM 靠 Sub-Agent 的 description 决定把任务分给谁
3. **`instruction` 要限定范围**：Sub-Agent 的 instruction 要明确说"ONLY task"，避免越权
4. **Root Agent 也能直接处理请求**：它不只是调度员，还保留自己的 tools 处理核心任务
5. **每个 Agent 可以用不同模型**：简单任务用便宜模型（如 Flash），复杂任务用强模型（如 GPT-4o）
