# 模块 04：Agent Flows 无代码编排

> 对应源文件：agent-flows/overview, agent-flows/getting-started, agent-flows/tutorial-hackernews, agent-flows/debugging-flows, agent-flows/blocks/*

---

## 概念地图

- **核心概念** (必须内化): Flow = 无代码版自定义 Agent 技能、Block 是最小执行单元、变量是 Block 间的数据传递机制
- **实操要点** (动手时需要): Flow 三默认块（Info + Variables + Complete）、5 种功能 Block 的配置、变量插值语法 `${varName}`
- **背景知识** (扩展理解): Flow vs 代码版 Agent Skill 的取舍、JSON 遍历语法、调试技巧

---

## 概念讲解

### 1. Agent Flow 是什么

**定义**：Agent Flow 是 AnythingLLM 提供的**无代码**可视化编排工具——让你不写任何代码，通过拖拽"积木块"（Block）的方式，构建自定义的 Agent 技能。

**直觉建立**：

如果模块 03 中的自定义 Agent 插件是"写代码造工具"，那 Agent Flow 就是"搭积木造工具"。最终效果一样——都是给 Agent 增加新技能，但构建方式完全不同：

```
自定义插件（代码方式）：
  开发者写 JavaScript → plugin.json + handler.js → Agent 调用

Agent Flow（无代码方式）：
  任何人拖拽 Block → 配置参数 → 连接起来 → Agent 调用
```

**使用方式完全相同**：Flow 创建完成后，Agent 以完全相同的方式使用它——通过 `@agent` 触发，LLM 根据 Flow 的名称和描述判断何时调用。甚至，强大的 LLM 可以**自动串联多个 Flow** 来完成一个复杂任务。

> **Flow vs Agent Skill 选型**：不会写代码 → 用 Flow。需要复杂逻辑（循环、条件分支、数据库操作）→ 用代码版 Skill。两者可以共存。

---

### 2. Flow 的三个默认块

**定义**：每个新建的 Flow 自动包含三个不可删除的默认块，它们构成了 Flow 的骨架。

#### 2.1 Flow Information Block（流程信息块）

**作用**：定义 Flow 的名称和描述——这是 LLM 判断"何时使用这个 Flow"的关键依据。

| 字段 | 说明 | 重要程度 |
|------|------|---------|
| **Name** | Flow 的名称，简短描述性 | 高——LLM 靠名称做初步匹配 |
| **Description** | 详细描述用途、使用方式、变量说明、示例 | **极高**——直接影响 LLM 调用准确性 |

> **最佳实践**：Description 应包含：(1) 这个 Flow 做什么 (2) 使用示例 (3) 变量的可选值说明。信息越丰富，LLM 判断越准确。

#### 2.2 Flow Variables Block（流程变量块）

**作用**：定义 Flow 中所有 Block 共享的变量——变量是 Block 之间传递数据的管道。

| 字段 | 说明 |
|------|------|
| **Variable Name** | 变量名（如 `pageContent`），在其他 Block 中用 `${pageContent}` 引用 |
| **Default Value** | 默认值（可留空）。如果 LLM 在调用时没有提供值，就用默认值 |

**变量的工作方式**：

```
Flow Variables 定义 → Block A 将结果写入变量 → Block B 读取变量作为输入
```

变量就像是 Block 之间的"传话纸条"——Block A 把结果写在纸条上，Block B 拿起纸条读内容。

**JSON 对象遍历**：当变量值是 JSON 对象时，可以用点号和数组索引访问嵌套字段：

```
变量 apiResponse 的值：
{
  "data": {
    "users": [{ "name": "John", "details": { "city": "New York" } }]
  }
}

使用方式：
${apiResponse.data.users[0].name}       → "John"
${apiResponse.data.users[0].details.city} → "New York"
```

#### 2.3 Flow Complete Block（流程完成块）

**作用**：标记 Flow 的结束点。纯视觉标识，不执行任何逻辑。

---

### 3. 五种功能 Block

**定义**：除三个默认块外，AnythingLLM 提供 5 种可添加的功能 Block，每种执行一种特定操作。

| Block | 功能 | 输入 | 输出 | 可用版本 |
|-------|------|------|------|---------|
| **Web Scraper** | 爬取网页文本内容 | URL | 页面文本（非 HTML） | 全版本 |
| **API Call** | 调用任意 HTTP API | URL、Method、Headers、Body | API 响应 | 全版本 |
| **LLM Instruction** | 向 LLM 发送指令处理数据 | 指令文本（可含变量） | LLM 回复 | 全版本 |
| **Read File** | 读取本地文件内容 | 文件路径 | 文件文本 | **Desktop ≥ 1.8.1** |
| **Write File** | 将内容写入本地文件 | 文件路径、内容 | 无 | **Desktop ≥ 1.8.1** |

**逐一详解**：

#### Web Scraper Block

- 输入：`URL to scrape`（支持 `${variable}` 动态 URL）
- 输出：`Result Variable`（存储爬取到的纯文本）
- 返回的是**纯文本**而非 HTML。如需 HTML，用 API Call Block

#### API Call Block

- 输入：`URL`、`Method`（GET/POST 等）、`Headers`、`Body`
- 输出：`Result Variable`（存储 API 响应）
- **所有字段都支持变量插值**——可以动态构造请求

POST Body 中使用变量的示例：
```json
{
  "query": "${userQuery}",
  "limit": 10,
  "${dynamicKey}": "staticValue"
}
```

#### LLM Instruction Block

- 输入：`Instructions`（发给 LLM 的指令，支持变量）
- 输出：`Result Variable`（存储 LLM 回复）
- **使用当前 Workspace 的 Agent LLM**
- 这是最灵活的 Block——可以用 LLM 做数据解析、内容过滤、格式转换等任何文本处理

> **注意**：LLM 的遵循能力取决于模型本身。3B 参数的模型和 GPT-4 的效果天差地别。

#### Read File / Write File Block

- 仅在 **Desktop 版 ≥ 1.8.1** 可用
- 支持变量动态路径：`/path/to/${fileName}.txt`
- 只支持文本文件

---

### 4. 完整教程：HackerNews 文章过滤器

**目标**：创建一个 Flow，爬取 HackerNews 页面，用 LLM 过滤出你感兴趣的话题文章。

**成品效果**：

```
用户：@agent Find AI-related posts on HackerNews
Agent：[调用 Flow] → 爬取页面 → LLM 过滤 → 返回相关文章链接列表
```

#### Step 1：创建 Flow

Workspace Agent Skills 页面 → 点击 "Create Flow"

#### Step 2：配置 Flow Information

- **Name**：`Hacker News Headline Viewer`
- **Description**：
```
This tool can be used to visit hacker news webpage and extract ALL
headlines and links from the page that have to do with a particular topic.

Available options for `page`:
(empty) - front page
"newest" - newest posts page

Examples of how to use this flow:
"Find AI-related posts on HackerNews"
"Show me political discussions from the newest HackerNews posts"

The flow will return relevant articles as clickable markdown links.
```

> **为什么 Description 这么详细**：LLM 靠 Description 判断何时调用这个 Flow、如何传参。示例和变量说明越清楚，调用越准确。

#### Step 3：配置 Flow Variables

| 变量名 | 默认值 | 用途 |
|--------|--------|------|
| `hackerNewsURLPath` | （空） | 控制爬取首页还是最新页 |
| `topicOfInterest` | `Political discussions or items` | 过滤话题 |
| `pageContentFromSite` | （空） | 存储爬取的页面内容 |

#### Step 4：添加 Web Scraper Block

- **URL**：`https://news.ycombinator.com/${hackerNewsURLPath}`
- **Result Variable**：`pageContentFromSite`

> `${hackerNewsURLPath}` 实现了动态 URL——LLM 传入 "newest" 就爬最新页，传空就爬首页。

#### Step 5：添加 LLM Instruction Block

- **Instructions**：
```
Extract all links from this content that would be relevant to this topic:
${topicOfInterest}

Content:
${pageContentFromSite}

Format your response as a list of markdown links, with a brief description
of why each link is relevant.
If no relevant links are found, say "No relevant articles found."
```
- **Result Variable**：（空——直接作为最终输出）

#### Step 6：保存并测试

1. 点击 "Save"
2. **禁用其他 Agent 技能**（确保只有这个 Flow 可用，方便调试）
3. 测试 Prompt：
   - `@agent Find AI-related posts on HackerNews`
   - `@agent Show me political discussions from the newest HackerNews posts`
   - `@agent What are the latest cryptocurrency articles on HackerNews?`

**数据流总结**：

```
用户问题 → LLM 解析出 hackerNewsURLPath 和 topicOfInterest
    ↓
Web Scraper: 爬取 https://news.ycombinator.com/{path}
    ↓ 结果存入 pageContentFromSite
LLM Instruction: 从 pageContentFromSite 中提取与 topicOfInterest 相关的链接
    ↓
返回 Markdown 格式的文章列表
```

---

### 5. 调试与排错

**定义**：Flow 的调试核心原则是**隔离测试**——确保每次只有一个 Flow 可用。

**调试方法**：

| 方法 | 具体操作 |
|------|---------|
| **隔离测试** | 禁用所有其他 Agent 技能，只留被测 Flow |
| **查看日志** | Desktop：打开日志文件查看详细执行过程；Docker：查看容器日志 |
| **检查变量** | 确认变量名拼写正确，特别是 `${varName}` 中的大小写 |
| **验证 URL** | Web Scraper 的 URL 是否正确拼接，变量是否正确插入 |
| **换模型** | 如果 LLM Instruction 结果不理想，尝试更强的模型 |

**常见问题**：

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| Flow 不被触发 | Description 不够清晰，LLM 不知道何时用 | 丰富 Description，加入示例 |
| Web Scraper 返回空 | URL 拼接错误 | 检查 `${variable}` 语法 |
| LLM Instruction 结果差 | 模型能力不足或指令模糊 | 换更强模型 / 改写指令 |
| Block 缺失 | 版本不对 | Read File / Write File 仅 Desktop ≥ 1.8.1 |

---

## 重点标记

1. **Flow 是无代码版的 Agent Skill**：最终效果与代码版插件完全相同，只是构建方式不同
2. **Description 决定 Flow 是否被正确调用**：LLM 完全依赖 Description 判断何时使用 Flow，写得越详细越好
3. **变量是 Block 间的数据桥梁**：`${varName}` 语法在所有 Block 的所有字段中通用
4. **调试时隔离测试**：禁用其他技能，确保 LLM 只能调用被测 Flow
5. **Read File / Write File 仅限 Desktop**：Docker 版目前不支持这两个 Block

---

## 自测：你真的理解了吗？

**Q1**：你想创建一个 Flow，功能是"输入一个 GitHub 仓库 URL，爬取 README，用 LLM 总结项目功能"。你需要哪些 Block？变量怎么定义？

**Q2**：一个 Flow 的 Description 只写了 "This flow gets news"。可能导致什么问题？你会怎么改进？

**Q3**：Agent Flow 和自定义 Agent 插件（handler.js）各自适合什么场景？如果你需要一个调用第三方 API 并解析 JSON 响应的工具，用哪种方式更合适？为什么？

**Q4**：你在调试一个 Flow，发现 Agent 总是调用其他技能而不是你的 Flow。可能的原因是什么？你会怎么排查？

**Q5**：一个 API Call Block 返回了 JSON 数据 `{"results": [{"title": "Hello"}, {"title": "World"}]}`，存在变量 `apiData` 中。你如何在后续 LLM Instruction Block 中引用第二个 title 的值？
