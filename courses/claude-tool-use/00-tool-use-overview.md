# 模块 00：Tool Use 核心概念与工作原理

> 对应源文件：`tools/overview.md`

---

## 概念地图

- **核心概念** (必须内化): Tool Use 交互模型、Client Tool vs Server Tool
- **实操要点** (动手时需要): 调用模式（Sequential / Parallel / Chain-of-thought）、MCP 工具集成
- **背景知识** (扩展理解): Tool Use 定价机制

---

## 概念讲解

### 1. Tool Use 交互模型

Claude 本身不能执行代码、不能访问数据库、不能调 API——它只会"说话"。Tool Use（工具使用）就是让 Claude **从只能说，变成能做事**的机制。

这个机制的核心思路出奇地简单：你告诉 Claude "你手边有哪些工具可以用"，Claude 在需要的时候说"我想用这个工具，参数是这些"，你去执行，把结果告诉它，它再继续回答。

**直觉建立**：

想象你在和一个被困在隔音房间里的专家对话。这个专家知识渊博，但看不到外面的世界。你们之间有一个小窗口可以传递纸条：

1. 你把一张菜单（工具列表）和一个问题递进去
2. 专家看完后，写了一张纸条："请帮我查一下旧金山的天气"（工具调用请求）
3. 你去查了，把结果写在纸条上递回去："15°C，多云"
4. 专家拿到数据，写出最终回答："旧金山现在 15°C，多云，建议带件外套"

这就是 Tool Use 的全部——Claude 决定**要不要用**工具、**用哪个**工具、**传什么参数**；你负责**真正执行**并返回结果。

> **类比边界**：现实中专家可以自己走出房间。Claude 不行——它永远需要你作为"手脚"去执行工具。这个约束是故意的：确保你对工具执行拥有完全控制权。

**协议细节**——一次完整的 Tool Use 交互在 API 层面是这样的：

```
你 → Claude:  messages: [{role: "user", content: "旧金山天气怎样？"}]
              tools: [{name: "get_weather", description: "...", input_schema: {...}}]

Claude → 你:  stop_reason: "tool_use"
              content: [{type: "text", text: "我来查一下..."},
                        {type: "tool_use", id: "toolu_xxx", name: "get_weather", input: {location: "San Francisco, CA"}}]

你 → Claude:  messages: [...之前的对话...,
              {role: "user", content: [{type: "tool_result", tool_use_id: "toolu_xxx", content: "15°C, mostly cloudy"}]}]

Claude → 你:  stop_reason: "end_turn"
              content: [{type: "text", text: "旧金山现在 15°C，多云..."}]
```

注意几个关键点：

| 要素 | 说明 |
|------|------|
| `stop_reason: "tool_use"` | Claude 在告诉你："我想用工具了，先暂停，等你给我结果" |
| `tool_use_id` | 每次工具调用的唯一 ID，用于匹配请求和结果 |
| `tool_result` 必须放在 `user` 消息中 | 因为 API 协议中只有 user 和 assistant 两种角色——工具结果从 Claude 的视角看就是"用户告诉我的信息" |
| 工具定义每次都要传 | `tools` 参数在每轮请求中都要带上（后面会讲 Prompt Caching 来优化这个开销） |

> **你正在体验的 Tool Use**：此刻你和我的对话中，Claude Code 就在频繁使用 Tool Use——当它需要读文件时调用 Read 工具，需要搜索时调用 Grep 工具，需要写文件时调用 Write 工具。你看到的每一次文件操作，底层都是一次 tool_use → tool_result 的来回。

### 2. Client Tool vs Server Tool

Claude 的工具分两种，区别在于**谁来执行**：

| | Client Tool（客户端工具） | Server Tool（服务端工具） |
|--|--------------------------|--------------------------|
| **执行者** | 你的代码 | Anthropic 的服务器 |
| **控制权** | 完全在你手上 | Anthropic 代为执行 |
| **交互模式** | 每次调用都暂停，等你返回结果 | Claude 自己在服务端循环执行，直到完成 |
| **stop_reason** | `tool_use`（需要你介入） | 通常直接返回最终结果 |
| **典型例子** | 自定义工具（查数据库、调内部 API）、Computer Use、Text Editor | Web Search、Web Fetch |
| **你需要写代码吗** | 必须——你要实现工具的执行逻辑 | 不需要——只需在请求中声明要用 |

**Client Tool 流程（四步）**：

