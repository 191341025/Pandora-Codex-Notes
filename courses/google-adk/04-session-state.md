# 模块四：状态管理 — Session State 与 Agent 记忆

> 对应 PDF 第 35-43 页（Step 4）

---

## 概念讲解

### 1. 为什么 Agent 需要记忆？

**问题**：到 Step 3 为止，Agent Team 能处理不同任务，但每次对话都是"失忆"的——不记得用户偏好，不记得上次查了什么城市。

**解决方案**：ADK 的 **Session State** —— 一个绑定在会话（Session）上的 Python 字典。

**Session State 是什么**：
- 本质上就是 `session.state`，一个 `dict`
- 绑定在特定的 `(app_name, user_id, session_id)` 组合上
- 在同一个 Session 内跨多轮对话持久化
- Agent 和 Tool 都可以读写

---

### 2. Agent 与 State 交互的两种方式

| 方式 | 机制 | 适用场景 |
|------|------|----------|
| **ToolContext** | Tool 函数声明 `tool_context: ToolContext` 参数，ADK 自动注入 | 在 Tool 执行时读写 state |
| **output_key** | Agent 配置 `output_key="key_name"` | 自动把 Agent 最后的回复文本保存到 state |

---

### 3. 初始化带状态的 Session

创建 Session 时可以传入初始状态：

```python
from google.adk.sessions import InMemorySessionService

session_service_stateful = InMemorySessionService()

initial_state = {
    "user_preference_temperature_unit": "Celsius"   # 用户偏好：摄氏度
}

session_stateful = await session_service_stateful.create_session(
    app_name=APP_NAME,
    user_id=USER_ID_STATEFUL,
    session_id=SESSION_ID_STATEFUL,
    state=initial_state         # ← 初始化 state
)
```

**验证**：创建后可以用 `get_session()` 取回来看：

```python
retrieved_session = await session_service_stateful.get_session(
    app_name=APP_NAME,
    user_id=USER_ID_STATEFUL,
    session_id=SESSION_ID_STATEFUL
)
print(retrieved_session.state)
# {'user_preference_temperature_unit': 'Celsius'}
```

---

### 4. ToolContext — Tool 函数读写状态

**定义**：`ToolContext` 是 ADK 自动注入给 Tool 的上下文对象，让 Tool 可以读写 Session State。

**用法**：只需在 Tool 函数的**最后一个参数**声明 `tool_context: ToolContext`，ADK 就会自动传入。

```python
from google.adk.tools.tool_context import ToolContext

def get_weather_stateful(city: str, tool_context: ToolContext) -> dict:
    """Retrieves weather, converts temp unit based on session state."""

    # 读取 state 中的用户偏好
    preferred_unit = tool_context.state.get("user_preference_temperature_unit", "Celsius")

    city_normalized = city.lower().replace(" ", "")
    mock_weather_db = {
        "newyork": {"temp_c": 25, "condition": "sunny"},
        "london":  {"temp_c": 15, "condition": "cloudy"},
        "tokyo":   {"temp_c": 18, "condition": "light rain"},
    }

    if city_normalized in mock_weather_db:
        data = mock_weather_db[city_normalized]
        temp_c = data["temp_c"]
        condition = data["condition"]

        # 根据 state 偏好格式化温度
        if preferred_unit == "Fahrenheit":
            temp_value = (temp_c * 9/5) + 32
            temp_unit = "°F"
        else:
            temp_value = temp_c
            temp_unit = "°C"

        report = f"The weather in {city.capitalize()} is {condition} with a temperature of {temp_value:.0f}{temp_unit}."

        # 写入 state（记录最近查询的城市）
        tool_context.state["last_city_checked_stateful"] = city

        return {"status": "success", "report": report}
    else:
        return {"status": "error", "error_message": f"Sorry, I don't have weather information for '{city}'."}
```

**关键操作总结**：

| 操作 | 代码 | 说明 |
|------|------|------|
| 读取 state | `tool_context.state.get("key", default)` | 用 `.get()` 带默认值，防止 key 不存在时崩溃 |
| 写入 state | `tool_context.state["key"] = value` | 直接赋值即可，持久化到当前 Session |

