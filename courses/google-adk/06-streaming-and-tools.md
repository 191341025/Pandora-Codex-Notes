# 模块六：流式处理与开发工具

> 对应 PDF 第 58-69 页

---

## 概念讲解

### 1. Streaming Agent 概念

**定义**：ADK 的 Streaming 能力让你的 Agent 支持实时双向交互——语音对话、视频输入、实时工具调用。不是等 Agent 全部处理完再回复，而是边处理边返回。

**核心方法**：
- Python：`Runner.run_live()`
- Java：`Runner.runLive()`

**前提条件**：需要使用支持 **Live API** 的 Gemini 模型。不是所有模型都支持 Streaming。

> **注意**：截至当前版本，ADK Streaming 尚不支持 Callback、LongRunningTool、ExampleTool 和 Shell Agent（如 SequentialAgent）。

---

### 2. Python Streaming Quickstart

#### 项目结构

```
adk-streaming/
  └── app/
      ├── .env                       # API Key
      └── google_search_agent/
          ├── __init__.py
          └── agent.py               # Agent 定义
```

#### Agent 定义

```python
from google.adk.agents import Agent
from google.adk.tools import google_search  # ADK 内置工具

root_agent = Agent(
    name="basic_search_agent",
    model="...",  # 填入支持 Live API 的模型 ID
    description="Agent to answer questions using Google Search.",
    instruction="You are an expert researcher. You always stick to the facts.",
    tools=[google_search]  # 内置 Google Search 工具
)
```

> **亮点**：`google_search` 是 ADK 内置的 Grounding 工具，一行代码就能让 Agent 具备搜索能力。不需要自己写 Tool 函数。

#### 运行与测试

```bash
cd app

# 设置 SSL 证书（语音/视频测试必需）
# Linux/Mac:
export SSL_CERT_FILE=$(python -m certifi)
# Windows PowerShell:
$env:SSL_CERT_FILE = (python -m certifi)

# 启动 Dev UI
adk web
```

打开 `http://localhost:8000`，选择 `google_search_agent`。

#### 语音/视频测试

- **语音**：点击麦克风按钮，口头提问，Agent 会实时语音回复
- **视频**：点击摄像头按钮，问"What do you see?"，Agent 可以描述画面

> **注意**：原生音频模型下，**不能使用文字聊天**，在 adk web 输入文字会报错。语音和文字是互斥的。

---

### 3. Visual Builder 可视化构建

**定义**：ADK Visual Builder 是一个基于 Web 的可视化工作流设计工具，让你用拖拽界面创建和管理 Agent，不用写代码。

> 需要 ADK Python v1.18.0+，目前为实验性功能。

#### 启动

```bash
adk web --port 8000
```

> **提示**：在你的代码开发目录下运行，Visual Builder 会在当前目录创建项目子文件夹。

#### 使用流程

1. 点击左上角 **+**（加号）创建新 Agent
2. 输入 Agent 应用名称，点击 Create
3. 编辑 Agent：
   - 左面板：编辑组件属性
   - 中央面板：添加新组件（拖拽）
   - 右面板：AI 助手（用 prompt 修改 Agent 或获取帮助）
4. 点击 **Save** 保存
5. 测试 Agent
6. 点击铅笔图标继续编辑

#### 支持的组件

**Agent 类型**：

| 类型 | 说明 |
|------|------|
| Root Agent | 工作流的主控 Agent |
| LLM Agent | 由生成式 AI 模型驱动的 Agent |
| Sequential Agent | 按顺序执行一系列 sub-agent |
| Loop Agent | 重复执行 sub-agent 直到满足条件 |
| Parallel Agent | 并发执行多个 sub-agent |

**其他组件**：
- **Prebuilt Tools**：ADK 内置工具
- **Custom Tools**：自定义 Python 函数（需指定完全限定函数名）
- **Callbacks**：在事件开始/结束时修改 Agent 行为

#### 生成的代码结构

```
DiceAgent/
  root_agent.yaml          # 主 Agent 配置（YAML 格式）
  sub_agent_1.yaml         # Sub Agent 配置
  tools/
    __init__.py
    dice_tool.py           # 工具代码
```

