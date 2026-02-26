# 模块 03：工具基础设施与 Files API

> 对应源文件：`tool-infrastructure/tool-search.md`, `programmatic-tool-calling.md`, `fine-grained-tool-streaming.md`, `files-and-assets/files-api.md`

---

## 概念地图

- **核心概念** (必须内化): Tool Search（defer_loading + 动态发现）、Programmatic Tool Calling（allowed_callers + 代码内调用工具）
- **实操要点** (动手时需要): Files API（上传/引用/下载）、Fine-grained Tool Streaming（eager_input_streaming）
- **背景知识** (扩展理解): 自定义 Tool Search 实现（embedding 检索）、Container 生命周期管理

---

## 概念讲解

### 1. Tool Search：当工具太多时怎么办

Module 00 提到"工具定义每次都要传"，Module 01 指出"30+ 工具时选择准确率下降"。当你的工具库增长到几十甚至几千个时，两个问题同时爆发：

- **Context 撑不下**：50 个工具定义 ≈ 10,000-20,000 tokens，吃掉大量上下文空间
- **选择变差**：工具越多，Claude 越容易选错或调用不必要的工具

Tool Search 的解决方案是**延迟加载**——不把所有工具定义一次性塞进上下文，而是让 Claude **先搜索再加载**需要的那几个。

**直觉建立**：

你手边有一本 500 页的工具手册。传统做法是每次干活前把整本手册复印一份放在桌上。Tool Search 的做法是：桌上只放一个搜索框——你输入关键词，手册自动翻到对应页，只把那几页给你。

**工作流程**：

```
1. 你在 tools 中声明 Tool Search 工具 + 所有工具定义（标记 defer_loading: true）
2. Claude 初始只看到 Tool Search 工具 + 未标记 defer 的工具
3. Claude 需要某个工具时 → 搜索（构造 regex 或自然语言查询）
4. API 返回 3-5 个最相关的 tool_reference
5. tool_reference 自动展开为完整工具定义
6. Claude 从发现的工具中选择并调用
```

**两种搜索变体**：

| 变体 | 类型名 | 查询方式 | 适用场景 |
|------|--------|---------|----------|
| Regex | `tool_search_tool_regex_20251119` | Claude 构造 Python 正则表达式 | 工具命名规范、前缀一致（如 `slack_*`, `github_*`） |
| BM25 | `tool_search_tool_bm25_20251119` | Claude 用自然语言描述需求 | 工具命名不规则、描述差异大 |

两种变体都搜索**工具名、描述、参数名、参数描述**四个字段。Regex 最大查询长度 200 字符。

**关键实现细节**：

```python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=2048,
    messages=[{"role": "user", "content": "What's the weather in SF?"}],
    tools=[
        # 搜索工具本身——不能标 defer_loading
        {"type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex"},

        # 常用工具——不标 defer，始终可见
        {"name": "get_location", "description": "...", "input_schema": {...}},

        # 大量工具——标记 defer_loading: true，按需加载
        {"name": "get_weather", "description": "...", "input_schema": {...},
         "defer_loading": True},
        {"name": "get_stock_price", "description": "...", "input_schema": {...},
         "defer_loading": True},
        # ... 几百个工具
    ]
)
```

**三条硬规则**：

1. **Tool Search 自身不能标 defer_loading**——它是搜索入口，必须始终可见
2. **至少保留一个非 defer 工具**——全部 defer 会报 400 错误
3. **被引用的工具必须有完整定义**——`tool_reference` 指向的工具必须在 `tools` 数组中存在（含 `defer_loading: true`）

**最佳实践**：把 3-5 个最常用的工具保持非 defer（始终可见），其余全部 defer。在工具描述中加入**语义关键词**（用户描述任务时可能用到的词），提升搜索命中率。

**MCP 集成——defer 整个 MCP Server**：

如果你通过 MCP Connector 接入远程 MCP Server（可能提供几十个工具），可以用 `mcp_toolset` 把整个 Server 的工具批量 defer：