```
1. 你传工具定义 + 用户提问
2. Claude 决定用哪个工具 → 返回 tool_use（暂停）
3. 你执行工具 → 返回 tool_result
4. Claude 根据结果生成最终回答
```

**Server Tool 流程（自动循环）**：

```
1. 你传工具声明 + 用户提问
2. Claude 在 Anthropic 服务器上自动执行工具，结果直接注入上下文
3. Claude 生成最终回答（可能经过多轮服务端循环）
```

Server Tool 的关键差异在于**你看不到中间过程**——Claude 在 Anthropic 的服务器上可能连续搜索了三次网页，但你只拿到最终整合后的回答。

> **`pause_turn` 特殊情况**：Server Tool 的服务端循环默认上限是 10 次迭代。如果 Claude 还没完成就达到上限，API 返回 `stop_reason: "pause_turn"`。这时你需要把响应原样发回去，让 Claude 继续处理——类似"翻页"操作。

**怎么选择？**

- 需要访问你的私有数据或内部系统 → **Client Tool**（数据不会经过 Anthropic 服务器）
- 需要搜索公开网页信息 → **Server Tool**（Web Search / Web Fetch，开箱即用）
- 需要在用户的电脑上操作 → **Client Tool**（Computer Use / Bash / Text Editor，虽然是 Anthropic 定义的工具，但执行在你的环境中）

> **常见误区**：不是所有 Anthropic 定义的工具都是 Server Tool。Computer Use、Text Editor、Bash 这三个虽然由 Anthropic 设计（使用版本化类型名如 `computer_20250124`），但它们是 **Client Tool**——需要你在客户端实现执行逻辑。Server Tool 目前只有 Web Search 和 Web Fetch。

### 3. 调用模式：Sequential、Parallel、Chain-of-thought

当你给 Claude 多个工具时，它会根据任务需要选择不同的调用模式：

**Sequential（顺序调用）**——后一个工具的输入依赖前一个工具的输出：

```
用户："我这里天气怎样？"（用户没说地点）

Claude → get_location() → "San Francisco, CA"
Claude → get_weather("San Francisco, CA") → "15°C, 多云"
Claude → "你在旧金山，现在 15°C，多云"
```

Claude 足够聪明，知道没有地点就查不了天气，所以先调 `get_location` 拿到地点，再调 `get_weather`。每次调用都是一轮完整的 API 请求-响应。

**Parallel（并行调用）**——多个工具之间没有依赖关系，一次性全部发出：

```
用户："纽约天气怎样？那里现在几点？"

Claude → [get_weather("New York"), get_time("America/New_York")]  // 一次性返回两个 tool_use
你 → [weather_result, time_result]  // 一次性返回两个 tool_result
Claude → "纽约现在下午 3 点，气温 20°C..."
```

并行调用的关键：**所有 `tool_use` 在同一个 assistant 消息中**，所有对应的 `tool_result` 必须在**同一个 user 消息中**返回。格式不对会导致 API 报错或 Claude 后续不再使用并行调用。

**Chain-of-thought（先思考再调用）**：

Claude Opus 默认会在调用工具前先"想一下"——评估是否真的需要工具、用哪个工具、参数是否充分。而 Sonnet 和 Haiku 更倾向于"先调再说"，可能在信息不足时猜测参数。

如果你希望 Sonnet/Haiku 也先思考再行动，可以在 system prompt 中加入引导：

```
Answer the user's request using relevant tools (if they are available).
Before calling a tool, do some analysis. First, think about which of the
provided tools is the relevant tool to answer the user's request. Second,
go through each of the required parameters of the relevant tool and
determine if the user has directly provided or given enough information
to infer a value...
```

这本质上就是 CoT（Chain-of-Thought）策略在 Tool Use 场景中的应用——你在 Prompt Engineering 阶段已经学过的技巧，这里直接复用。

> **实战判断**：
> - 用 Opus → 不需要额外引导，它天然会思考后再决策
> - 用 Sonnet/Haiku → 需要权衡：加 CoT 引导会更准确但更慢，不加则更快但可能调用不必要的工具
> - 参数缺失时：Opus 更可能追问，Sonnet 更可能猜测（比如没给地点就猜 New York）

### 4. MCP 工具集成

MCP（Model Context Protocol，模型上下文协议）是一个让 AI 模型连接外部工具和数据源的开放标准。如果你已经有 MCP Server 提供的工具，可以直接接入 Claude 的 Tool Use。

