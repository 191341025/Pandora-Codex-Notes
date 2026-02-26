# 模块五：安全护栏 — Callbacks 机制

> 对应 PDF 第 43-57 页（Step 5 ~ Step 6 + 总结）

---

## 概念讲解

### 1. Callbacks 是什么？

**定义**：Callbacks 是你定义的 Python 函数，ADK 会在 Agent 执行生命周期的特定时间点自动调用它们。简单说就是"钩子"——在关键操作前后插入自定义逻辑。

**为什么需要**：Agent 越强大越需要"安全带"。Callbacks 让你在以下关键节点做检查：
- **发给 LLM 之前**（`before_model_callback`）→ 输入护栏
- **Tool 执行之前**（`before_tool_callback`）→ 工具参数护栏

```
用户输入
    ↓
  [before_model_callback]  ← 检查/拦截/修改输入
    ↓
  LLM 推理 → 决定调用 Tool
    ↓
  [before_tool_callback]   ← 检查/拦截/修改工具参数
    ↓
  Tool 执行
    ↓
  LLM 生成最终回复
    ↓
  返回给用户
```

---

### 2. before_model_callback — 输入守卫

**触发时机**：Agent 把编排好的请求（对话历史 + instructions + 最新用户消息）发给 LLM **之前**。

**函数签名**：

```python
from google.adk.agents.callback_context import CallbackContext
from google.adk.models.llm_request import LlmRequest
from google.adk.models.llm_response import LlmResponse
from typing import Optional

def my_guardrail(
    callback_context: CallbackContext,
    llm_request: LlmRequest
) -> Optional[LlmResponse]:
    ...
```

**参数说明**：

| 参数 | 类型 | 作用 |
|------|------|------|
| `callback_context` | `CallbackContext` | 提供 Agent 信息、Session State（`callback_context.state`） |
| `llm_request` | `LlmRequest` | 即将发给 LLM 的完整 payload（对话内容、配置） |

**返回值语义**：

| 返回值 | 效果 |
|--------|------|
| `None` | **放行** — ADK 继续正常调用 LLM |
| `LlmResponse(...)` | **拦截** — ADK 跳过 LLM 调用，直接用这个 Response 回复用户 |

#### 示例：关键词拦截器

```python
def block_keyword_guardrail(
    callback_context: CallbackContext, llm_request: LlmRequest
) -> Optional[LlmResponse]:
    """检查用户输入中是否包含 'BLOCK' 关键词"""
    agent_name = callback_context.agent_name

    # 提取最新的用户消息
    last_user_message_text = ""
    if llm_request.contents:
        for content in reversed(llm_request.contents):
            if content.role == 'user' and content.parts:
                if content.parts[0].text:
                    last_user_message_text = content.parts[0].text
                    break

    # 检查关键词
    if "BLOCK" in last_user_message_text.upper():
        # 记录到 state
        callback_context.state["guardrail_block_keyword_triggered"] = True

        # 返回 LlmResponse = 拦截，直接回复用户
        return LlmResponse(
            content=types.Content(
                role="model",
                parts=[types.Part(text="I cannot process this request because it contains the blocked keyword 'BLOCK'.")],
            )
        )
    else:
        return None  # 放行
```

**常见用途**：

| 用途 | 说明 |
|------|------|
| 输入过滤 | 检查是否包含 PII、敏感词、违规内容 |
| 安全护栏 | 阻止有害/离题/违反策略的请求到达 LLM |
| 动态 Prompt 修改 | 在发送前注入来自 state 的上下文信息 |

---

### 3. before_tool_callback — 工具参数守卫

**触发时机**：LLM 已经决定调用某个 Tool 并生成了参数，但 Tool 函数**还没执行**之前。

**函数签名**：

```python
from google.adk.tools.base_tool import BaseTool
from google.adk.tools.tool_context import ToolContext
from typing import Optional, Dict, Any

def my_tool_guardrail(
    tool: BaseTool,
    args: Dict[str, Any],
    tool_context: ToolContext
) -> Optional[Dict]:
    ...
```

**参数说明**：

| 参数 | 类型 | 作用 |
|------|------|------|
| `tool` | `BaseTool` | 即将执行的工具对象，用 `tool.name` 获取名称 |
| `args` | `Dict[str, Any]` | LLM 为工具生成的参数字典 |
| `tool_context` | `ToolContext` | 提供 Session State（`tool_context.state`）、Agent 信息等 |

**返回值语义**：

| 返回值 | 效果 |
|--------|------|
| `None` | **放行** — Tool 正常执行（如果你修改了 `args`，Tool 用修改后的参数执行） |
| `Dict` | **拦截** — Tool 不执行，返回的 Dict 被当作 Tool 的执行结果传给 LLM |

#### 示例：城市黑名单

```python
def block_paris_tool_guardrail(
    tool: BaseTool, args: Dict[str, Any], tool_context: ToolContext
) -> Optional[Dict]:
    """阻止对 'Paris' 的天气查询"""
    target_tool = "get_weather_stateful"
    blocked_city = "paris"

    if tool.name == target_tool:
        city = args.get("city", "")
        if city and city.lower() == blocked_city:
            # 记录到 state
            tool_context.state["guardrail_tool_block_triggered"] = True

            # 返回 Dict = 拦截，这个 Dict 被当作 Tool 的执行结果
            return {
                "status": "error",
                "error_message": f"Policy restriction: Weather checks for '{city.capitalize()}' are currently disabled."
            }

    return None  # 放行
```