```python
response = client.beta.messages.create(
    model="claude-opus-4-6",
    betas=["mcp-client-2025-11-20"],
    max_tokens=2048,
    mcp_servers=[
        {"type": "url", "name": "database-server", "url": "https://mcp-db.example.com"}
    ],
    tools=[
        {"type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex"},
        {
            "type": "mcp_toolset",
            "mcp_server_name": "database-server",
            "default_config": {"defer_loading": True},  # 默认全部 defer
            "configs": {
                "search_events": {"defer_loading": False}  # 这个工具始终可见
            }
        }
    ],
    messages=[...]
)
```

`default_config` 设置全局默认，`configs` 按工具名覆盖——这样你不需要逐个管理 MCP Server 的几十个工具。

**自定义搜索实现**：

如果内置的 regex/BM25 不够（比如你想用 embedding 语义搜索），可以实现自己的搜索工具——只要返回 `tool_reference` 块：

```json
// 你的自定义搜索工具返回的 tool_result
{
  "type": "tool_result",
  "tool_use_id": "toolu_your_search",
  "content": [
    {"type": "tool_reference", "tool_name": "discovered_tool_1"},
    {"type": "tool_reference", "tool_name": "discovered_tool_2"}
  ]
}
```

被引用的工具必须在 `tools` 数组中有完整定义（`defer_loading: true`）。API 会自动把 `tool_reference` 展开为完整定义。

**限制**：

| 限制项 | 值 |
|--------|------|
| 工具目录上限 | 10,000 个工具 |
| 每次搜索返回 | 3-5 个工具 |
| Regex 查询长度 | 最多 200 字符 |
| 模型要求 | Sonnet 4.0+ / Opus 4.0+（不支持 Haiku） |
| ZDR 兼容 | 服务端版本不兼容 ZDR；自定义实现兼容 |

> **什么时候用 Tool Search？** 10+ 工具时值得考虑，30+ 工具时强烈推荐。如果你的工具只有 5-8 个且每个都常用，传统方式（全部传入）更简单高效。

### 2. Programmatic Tool Calling：让代码代替模型调工具

传统 Tool Use 中，每次工具调用都需要一个完整的 API 往返——Claude 决定调用 → 你执行 → 返回结果 → Claude 再决定下一步。如果需要调 10 个工具，就是 10 次往返。

Programmatic Tool Calling 换了一种思路：**Claude 写一段代码，在代码中像调函数一样调用工具**，只需要一次代码执行就能完成所有工具调用。

**直觉建立**：

传统方式像打电话——你和 Claude 每次只能说一句话然后等对方回应。Programmatic Calling 像发邮件附带一个脚本——"帮我执行这个脚本，里面需要调的 API 都替我调了"。脚本跑完，一次性拿到所有结果。

**工作原理**：

```
1. Claude 写 Python 代码，把你的工具当作 async 函数调用
2. 代码在 Code Execution 沙箱中运行
3. 运行到工具调用时，沙箱暂停 → API 返回 tool_use（你执行工具）
4. 你返回 tool_result → 沙箱继续运行
5. 所有工具调用完成后，代码的最终输出（stdout）返回给 Claude
```

关键区别：**工具结果不进入 Claude 的上下文窗口**——只有代码的最终 print 输出进入。这意味着 Claude 可以在代码中过滤、聚合、转换大量工具返回的原始数据，只把摘要交给自己。

**核心字段——`allowed_callers`**：

```python
tools = [
    {"type": "code_execution_20260120", "name": "code_execution"},
    {
        "name": "query_database",
        "description": "Execute a SQL query. Returns JSON array of row objects.",
        "input_schema": {...},
        "allowed_callers": ["code_execution_20260120"]
        # 只能从代码中调用，不能 Claude 直接调用
    }
]
```

`allowed_callers` 的三种设置：

| 值 | 含义 | 适用场景 |
|----|------|---------|
| `["direct"]` | 只能 Claude 直接调用（默认） | 简单工具、需要 Claude 推理的场景 |
| `["code_execution_20260120"]` | 只能从代码中调用 | 批量处理、数据过滤、循环调用 |
| `["direct", "code_execution_20260120"]` | 两种方式都可以 | 灵活但可能让 Claude 犹豫选哪种 |

> **建议二选一**：对每个工具明确选择 `direct` 或 `code_execution`，不要同时开启——给 Claude 清晰的引导。

**完整交互流程**：

