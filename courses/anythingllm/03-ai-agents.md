# 模块 03：AI Agents 实战

> 对应源文件：agent/overview, agent/setup, agent/usage, features/ai-agents, agent/custom/introduction+developer-guide+plugin-json+handler-js

---

## 概念地图

- **核心概念** (必须内化): Agent = LLM + 工具调用、Agent LLM 独立于 System/Workspace LLM、工具调用依赖模型能力
- **实操要点** (动手时需要): `@agent` 触发与 `/exit` 退出、8 种内置工具的使用场景、自定义插件的文件结构
- **背景知识** (扩展理解): Few-shot prompting 在工具调用中的作用、模型量化等级与 Agent 能力的关系

---

## 概念讲解

### 1. Agent 是什么

**定义**：Agent 是一个配备了"工具"的 LLM——不仅能对话，还能**执行操作**：搜索网页、爬取网站、生成图表、查询数据库、保存文件。

**直觉建立**：

普通 LLM 是一个**只能说话的顾问**——你问什么它答什么，但它不能帮你做事。Agent 是一个**有手的顾问**——它不仅能回答问题，还能帮你上网查资料、从数据库里拉数据、把结果存成文件。

> **类比边界**：Agent 的"手"（工具）是有限的——它只能使用你启用的工具。它不能自己发明新工具，也不能执行超出工具定义范围的操作。

**Agent 与普通对话的核心区别**：

| 维度 | 普通对话 | Agent 会话 |
|------|---------|-----------|
| LLM 来源 | System LLM 或 Workspace LLM | **Agent LLM**（独立配置） |
| 能力 | 仅对话生成 | 对话 + 工具调用 |
| 触发方式 | 直接输入 | 以 `@agent` 开头 |
| 退出方式 | 无需退出 | 输入 `/exit` 或 `exit` |
| 适合场景 | 问答、写作、翻译 | 搜索、数据分析、自动化任务 |

---

### 2. Agent LLM：独立的第三层配置

**定义**：Agent 使用独立配置的 LLM，与 System LLM 和 Workspace LLM 分开。这是模块 01 讲过的三层配置中的第三层。

**为什么需要独立配置**：

Agent 需要 LLM 具备**工具调用（Tool Calling / Function Calling）**能力——理解工具的定义、判断何时调用哪个工具、按正确格式传递参数。**不是所有 LLM 都能胜任这个任务**。

| LLM 特征 | 对 Agent 的影响 |
|----------|----------------|
| 模型大小 | 越大越好——7B 参数以下的模型作为 Agent 效果很差 |
| 量化等级 | 同一模型，8-Bit 量化明显优于 4-Bit 量化 |
| 工具调用支持 | GPT-4o、Claude 等原生支持 Function Calling 的模型最佳 |
| 指令跟随能力 | Agent 需要精确遵循复杂指令，弱模型容易"幻觉调用" |

**配置路径**：Workspace 设置 → Agent Configuration → 选择 LLM Provider 和 Model → 点击 "Update workspace agent"

> **常见问题**："Agent 说它无法访问网络"——这通常不是网络问题，而是你使用的 LLM 不擅长工具调用。尝试使用更大的模型或更高量化等级的模型。

---

### 3. 八种内置工具详解

**定义**：AnythingLLM 的 Agent 自带 8 种工具，其中 3 种是默认启用且不可禁用的核心工具，其余 5 种可按需开关。

#### 默认工具（不可禁用）

| 工具 | 功能 | 使用示例 |
|------|------|---------|
| **RAG Search** | 在 Workspace 已嵌入的文档中搜索信息 | `@agent can you check what you already know about AnythingLLM?` |
| **Summarize Documents** | 对特定文档生成摘要 | `@agent can you summarize readme.pdf` |
| **Scrape Websites** | 爬取网页内容，自动嵌入到 Workspace 并回答问题 | `@agent can you scrape anythingllm.com and summarize its features?` |

#### 可选工具

| 工具 | 功能 | 使用示例 | 前置条件 |
|------|------|---------|---------|
| **Web Browsing** | 搜索互联网获取实时信息 | `@agent search the web for "latest AI news"` | 需配置 Search Provider |
| **Save Files** | 将信息保存为文件到本地 | `@agent save this as a PDF on my desktop` | 无 |
| **List Documents** | 列出 Workspace 中所有可访问的文档 | `@agent what documents can you see?` | 无 |
| **Chart Generation** | 根据数据或公式生成图表 | `@agent plot y=10x as a chart` | 无 |
| **SQL Agent** | 连接关系数据库，执行 SQL 查询 | `@agent summarize sales for May 2024 in the backend-office DB` | 需配置数据库连接 |

