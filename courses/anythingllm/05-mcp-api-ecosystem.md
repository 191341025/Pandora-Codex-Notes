# 模块 05：MCP 协议、API 与生态扩展

> 对应源文件：mcp-compatibility/overview+desktop+docker, features/api, features/chat-widgets, features/security-and-access, features/privacy-and-data-handling

---

## 概念地图

- **核心概念** (必须内化): MCP 是标准化的工具接入协议、三种传输方式（StdIO/SSE/Streamable）、三级用户角色（Admin/Manager/Default）
- **实操要点** (动手时需要): MCP 配置文件格式、嵌入式聊天组件的域名限制和速率控制、API Key 管理
- **背景知识** (扩展理解): MCP 生态现状、隐私与遥测、单人模式 vs 多人模式的不可逆性

---

## 概念讲解

### 1. MCP 协议：标准化的工具接入

**定义**：MCP（Model Context Protocol，模型上下文协议）是 Anthropic 开源的协议标准，定义了 LLM 应用如何与外部工具和数据源对接的统一方式。AnythingLLM 支持任何兼容 MCP 的 Server。

**直觉建立**：

如果模块 03 的自定义 Agent 插件是"自己造工具"，模块 04 的 Agent Flow 是"搭积木造工具"，那 MCP 就是"买现成的工具直接插上用"。

MCP 定义了一个**通用插座标准**——只要第三方工具按 MCP 规范开发，就能直接插到 AnythingLLM 上。已经有大量现成的 MCP Server 可用（GitHub 操作、数据库查询、YouTube 内容获取等）。

> **AnythingLLM 的 MCP 支持范围**：目前仅支持 **Tools**（工具调用）。不支持 Resources、Prompts、Sampling 等 MCP 的其他能力。

---

### 2. MCP 配置

**配置文件**：`STORAGE_DIR/plugins/anythingllm_mcp_servers.json`

文件在你首次打开 "Agent Skills" 页面时自动创建（如果不存在）。

**配置格式**：

```json
{
  "mcpServers": {
    "mcp-youtube": {
      "command": "uvx",
      "args": ["mcp-youtube"]
    },
    "postgres-http": {
      "type": "streamable",
      "url": "http://localhost:3003",
      "headers": {
        "X-API-KEY": "api-key"
      }
    }
  }
}
```

#### 三种传输方式

| 传输方式 | 配置方式 | 特点 |
|----------|---------|------|
| **StdIO**（默认） | 设置 `command` + `args` | 最简单，本地进程通信 |
| **SSE** | 设置 `type: "sse"` + `url` | 远程服务，流式响应 |
| **Streamable** | 设置 `type: "streamable"` + `url` | 类似 SSE，更新的协议 |

**StdIO 示例**（最常用）：
```json
{
  "face-generator": {
    "command": "npx",
    "args": ["@dasheck0/face-generator"],
    "env": {
      "MY_API_KEY": "sk-xxx"
    }
  }
}
```

**远程服务示例**：
```json
{
  "postgres-http": {
    "type": "streamable",
    "url": "http://localhost:3003",
    "headers": {
      "X-API-KEY": "api-key"
    }
  }
}
```

#### 命令可用性

**Desktop**：`command` 必须是你本机已安装且在 PATH 中可用的命令（如 `npx`、`uvx`、`node`）。AnythingLLM 不会自动安装。

**Docker**：容器预装了 `npx`、`uv`/`uvx`、`node`、`bash`，大多数 MCP Server 可以直接使用。

#### 自动启动控制

默认情况下，所有 MCP Server 在你打开 Agent Skills 页面或触发 `@agent` 时自动启动。如果某个 Server 资源消耗大，可以关闭自动启动：

```json
{
  "face-generator": {
    "command": "npx",
    "args": ["@dasheck0/face-generator"],
    "anythingllm": {
      "autoStart": false
    }
  }
}
```

> `anythingllm.autoStart` 是 AnythingLLM 专有属性，对其他 MCP 客户端无效。

#### 管理操作

所有操作都可以在 Agent Skills 页面的 MCP 管理 UI 中完成：

| 操作 | 方式 |
|------|------|
| 添加 Server | 编辑 JSON 配置文件，在 UI 点 "Refresh" |
| 删除 Server | UI 点齿轮图标 → "Delete"（同时停止进程并从配置文件移除） |
| 启停 Server | UI 点齿轮图标 → "Start" / "Stop" |
| 查看状态/错误 | UI 点击某个 Server 查看详情 |
| 热重载 | UI 点 "Refresh"（无需重启应用/容器） |

#### Desktop vs Docker 差异

| 维度 | Desktop | Docker |
|------|---------|--------|
| 命令 | 需自行安装 | 预装常用命令 |
| 文件路径 | 直接使用宿主机路径 | 必须使用 `/app/server/storage/...` |
| 工具持久化 | 下载的库持久保存在宿主机 | 容器删除后库丢失，需重新下载 |
| 调试 | 查看 Desktop 应用日志 | 查看 Docker 容器日志 |

> **安全警告**：MCP Server 可以执行任意代码。**永远不要运行你不信任的 MCP Server**。

---

### 3. 开发者 API

**定义**：AnythingLLM 提供完整的 REST API，可以用代码管理 Workspace、上传文档、发起对话——一切 UI 能做的事情，API 都能做。

**API 文档位置**：你的实例的 `/api/docs` 路径（如 `http://localhost:3001/api/docs`）