```
你 → API:  用户提问 + code_execution + query_database(allowed_callers: code_execution)

API → 你:  Claude 写代码 + 代码中调用 query_database
          stop_reason: "tool_use"
          caller: {type: "code_execution_20260120", tool_id: "srvtoolu_abc"}
          container: {id: "container_xyz", expires_at: "..."}

你 → API:  tool_result（你执行 query_database 的结果）
          container: "container_xyz"  // 复用容器让代码继续

API → 你:  代码继续运行 → 可能再调工具（重复上面）→ 最终输出
          code_execution_result: {stdout: "分析完成，Top 5 客户...", return_code: 0}
          Claude 的文字总结
```

**核心优势——Token 效率**：

```python
# Claude 在代码中写的逻辑（在沙箱中执行）
regions = ["West", "East", "Central", "North", "South"]
results = {}
for region in regions:
    # 每次调用 query_database，你执行并返回结果
    # 但这些结果不进入 Claude 的上下文！
    data = await query_database(f"SELECT sum(revenue) FROM sales WHERE region='{region}'")
    results[region] = data[0]["sum"]

# 只有这个 print 的内容进入 Claude 上下文
top = max(results.items(), key=lambda x: x[1])
print(f"Top region: {top[0]} with ${top[1]:,}")
```

5 次工具调用，但 Claude 只看到一行汇总。传统方式需要 5 轮 API 往返 + 所有原始数据进入上下文。

**高级模式**：

```python
# 提前终止——找到即停
endpoints = ["us-east", "eu-west", "apac"]
for ep in endpoints:
    status = await check_health(ep)
    if status == "healthy":
        print(f"Found healthy: {ep}")
        break  # 不检查剩余的

# 条件分支——根据中间结果选工具
file_info = await get_file_info(path)
if file_info["size"] < 10000:
    content = await read_full_file(path)
else:
    content = await read_file_summary(path)

# 数据过滤——大结果集中只取需要的
logs = await fetch_logs(server_id)
errors = [log for log in logs if "ERROR" in log]
print(f"Found {len(errors)} errors")
for error in errors[-10:]:  # 只返回最后 10 条
    print(error)
```

**响应中的 `caller` 字段**——区分直接调用和程序化调用：

```json
// 传统直接调用
{"type": "tool_use", "name": "query_database", "caller": {"type": "direct"}}

// 程序化调用（从代码中发起）
{"type": "tool_use", "name": "query_database",
 "caller": {"type": "code_execution_20260120", "tool_id": "srvtoolu_abc123"}}
```

**消息格式限制**——回复程序化调用的 tool_result 时：

```json
// ❌ 不能包含 text——代码在等结果，不是在等用户输入
{"role": "user", "content": [
  {"type": "tool_result", "tool_use_id": "toolu_01", "content": "..."},
  {"type": "text", "text": "What next?"}
]}

// ✅ 只能包含 tool_result
{"role": "user", "content": [
  {"type": "tool_result", "tool_use_id": "toolu_01", "content": "..."}
]}
```

这个限制只在程序化调用时存在——代码执行被暂停了，它只在等工具结果，不需要其他输入。

**Container 超时**——沙箱等你返回 tool_result 的耐心有限：

```json
"container": {"id": "container_xyz", "expires_at": "2025-01-15T14:30:00Z"}
```

约 4.5 分钟不活动就过期。过期后代码收到 `TimeoutError`，Claude 可能重试。监控 `expires_at` 字段，确保工具执行够快。

**不兼容项**：

| 特性 | 与 Programmatic Calling 兼容？ |
|------|-------------------------------|
| `strict: true`（Structured Output） | 不兼容 |
| `tool_choice` 强制指定 | 不兼容 |
| `disable_parallel_tool_use` | 不兼容 |
| MCP Connector 工具 | 暂不支持 |

> **什么时候用？** 3+ 次连续工具调用、需要过滤大量返回数据、批量操作（查 50 个端点）——Programmatic Calling 优势明显。单次工具调用、需要用户中间反馈——传统直接调用更合适。

### 3. Fine-grained Tool Streaming：降低工具调用延迟

普通 Streaming 模式下，Claude 的工具调用参数会先**完整生成并做 JSON 验证**，然后才开始传给你。如果参数很大（比如一首长诗的内容），你可能要等 15 秒才能收到第一个字节。

Fine-grained Tool Streaming 去掉了缓冲和验证——参数**生成即传输**：