**逐一详解关键工具**：

#### RAG Search

与普通 RAG 对话不同，Agent 的 RAG Search 是**主动行为**——Agent 自己决定何时去搜索文档，而不是每次对话都自动触发。此外，Agent 可以**将搜索结果保存到自己的记忆中**，供后续对话调用。

> 示例：`@agent summarize and save that summary for later to your memory`——Agent 会将摘要嵌入为一个虚拟文档，后续可以直接调用。

#### Web Browsing vs Web Scraping

这两个工具容易混淆：

| 维度 | Web Browsing | Web Scraping |
|------|-------------|--------------|
| 做什么 | 搜索引擎搜索关键词 | 爬取指定 URL 的页面内容 |
| 需要什么 | Search Provider（SerpApi、Google 等） | 无额外配置 |
| 结果处理 | 返回搜索结果摘要 | 将网页内容嵌入 Workspace |

#### SQL Agent

SQL Agent 是最强大也是最危险的内置工具。它具备四个子能力：

1. `list-databases`：查看可用数据库连接
2. `list-tables`：列出某个数据库的所有表
3. `check-table-schema`：查看表结构（列名、类型）
4. `query`：执行 SQL 查询返回结果

> **安全警告**：虽然 Agent 被指示只执行 SELECT 语句，但**没有技术手段阻止它运行 DELETE、UPDATE 等破坏性操作**。务必使用**只读数据库用户**连接。

#### Save Files 的幻觉问题

Agent 可能**声称已保存文件但实际上没有**——这是一种幻觉。判断方法：如果对话中没有出现 `@agent is attempting to call save-file-to-browser` 日志，说明 Agent 只是"说"它做了，但并没有真正调用工具。

解决方案：
- 明确要求 Agent 调用 `save-file-to-browser` 工具
- 使用工具调用能力更强的模型

---

### 4. Agent 会话的生命周期

**定义**：Agent 会话有明确的开始和结束标志，在会话期间，所有消息都走 Agent LLM。

**完整流程**：

```
1. 输入 "@agent 你的问题" → 看到 "Agent @agent invoked" 日志
2. Agent 分析问题 → 决定调用哪些工具
3. 工具执行 → 结果返回给 Agent
4. Agent 综合结果生成回答
5. 继续对话（无需再加 @agent 前缀）
6. 输入 "/exit" 或 "exit" → 看到 "Agent session completed" 日志
```

> **注意**：Agent 会话期间，你不需要每条消息都加 `@agent`——只需要在第一条消息触发，后续直接对话即可。退出后回到普通对话模式。

---

### 5. 自定义 Agent 插件开发

**定义**：AnythingLLM 允许开发者用 JavaScript（NodeJS）编写自定义 Agent 工具，扩展 Agent 的能力。从调用外部 API 到执行本地脚本，只要 NodeJS 能做的事情，插件都能做。

**前置条件**：
- NodeJS 18+ 开发经验
- AnythingLLM Desktop ≥ 1.6.5 或 Docker ≥ v1.2.2

#### 文件结构

每个自定义插件是一个文件夹，放在 `STORAGE_DIR/plugins/agent-skills/` 目录下：

```
plugins/agent-skills/my-custom-skill/
├── plugin.json      ← 插件定义（元数据、参数、示例）
├── handler.js       ← 入口文件（执行逻辑）
└── (其他辅助文件)    ← 可选，如外部模块、配置等
```

**关键约束**：
- 文件夹名**必须**与 `plugin.json` 中的 `hubId` 一致
- `handler.js` 的 `handler` 函数**必须返回字符串**
- 所有代码**必须**用 `try/catch` 包裹

#### plugin.json 核心字段

```json
{
  "active": true,
  "hubId": "open-meteo-weather-api",
  "name": "Get Weather",
  "schema": "skill-1.0.0",
  "version": "1.0.0",
  "description": "Gets weather for a given location",

  "setup_args": {
    "API_KEY": {
      "type": "string",
      "required": false,
      "input": {
        "type": "text",
        "default": "",
        "placeholder": "sk-xxx",
        "hint": "Your API key"
      }
    }
  },

  "examples": [
    {
      "prompt": "What is the weather in Tokyo?",
      "call": "{\"latitude\": 35.6895, \"longitude\": 139.6917}"
    }
  ],

  "entrypoint": {
    "file": "handler.js",
    "params": {
      "latitude": {
        "description": "Latitude of the location",
        "type": "string"
      },
      "longitude": {
        "description": "Longitude of the location",
        "type": "string"
      }
    }
  },

  "imported": true
}
```

**字段解读**：