转换很简单——MCP 的工具定义用 `inputSchema`（驼峰），Claude 的 API 用 `input_schema`（下划线）：

```python
async def get_claude_tools(mcp_session):
    """把 MCP 工具转换成 Claude 的格式"""
    mcp_tools = await mcp_session.list_tools()

    return [
        {
            "name": tool.name,
            "description": tool.description or "",
            "input_schema": tool.inputSchema,  # 驼峰 → 下划线，仅此而已
        }
        for tool in mcp_tools.tools
    ]
```

转换后，当 Claude 返回 `tool_use` 时，你用 MCP 的 `call_tool()` 执行，再把结果以 `tool_result` 返回给 Claude。整个流程和普通 Client Tool 完全一致。

> **如果不想自己写 MCP Client**：Anthropic 提供了 MCP Connector，可以直接从 Messages API 连接远程 MCP Server，免去客户端实现。这属于 Tool Infrastructure 的范畴，Module 03 会详细展开。

### 5. Tool Use 定价机制

Tool Use 没有额外的"功能费"——收费逻辑和普通 API 请求一样，按 input/output token 计费。但 Token 开销会比普通对话高，因为多了几样东西：

| 额外 Token 来源 | 说明 |
|-----------------|------|
| `tools` 参数本身 | 工具名、描述、JSON Schema 都要序列化成 Token |
| 系统提示词（自动注入） | 当你传了 `tools`，Claude 会自动添加一段启用 Tool Use 的系统提示词 |
| `tool_use` 内容块 | Claude 输出的工具调用请求 |
| `tool_result` 内容块 | 你返回的工具执行结果 |
| 多轮对话累积 | 每一轮都要带上完整的对话历史（包括之前的 tool_use 和 tool_result） |

自动注入的系统提示词开销（至少传了 1 个工具时）：

| 模型 | `auto`/`none` | `any`/`tool` |
|------|---------------|--------------|
| Claude 4.x 系列 | 346 tokens | 313 tokens |
| Claude 3.5 Haiku | 264 tokens | 340 tokens |
| Claude 3 系列 | 159-530 tokens | 235-340 tokens |

> **成本优化思路**：工具定义是每轮都传的"固定开销"。如果你的工具列表很长，可以用 **Prompt Caching**（Module 04）来缓存这部分，避免每轮都重新计费。也可以用 **Tool Search**（Module 03）让 Claude 从大量工具中动态检索需要的子集，减少每次传递的工具数量。

---

## 重点标记

1. **Tool Use 是"声明-请求-执行-反馈"协议**：你声明工具，Claude 请求调用，你执行并返回结果——控制权始终在你手上
2. **Client Tool vs Server Tool 的根本区别是执行位置**：不是谁定义的，而是谁运行的。Computer Use 由 Anthropic 定义，但在你的环境执行，是 Client Tool
3. **`tool_result` 放在 `user` 消息中**：这是 API 协议约束，不是设计缺陷——从 Claude 视角看，工具结果就是来自外部的输入
4. **并行调用格式严格**：所有 `tool_result` 必须在同一个 `user` 消息中，否则 Claude 后续可能退化为顺序调用
5. **Token 开销比普通对话高**：工具定义 + 系统提示词 + 工具调用/结果都占 token，多轮对话还会累积。Prompt Caching 是关键优化手段

---

## 自测：你真的理解了吗？

**Q1**：你在开发一个客服机器人，需要 Claude 查询内部订单数据库和搜索公开的退货政策网页。你会把"查订单"和"搜退货政策"分别设计成什么类型的工具？为什么？

**Q2**：用户问"帮我查一下北京和上海的天气"。Claude 返回了两个 `tool_use` 块（`get_weather("北京")` 和 `get_weather("上海")`）。你应该返回几个 `tool_result`？放在几条消息里？如果分两条消息返回会怎样？

**Q3**：你发现 Claude Sonnet 在参数不足时总是猜测，经常猜错。你有两个优化方向：A) 在 system prompt 里加 CoT 引导；B) 把所有参数都标为 required。哪个更合理？为什么？两个都用会怎样？

**Q4**：你的应用有 20 个自定义工具，每次 API 请求都传全部 20 个。用户反馈"每次调用都好贵"。你能想到至少两种降低 Token 开销的方法吗？（提示：一种在本模块提到了，另一种会在后续模块详细讲。）