```
普通 Streaming（15 秒延迟）：
  等待... 等待... 等待...
  Chunk 1: '{"'
  Chunk 2: 'query": "Ty'
  Chunk 3: 'peScri'

Fine-grained Streaming（3 秒延迟）：
  Chunk 1: '{"query": "TypeScript 5.0 5.1 5.2 5.3'
  Chunk 2: ' new features comparison'
```

**启用方式**——在工具定义上加一个字段：

```python
tools=[{
    "name": "make_file",
    "description": "Write text to a file",
    "eager_input_streaming": True,  # 就这一行
    "input_schema": {
        "type": "object",
        "properties": {
            "filename": {"type": "string"},
            "lines_of_text": {"type": "array", "items": {"type": "string"}}
        },
        "required": ["filename", "lines_of_text"]
    }
}]
```

**代价——可能收到无效 JSON**：

因为跳过了 JSON 验证，如果 `max_tokens` 截断发生在参数中间，你收到的是**不完整的 JSON**。必须在代码中处理这种情况：

```python
try:
    tool_input = json.loads(partial_json)
except json.JSONDecodeError:
    # 把无效 JSON 包装后返回给 Claude
    error_response = {"INVALID_JSON": partial_json}
```

**适用场景**：

| 场景 | 用不用 |
|------|--------|
| 工具参数很大（长文本、大数组） | 用——延迟从秒级降到亚秒级 |
| 参数很小（一个 URL、一个数字） | 不用——没有可感知的差异 |
| 需要保证 JSON 100% 有效 | 不用——优先保证正确性 |
| 实时展示工具调用进度 | 用——用户可以看到参数逐步生成 |

特性说明：GA（正式版），所有模型、所有平台支持，无需 beta header。

### 4. Files API：上传一次，多次引用

传统做法中，每次 API 请求要用文件时都要把文件内容编码后放在消息体里。Files API 提供了一种**上传一次、用 ID 引用**的模式——上传文件拿到 `file_id`，后续请求用 `file_id` 引用即可。

**核心流程**：

```
上传文件 → 拿到 file_id
         ↓
在消息中用 file_id 引用（不用重新上传）
         ↓
Claude 在 Code Execution 中生成的文件 → 用 file_id 下载
```

**上传文件**：

```python
client = anthropic.Anthropic()
uploaded = client.beta.files.upload(
    file=("report.pdf", open("report.pdf", "rb"), "application/pdf"),
)
# uploaded.id → "file_011CNha8iCJcU1wXNR6q4V8w"
```

**在消息中引用**：

```python
response = client.beta.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    betas=["files-api-2025-04-14"],
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "请总结这份文档"},
            {
                "type": "document",       # PDF/文本用 document
                "source": {
                    "type": "file",
                    "file_id": "file_011CNha8iCJcU1wXNR6q4V8w"
                }
            }
        ]
    }]
)
```

**文件类型与内容块的对应关系**：

| 文件类型 | MIME 类型 | 内容块类型 | 用途 |
|----------|----------|-----------|------|
| PDF | `application/pdf` | `document` | 文档分析 |
| 纯文本 | `text/plain` | `document` | 文本处理 |
| 图片 | `image/jpeg`, `image/png`, `image/gif`, `image/webp` | `image` | 图像分析 |
| 数据集等 | 各种 | `container_upload` | Code Execution 数据输入 |

`document` 块支持可选字段：`title`（文档标题）、`context`（背景说明）、`citations`（启用引用）。

**下载 Claude 生成的文件**：

```python
# 只能下载 Claude（通过 Code Execution 或 Skills）创建的文件
# 你自己上传的文件不能重新下载
content = client.beta.files.download("file_011CNha8iCJcU1wXNR6q4V8w")
with open("chart.png", "wb") as f:
    f.write(content)
```

**文件管理**：

```python
# 列出所有文件
files = client.beta.files.list()

# 获取单个文件元数据
meta = client.beta.files.retrieve_metadata("file_xxx")

# 删除文件（不可恢复）
client.beta.files.delete("file_xxx")
```

**存储限制**：