| 字段 | 作用 | 重要程度 |
|------|------|---------|
| `hubId` | 唯一标识，必须与文件夹名一致 | 必填 |
| `name` | UI 显示名称 | 必填 |
| `setup_args` | 运行时参数（如 API Key），自动生成 UI 输入框 | 可选但推荐 |
| `examples` | Few-shot 示例，帮助 LLM 理解何时调用此工具 | 可选但**强烈推荐** |
| `entrypoint.params` | 定义 handler 函数接收的参数 | 必填 |
| `imported` | 必须设为 `true` | 必填 |

#### handler.js 核心模式

```javascript
module.exports.runtime = {
  handler: async function ({ latitude, longitude }) {
    const callerId = `${this.config.name}-v${this.config.version}`;
    try {
      // 向用户界面输出"思考过程"
      this.introspect(`${callerId} called with lat:${latitude}...`);

      // 调用外部 API
      const response = await fetch(`https://api.example.com?lat=${latitude}`);
      const data = await response.json();

      // 必须返回字符串
      return JSON.stringify(data);
    } catch (e) {
      this.introspect(`${callerId} failed: ${e.message}`);
      this.logger(`${callerId} error`, e.message);
      return `Tool failed: ${e.message}`;
    }
  },
};
```

**可用的 `this` 方法**：

| 方法 | 用途 |
|------|------|
| `this.introspect(msg)` | 在用户界面显示 Agent 的"思考过程" |
| `this.logger(msg)` | 输出到控制台（调试用） |
| `this.config` | 访问插件配置（name, hubId, version） |
| `this.runtimeArgs` | 访问 `setup_args` 中用户配置的值 |

#### 开发技巧

1. **热加载**：修改插件代码后无需重启 AnythingLLM，但需要 `/exit` 当前 Agent 会话再重新进入
2. **模块引入**：推荐在函数内部 `require`（而非全局 `require`），避免模块加载冲突
3. **示例很重要**：`examples` 字段本质上是 Few-shot Prompting——帮助 LLM 判断"用户说了什么时应该调用这个工具"
4. **只返回字符串**：返回其他类型会导致 Agent 崩溃或无限循环

> **安全警告**：自定义插件可以执行任意 NodeJS 代码——只运行你信任的插件。不要从不明来源安装未经审查的插件。

---

### 6. Search Provider 配置（Web Browsing 前置条件）

**定义**：如果你想让 Agent 使用 Web Browsing 工具，需要配置一个搜索引擎 Provider。

**支持的 Provider**：

| Provider | 特点 |
|----------|------|
| Google | 原生 Google 搜索，需创建 Programmable Search Engine |
| SerpApi | 聚合多个搜索引擎（Google、Amazon、Baidu 等） |
| SearchApi | 支持 Google、Bing、YouTube 等 |
| Serper | Google 搜索 API |
| Bing Search | 微软 Bing 搜索 |
| Serply | 搜索 API 服务 |

> **不需要 Web Browsing 就不用配**：如果你的 Agent 只需要操作本地文档和数据库，可以跳过这一步。

---

## 重点标记

1. **Agent = LLM + 工具**：Agent 不是新模型，是给 LLM 配上了"手脚"
2. **Agent LLM 必须支持工具调用**：小模型和低量化模型作为 Agent 效果很差，优先用 GPT-4o、Claude 等
3. **3 个默认工具 + 5 个可选工具**：RAG Search、Summarize、Scrape 默认启用，Web Browsing 需要配 Search Provider
4. **SQL Agent 用只读用户**：没有技术手段阻止 Agent 执行破坏性 SQL
5. **自定义插件必须返回字符串**：这是最常踩的坑——返回其他类型会导致 Agent 崩溃

---

## 自测：你真的理解了吗？

**Q1**：你的 System LLM 是 GPT-4o-mini，Workspace LLM 是 Claude Sonnet，Agent LLM 是 GPT-4o。当你在这个 Workspace 中发起 `@agent` 会话时，Agent 用的是哪个 LLM？

**Q2**：你让 Agent 搜索"最新的 AI 新闻"，但 Agent 回复"I cannot access the internet"。你的 LLM 是 Ollama 的 Llama 3 8B 4-Bit。可能的原因是什么？给出至少两个改进方案。

**Q3**：一个同事写了一个自定义 Agent 插件，handler.js 的 handler 函数返回了一个对象 `{ result: "success" }` 而不是字符串。会发生什么？怎么修复？

**Q4**：你想让 Agent 查询公司的 MySQL 数据库来回答业务问题。在安全方面，你应该注意什么？

**Q5**：Web Browsing 和 Web Scraping 有什么区别？如果你想让 Agent 获取某个特定网页的完整内容，应该用哪个？如果你想让 Agent 搜索某个话题的最新信息，应该用哪个？