**常见用途**：

| 用途 | 说明 |
|------|------|
| 参数验证 | 检查参数是否有效、在允许范围内 |
| 资源保护 | 阻止对受限数据/敏感 API 的调用 |
| 动态参数修改 | 根据 state 调整参数（修改 `args` 后返回 `None`） |

---

### 4. 两种 Callback 的对比

| 维度 | `before_model_callback` | `before_tool_callback` |
|------|------------------------|----------------------|
| **触发时机** | LLM 调用前 | Tool 执行前 |
| **检查对象** | 用户输入（`llm_request.contents`） | 工具参数（`args`） |
| **拦截方式** | 返回 `LlmResponse` | 返回 `Dict`（伪装成 Tool 结果） |
| **放行方式** | 返回 `None` | 返回 `None` |
| **典型用途** | 输入过滤、关键词黑名单 | 参数验证、城市/资源黑名单 |
| **作用范围** | 只作用于定义它的 Agent | 只作用于定义它的 Agent |

> **重要**：Root Agent 上定义的 `before_model_callback` **不会自动传递给 Sub-Agent**。如果需要给 Sub-Agent 也加护栏，必须在 Sub-Agent 上单独定义。

---

### 5. 多层防护的协同工作

在 Agent 上同时配置两个 Callback：

```python
root_agent = Agent(
    name="weather_agent_v6_tool_guardrail",
    model=root_agent_model,
    tools=[get_weather_stateful],
    sub_agents=[greeting_agent, farewell_agent],
    output_key="last_weather_report",
    before_model_callback=block_keyword_guardrail,     # 第一道防线
    before_tool_callback=block_paris_tool_guardrail    # 第二道防线
)
```

**测试场景分析**：

| 请求 | before_model | before_tool | 结果 |
|------|-------------|-------------|------|
| "Weather in New York" | 放行（无 BLOCK 关键词） | 放行（不是 Paris） | 正常返回天气 |
| "BLOCK weather in Tokyo" | **拦截**（包含 BLOCK） | 不触发 | 返回拦截消息 |
| "Weather in Paris" | 放行 | **拦截**（Paris 黑名单） | 返回策略限制错误 |
| "Hello" | 放行 | 不触发（委派给 greeting_agent，它没有 tool callback） | 正常问候 |

**执行链路图**：

```
"Weather in Paris"
    ↓
before_model_callback → 检查输入 → 无 BLOCK → 放行 (return None)
    ↓
LLM 推理 → 决定调用 get_weather_stateful(city="Paris")
    ↓
before_tool_callback → 检查参数 → 发现 Paris → 拦截!
    ↓
返回 {"status": "error", "error_message": "Policy restriction..."}
    ↓
LLM 收到这个 "Tool 结果" → 组织回复给用户
    ↓
Agent: "Sorry, weather checks for Paris are currently disabled."
```

> **注意**：Tool callback 返回的 Dict 被 LLM 当作真实的 Tool 执行结果。LLM 不知道 Tool 其实没执行，它会基于这个 "结果" 来生成回复。

---

### 6. 教程总结与完整架构

经过 Step 1-6，你构建的完整 Agent 系统：

```
                        ┌─────────────────────────────────┐
  用户输入 ───────────→ │        Root Agent (v6)           │
                        │  before_model_callback ✓        │
                        │  before_tool_callback  ✓        │
                        │  tools: [get_weather_stateful]  │
                        │  output_key: last_weather_report│
                        │  sub_agents: [greeting, farewell]│
                        └───────┬──────────┬──────────────┘
                                │          │
                  ┌─────────────┘          └──────────────┐
                  ↓                                        ↓
        ┌──────────────────┐                    ┌──────────────────┐
        │  greeting_agent  │                    │  farewell_agent  │
        │  tools: [say_hi] │                    │  tools: [say_bye]│
        └──────────────────┘                    └──────────────────┘
```

**你掌握的核心能力**：

| 能力 | 机制 |
|------|------|
| 工具定义 | Python 函数 + Docstring |
| 多模型切换 | `LiteLlm(model="provider/name")` |
| 多 Agent 协作 | `sub_agents` + Auto Flow |
| 状态记忆 | `ToolContext.state` + `output_key` |
| 输入安全 | `before_model_callback` |
| 工具安全 | `before_tool_callback` |

**进阶方向**：
- `after_model_callback`：在 LLM 回复后格式化/清洗输出
- `after_tool_callback`：在 Tool 执行后处理/记录结果
- `before_agent_callback` / `after_agent_callback`：Agent 级别的进入/退出钩子
- 替换 `InMemorySessionService` 为持久化存储（Firestore、Cloud SQL 等）
- 替换 mock 数据为真实 API（OpenWeatherMap 等）

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **返回值决定一切**：`None` = 放行，返回对象 = 拦截（LlmResponse 或 Dict）
2. **Callback 不传递给 Sub-Agent**：Root Agent 的 callback 不自动保护 Sub-Agent
3. **Tool callback 返回的 Dict 伪装成 Tool 结果**：LLM 不知道 Tool 没执行
4. **两道防线**：before_model 拦截输入，before_tool 拦截参数，可以同时使用
5. **Callback 可以读写 State**：通过 `callback_context.state` 或 `tool_context.state` 记录拦截事件