| 限制项 | 值 |
|--------|------|
| 单文件大小 | 500 MB |
| 组织总存储 | 100 GB |
| 文件名长度 | 1-255 字符 |
| 禁止字符 | `< > : " | ? * \ /` 和 Unicode 0-31 |
| API 速率 | ~100 请求/分钟（Beta 期间） |

**定价**——文件操作本身免费：

| 操作 | 费用 |
|------|------|
| 上传 / 下载 / 列出 / 删除 | 免费 |
| 文件内容在 Messages 中使用 | 按 input tokens 计费 |

**文件生命周期**：

- 文件属于 API Key 所在的 Workspace——同一 Workspace 下的其他 Key 可以访问
- 文件持久存在直到你主动删除
- 删除不可恢复（但可能在活跃的 Messages 请求中仍短暂可用）
- 不支持 Amazon Bedrock 和 Google Vertex AI

**与 Code Execution 的协同**——Files API 最强大的场景：

```
1. 上传 CSV 数据集 → file_id
2. 在消息中用 container_upload 类型引用 → 文件进入 Code Execution 沙箱
3. Claude 用 Python 分析数据 → 生成图表
4. 图表以新 file_id 返回 → 你用 download 取回
```

这个流程把 Files API 和 Code Execution 串成了完整的"数据进 → 分析 → 结果出"管道。

> **Beta 状态**：Files API 需要 beta header（`files-api-2025-04-14`），不受 ZDR 覆盖。功能已稳定但 API 可能变化。

---

## 重点标记

1. **Tool Search 是工具规模化的关键**：`defer_loading: true` 让工具"按需加载"，保持 3-5 个常用工具始终可见，其余全部 defer。上限 10,000 个工具
2. **Programmatic Tool Calling 的核心优势是 Token 效率**：工具返回的原始数据**不进入 Claude 上下文**——Claude 在代码中过滤/聚合后，只把摘要 print 出来。10 次工具调用的 Token 开销≈1 次
3. **`allowed_callers` 对每个工具二选一**：`["direct"]` 或 `["code_execution_20260120"]`，不要同时开启。清晰的引导让 Claude 做更好的决策
4. **Fine-grained Streaming 用 `eager_input_streaming: true`**：大参数场景延迟从 15 秒降到 3 秒，但可能收到无效 JSON——你的代码必须处理这种情况
5. **Files API 是"上传一次、引用多次"**：PDF/图片/数据集上传后用 `file_id` 引用，避免每次请求重新编码。文件操作免费，内容使用按 token 计费
6. **Programmatic Calling 的消息格式有特殊限制**：回复程序化工具调用时，user 消息只能包含 `tool_result`，不能有 text——因为代码在等结果不是在等对话
7. **三个功能都与 Code Execution 深度耦合**：Programmatic Calling 依赖 Code Execution 运行代码，Files API 配合 Code Execution 实现数据管道，Tool Search 可与 Code Execution 的 Dynamic Filtering 协同

---

## 自测：你真的理解了吗？

**Q1**：你的系统有 200 个 MCP 工具（来自 Slack、GitHub、Jira 三个 MCP Server），每次对话只用到 2-3 个。你用传统方式全部传入，发现每次请求消耗 40K+ input tokens。用 Tool Search 怎么优化？`defer_loading` 和 `mcp_toolset` 分别起什么作用？

**Q2**：你需要 Claude 检查 50 个服务器的健康状态并汇总。传统 Tool Use 和 Programmatic Tool Calling 各需要多少轮 API 往返？Token 消耗有什么差异？如果第 3 个服务器检查时 Container 过期了怎么办？

**Q3**：你在 Tool Search 中把所有工具都标了 `defer_loading: true`（包括 Tool Search 自身）。会怎样？修复后，Claude 搜索 "weather" 但没找到你的 `get_current_temperature` 工具。最可能的原因和解决办法是什么？

**Q4**：你的应用需要 Claude 分析一个 200MB 的 CSV 文件。你有三种方案：A) 文件内容直接放在消息体里，B) 通过 Files API 上传后用 `document` 块引用，C) 通过 Files API 上传后用 `container_upload` 块给 Code Execution。哪种方案可行？为什么？

**Q5**：Claude 在 Programmatic Calling 中写了代码调用 `query_database` 3 次。第 2 次调用时，你返回 tool_result 的同时附带了一条 text："这是第二次查询结果"。会发生什么？正确的做法是什么？