> Visual Builder 生成的是 **Agent Config 格式**（`.yaml` 配置 + Python 工具代码），不是纯 Python Agent 代码。

---

### 4. llms.txt 与 AI 编码集成

**定义**：ADK 文档遵循 `/llms.txt` 标准——一个机器可读的文档索引，专门优化给 LLM 使用。让你的 AI 编码助手能理解和检索 ADK 文档。

**两个文件**：

| 文件 | 适用场景 | URL |
|------|----------|-----|
| `llms.txt` | 能动态抓取链接的工具 | `https://google.github.io/adk-docs/llms.txt` |
| `llms-full.txt` | 需要完整文本 dump 的工具 | `https://google.github.io/adk-docs/llms-full.txt` |

#### Gemini CLI 集成

```bash
# 安装 ADK 文档扩展
gemini extensions install https://github.com/derailed-dash/adk-docs-ext

# 直接在 Gemini CLI 问 ADK 问题
# How do I create a function tool using Agent Development Kit?
```

#### Claude Code 集成

```bash
# 添加 MCP Server
claude mcp add adk-docs --transport stdio -- uvx --from mcpdoc mcpdoc --urls AgentDevelopmentKit:https://google.github.io/adk-docs/llms.txt --transport stdio
```

添加后，Claude Code 可以直接查询 ADK 文档并生成代码。

#### Cursor IDE 集成

在 Cursor Settings → Tools & MCP → New MCP Server，添加 `mcp.json` 配置：

```json
{
  "mcpServers": {
    "adk-docs-mcp": {
      "command": "uvx",
      "args": [
        "--from", "mcpdoc", "mcpdoc",
        "--urls", "AgentDevelopmentKit:https://google.github.io/adk-docs/llms.txt",
        "--transport", "stdio"
      ]
    }
  }
}
```

#### Antigravity IDE 集成

同样通过 MCP Server 配置，配置文件格式与 Cursor 相同。

> **通用方案**：任何支持 `llms.txt` 标准或能从 URL 获取文档的工具，都可以配置使用 ADK 的文档。

---

### 5. ADK 安装参考（多语言）

#### Python

```bash
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
.venv\Scripts\activate.bat  # Windows CMD
.venv\Scripts\Activate.ps1  # Windows PowerShell

pip install google-adk
pip show google-adk  # 验证安装
```

#### TypeScript

```bash
npm install @google/adk @google/adk-devtools
```

#### Go

```bash
go mod init example.com/my-agent
go get google.golang.org/adk
```

验证：检查 `go.mod` 文件中的 `google.golang.org/adk` 条目。

#### Java（Maven）

```xml
<dependencies>
  <dependency>
    <groupId>com.google.adk</groupId>
    <artifactId>google-adk</artifactId>
    <version>0.5.0</version>
  </dependency>
  <dependency>
    <groupId>com.google.adk</groupId>
    <artifactId>google-adk-dev</artifactId>
    <version>0.5.0</version>
  </dependency>
</dependencies>
```

需要 Java 17+，`maven.compiler.source` 和 `target` 设为 17。

#### Java（Gradle）

```groovy
dependencies {
    implementation 'com.google.adk:google-adk:0.5.0'
    implementation 'com.google.adk:google-adk-dev:0.5.0'
}
```

> **注意**：Java 需要配置 Gradle 将 `-parameters` 传给 javac，或者使用 `@Schema(name = "...")` 注解。

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **Streaming 需要 Live API 模型**：不是所有 Gemini 模型都支持，要查文档确认
2. **语音和文字互斥**：原生音频模型下不能发文字消息
3. **Visual Builder 生成 YAML 配置**：不是纯 Python 代码，是 Agent Config 格式
4. **llms.txt 是 AI 编码的桥梁**：让你的 IDE 助手（Claude Code、Cursor、Gemini CLI）能查阅 ADK 文档
5. **ADK 支持 5 种语言**：Python、TypeScript、Java、Go（均可安装），教程主要用 Python