> **最佳实践**：读取时永远用 `dict.get('key', default_value)`，不要直接 `dict['key']`，避免 KeyError。

---

### 5. output_key — 自动保存 Agent 回复

**定义**：在 Agent 定义时设置 `output_key`，ADK 会在每轮对话结束时，自动把该 Agent 的最终文本回复保存到 `session.state[output_key]`。

```python
root_agent_stateful = Agent(
    name="weather_agent_v4_stateful",
    model=root_agent_model,
    description="Main agent: Provides weather (state-aware unit), delegates greetings/farewells.",
    instruction="You are the main Weather Agent...",
    tools=[get_weather_stateful],
    sub_agents=[greeting_agent, farewell_agent],
    output_key="last_weather_report"   # ← 自动保存最终回复到 state
)
```

**行为**：
- Agent 每次回复后，`session.state["last_weather_report"]` 会被自动更新
- 如果 Agent 委派给 Sub-Agent，output_key 保存的是 Root Agent 层面的最终输出
- **会覆盖**：新回复会覆盖旧值

> **注意**：如果最后一轮是委派给 greeting_agent 的 "Hello there!"，那么 `last_weather_report` 的值就是 "Hello there!"，不再是上次的天气报告。output_key 保存的是"最后一次回复"，不管内容是什么。

---

### 6. 状态驱动的多轮对话实验

完整的测试流程：

```
Turn 1: "What's the weather in London?"
  → Tool 读取 state: unit = "Celsius"
  → 返回 "15°C"
  → output_key 保存天气报告
  → Tool 写入 state: last_city = "London"

手动修改 state: unit → "Fahrenheit"

Turn 2: "Tell me the weather in New York."
  → Tool 读取 state: unit = "Fahrenheit"
  → 计算 25°C → 77°F
  → 返回 "77°F"
  → output_key 覆盖保存新天气报告
  → Tool 写入 state: last_city = "New York"

Turn 3: "Hi!"
  → 委派给 greeting_agent
  → 返回 "Hello there!"
  → output_key 覆盖保存为 "Hello there!"（不再是天气报告）
```

**手动修改 state 的方法（仅限 InMemorySessionService 测试用）**：

```python
stored_session = session_service_stateful.sessions[APP_NAME][USER_ID][SESSION_ID]
stored_session.state["user_preference_temperature_unit"] = "Fahrenheit"
```

> **注意**：`get_session()` 返回的是 **副本**，修改副本不会影响实际存储。测试时需要直接访问 `session_service_stateful.sessions` 内部字典。生产环境中应该通过 Tool 或 Agent 逻辑（`EventActions(state_delta=...)`）来修改 state。

---

### 7. 最终状态检查

```python
final_session = await session_service_stateful.get_session(
    app_name=APP_NAME,
    user_id=USER_ID_STATEFUL,
    session_id=SESSION_ID_STATEFUL
)

print(final_session.state.get('user_preference_temperature_unit'))  # "Fahrenheit"
print(final_session.state.get('last_weather_report'))               # "Hello there!"（最后一轮的回复）
print(final_session.state.get('last_city_checked_stateful'))        # "New York"
```

**确认清单**：

| 验证项 | 预期结果 |
|--------|---------|
| Tool 读取初始 state | London 用 Celsius (15°C) |
| 手动修改 state 后 Tool 读取 | New York 用 Fahrenheit (77°F) |
| Tool 写入 state | `last_city_checked_stateful` = "New York" |
| output_key 保存 | `last_weather_report` = 最后一轮回复 |
| 委派仍然正常 | "Hi!" 委派给 greeting_agent |

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **ToolContext 是 Tool 与 State 的桥梁**：声明为最后一个参数，ADK 自动注入，通过 `tool_context.state` 读写
2. **`state.get('key', default)` 防崩溃**：永远用 `.get()` 带默认值读取 state
3. **output_key 自动保存但会覆盖**：每轮回复都覆盖，包括委派后的回复
4. **`get_session()` 返回副本**：修改副本不影响实际 state，测试时需直接操作内部存储
5. **State 贯穿整个 Agent 生命周期**：初始化 → Tool 读写 → output_key 保存 → 跨轮持久化