**核心能力**：
- 管理 Workspace（创建、删除、配置）
- 上传和嵌入文档
- 发起对话（Chat / Query 模式）
- 管理用户（多用户模式下）

**API Key 管理**：
- 在设置页面创建/删除 API Key
- 需要相应权限级别才能生成 Key
- **不要公开发布 API Key**——任何持有 Key 的人都能完整操作你的实例

> **适用场景**：自动化工作流（如定时上传文档）、集成到其他系统（如 CRM 自动查询知识库）、开发自定义前端。

---

### 4. 嵌入式聊天组件（Chat Widget）

**定义**：AnythingLLM（Docker 版专属）允许你生成一段 `<script>` 代码，嵌入到任何网站中，创建一个客服/问答聊天窗口。

**配置选项**：

| 选项 | 说明 |
|------|------|
| **Workspace** | 关联的 Workspace（决定文档和默认配置） |
| **Chat Method** | Chat（开放回答）或 Query（仅基于文档） |
| **Domain Restriction** | 限制哪些域名可以使用此组件（防止被盗用） |
| **Max Chats per Day** | 每日最大对话数（全局限制），0 = 无限 |
| **Max Chats per Session** | 每个用户会话的最大对话数，0 = 无限 |
| **Dynamic Model** | 允许覆盖 Workspace 默认的 LLM 模型 |
| **Dynamic Temperature** | 允许覆盖 LLM 温度参数 |
| **Prompt Override** | 允许覆盖 System Prompt |

**嵌入方式**：配置完成后，系统生成一段 `<script>` 标签，粘贴到你的网站 HTML 中即可。

> **关键安全措施**：
> - **必须设置域名限制**——否则任何人复制你的嵌入代码就能在自己的网站上使用你的 AI（消耗你的 Token）
> - **设置每日对话上限**——防止被恶意刷量

---

### 5. 安全与权限管理

**定义**：AnythingLLM Docker 版支持两种安全模式——单人模式和多人模式。

#### 单人模式

- 默认模式
- 可选设置实例密码
- 所有登录用户权限相同（完全控制）
- 适合个人或少数可信人员使用

#### 多人模式

> **不可逆**：一旦启用多人模式，无法回退到单人模式。

**三级角色**：

| 角色 | 权限范围 |
|------|---------|
| **Admin** | 完全控制——系统设置、LLM/Embedder/VectorDB 配置、用户管理、日志、分析 |
| **Manager** | 管理所有 Workspace 和文档，但**不能**修改 LLM、Embedder、VectorDB 的系统级配置 |
| **Default** | 只能在被明确授权的 Workspace 中对话，不能查看或编辑 Workspace 设置 |

**启用方式**：设置页面 → 启用多人模式 → 创建第一个 Admin 账户 → 系统重启后用新账户登录。

**权限设计逻辑**：

```
Admin：什么都能做
  ↓ 不能做：修改系统级配置（LLM/Embedder/VectorDB）
Manager：管理内容和 Workspace
  ↓ 不能做：看到或操作未授权的 Workspace
Default：只聊天
```

---

### 6. 隐私与数据处理

**定义**：AnythingLLM 在隐私方面的核心承诺是**透明**——明确告诉你数据流向哪里。

**匿名遥测**：
- AnythingLLM 默认收集匿名使用数据（不包含个人信息）
- 可以在设置中关闭
- 用于产品改进

**数据流向透明度**：

AnythingLLM 会在隐私设置页面明确展示：
- 你的 LLM Provider 是谁（数据是否离开本机）
- 你的 Embedding Provider 是谁
- 你的 VectorDB 是否是本地的

> **最大化隐私的配置**：LLM 用 Ollama（本地）+ Embedding 用 Built-in（本地）+ VectorDB 用 LanceDB（本地）→ 数据完全不离开你的机器。

---

## 重点标记

1. **MCP 是"买现成工具直接插上"**：配置一个 JSON 文件就能接入大量第三方工具，不需要写代码
2. **MCP 只支持 Tools**：Resources、Prompts、Sampling 等其他 MCP 能力暂不支持
3. **嵌入式聊天组件必须设域名限制和对话上限**：否则有被滥用和刷量的风险
4. **多人模式不可逆**：启用前确认你确实需要，否则只能重新安装
5. **API 文档在 `/api/docs`**：这是开发集成时最重要的参考，不需要看外部文档

---

## 自测：你真的理解了吗？

**Q1**：你想让 Agent 能够操作 GitHub（创建 Issue、查看 PR）。你会选择：(a) 写自定义 Agent 插件 (b) 创建 Agent Flow (c) 接入 MCP Server？为什么？

**Q2**：你的 AnythingLLM Docker 实例需要接入一个远程 MCP Server（运行在另一台机器的 3003 端口），使用 SSE 传输。写出配置片段。

**Q3**：你要在公司官网嵌入一个 AnythingLLM 聊天组件，基于产品文档回答客户问题。在安全方面你至少要做哪三件事？

**Q4**：公司有 3 个部门使用 AnythingLLM：IT 部门需要完全控制，HR 部门需要管理自己的 Workspace，实习生只需要查文档。你会如何分配角色？

**Q5**：一个同事说"我用了 OpenAI 的 LLM 和 Embedding，但我配了本地 LanceDB，所以我的数据完全在本地"。这个说法对吗？哪些数据离开了本机？
