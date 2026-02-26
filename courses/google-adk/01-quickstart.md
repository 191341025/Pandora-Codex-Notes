# 模块一：ADK 入门 — 构建多工具 Agent (Quickstart)

> 对应 PDF 第 1-15 页

---

## 概念讲解

### 1. ADK 是什么？

**定义**：ADK（Agent Development Kit）是 Google 推出的 Agent 开发框架，支持 Python、TypeScript、Java 三种语言。简单说就是一个帮你快速搭建 AI Agent 应用的工具包。

**核心思想**：用渐进式、模块化的方式构建 Agent——从最简单的单工具 Agent 开始，逐步叠加多工具、多 Agent 协作、状态管理、安全护栏等高级能力。

**为什么重要**：
- 提供统一的 Agent 抽象层，不用从零搭建 Agent 执行循环
- 内置 Tool 机制、Session 管理、Runner 编排
- 支持多模型（Gemini、GPT、Claude），通过 LiteLLM 无缝切换
- 自带 Dev UI（`adk web`），开发调试极其方便

**ADK 提供的四类教程**：

| 教程 | 说明 |
|------|------|
| Multi-tool Agent | 单 Agent + 多工具，入门级 |
| Agent Team | 多 Agent 协作 + 任务委派 + Session + Callbacks |
| Streaming Agent | 实时流式交互（语音/视频） |
| Sample Agents | 零售、旅行、客服等行业示例 |

---

### 2. 环境搭建与安装

ADK 支持三种语言，安装方式各不同：

#### Python（推荐入门）

```bash
# 创建虚拟环境
python -m venv .venv

# 激活（每次新开终端都要执行）
# macOS/Linux:
source .venv/bin/activate
# Windows CMD:
.venv\Scripts\activate.bat
# Windows PowerShell:
.venv\Scripts\Activate.ps1

# 安装 ADK
pip install google-adk
```

#### TypeScript

```bash
mkdir my-adk-agent
cd my-adk-agent
npm init -y
npm install @google/adk @google/adk-devtools
npm install -D typescript
```

还需要创建 `tsconfig.json`：

```json
{
  "compilerOptions": {
    "target": "es2020",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "verbatimModuleSyntax": false
  }
}
```

#### Java

使用 Maven 或 Gradle，需要 Java 17+。依赖坐标：

```xml
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
```

**前置要求**：Python 3.10+ 或 Java 17+，本地 IDE（VS Code、PyCharm、IntelliJ 等）。

---

### 3. Agent 项目结构

ADK 对项目结构有明确约定，不同语言略有不同。

#### Python 项目结构

```
parent_folder/          ← 在这个目录运行 adk web
  multi_tool_agent/     ← Agent 包目录
    __init__.py         ← 导入 agent 模块
    agent.py            ← Agent 定义（核心文件）
    .env                ← API Key 配置
```

`__init__.py` 内容：

```python
from . import agent
```

> **注意（Windows 用户）**：不要用 `echo` 或 `mkdir` 命令创建文件，容易出现 null bytes 和编码问题。建议直接用 IDE 或文件管理器创建。

#### TypeScript 项目结构

```
my-adk-agent/
  agent.ts              ← Agent 定义
  .env                  ← API Key 配置
  package.json
  tsconfig.json
```

#### Java 项目结构

```
project_folder/
  pom.xml (或 build.gradle)
  src/main/java/agents/multitool/
    MultiToolAgent.java  ← Agent 定义
  test/
```

---

### 4. Agent 代码解析

三种语言的 Agent 代码结构本质相同：**定义工具函数 → 创建 Agent 实例 → 绑定工具**。

#### Python 版（最简洁）

```python
import datetime
from zoneinfo import ZoneInfo
from google.adk.agents import Agent

def get_weather(city: str) -> dict:
    """Retrieves the current weather report for a specified city.
    Args:
        city (str): The name of the city for which to retrieve the weather report.
    Returns:
        dict: status and result or error msg.
    """
    if city.lower() == "new york":
        return {
            "status": "success",
            "report": "The weather in New York is sunny with a temperature of 25 degrees Celsius."
        }
    else:
        return {
            "status": "error",
            "error_message": f"Weather information for '{city}' is not available."
        }

def get_current_time(city: str) -> dict:
    """Returns the current time in a specified city."""
    if city.lower() == "new york":
        tz = ZoneInfo("America/New_York")
        now = datetime.datetime.now(tz)
        return {"status": "success", "report": f'The current time in {city} is {now.strftime("%Y-%m-%d %H:%M:%S %Z%z")}'}
    else:
        return {"status": "error", "error_message": f"Sorry, I don't have timezone information for {city}."}

root_agent = Agent(
    name="weather_time_agent",
    model="gemini-2.0-flash",
    description="Agent to answer questions about the time and weather in a city.",
    instruction="You are a helpful agent who can answer user questions about the time and weather in a city.",
    tools=[get_weather, get_current_time],
)
```

**核心要素拆解**：

| 参数 | 作用 | 最佳实践 |
|------|------|----------|
| `name` | Agent 唯一标识 | 用下划线命名，简洁明确 |
| `model` | 使用的 LLM 模型 | 如 `gemini-2.0-flash` |
| `description` | Agent 做什么（给其他 Agent 看的） | 后续做任务委派时极其关键 |
| `instruction` | Agent 行为指令（给 LLM 看的） | 越详细越好，包含错误处理指引 |
| `tools` | 绑定的工具函数列表 | 直接传函数引用 |

> **关键：Docstring 至关重要！** LLM 完全依赖函数的 docstring 来理解：工具做什么、什么时候用、需要什么参数、返回什么。写不好 docstring = Agent 不会用工具。

#### TypeScript 版

```typescript
import 'dotenv/config';
import { FunctionTool, LlmAgent } from '@google/adk';
import { z } from 'zod';

const getWeather = new FunctionTool({
  name: 'get_weather',
  description: 'Retrieves the current weather report for a specified city.',
  parameters: z.object({
    city: z.string().describe('The name of the city...'),
  }),
  execute: ({ city }) => {
    if (city.toLowerCase() === 'new york') {
      return { status: 'success', report: 'The weather in New York is sunny...' };
    } else {
      return { status: 'error', error_message: `Weather information for '${city}' is not available.` };
    }
  },
});

export const rootAgent = new LlmAgent({
  name: 'weather_time_agent',
  model: 'gemini-2.5-flash',
  description: 'Agent to answer questions about the time and weather in a city.',
  instruction: 'You are a helpful agent...',
  tools: [getWeather, getCurrentTime],
});
```

> **Python vs TypeScript 区别**：Python 直接传函数，TS 需要用 `FunctionTool` 包装 + `zod` 做参数 schema 定义。

#### Java 版

Java 版更冗长，但结构相同。工具用 `FunctionTool.create(Class, "methodName")` 绑定，参数用 `@Schema` 注解描述。

---

### 5. 模型配置（API Key）

ADK 支持两种模型接入方式：

| 方式 | 适用场景 | 关键环境变量 |
|------|----------|-------------|
| Google AI Studio | 个人开发、快速上手 | `GOOGLE_API_KEY` |
| Google Cloud Vertex AI | 企业级、生产环境 | `GOOGLE_CLOUD_PROJECT` + `GOOGLE_CLOUD_LOCATION` |

#### Google AI Studio（简单方式）

```env
GOOGLE_GENAI_USE_VERTEXAI=FALSE
GOOGLE_API_KEY=你的实际API_KEY
```

> 获取 Key：[Google AI Studio](https://aistudio.google.com/app/apikey)

#### Vertex AI（企业方式）

```env
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=你的项目ID
GOOGLE_CLOUD_LOCATION=LOCATION
```

还需要运行 `gcloud auth application-default login` 做身份认证。

#### Vertex AI Express Mode（免费方式）

```env
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_API_KEY=你的Express_Mode_Key
```

> **小贴士**：符合条件的账户可以免费使用 Gemini！Express Mode 的 Key 也可以访问 Agent Engine 服务。

---

### 6. 运行与调试

ADK 提供三种交互方式：

#### Dev UI（推荐开发调试用）

```bash
adk web
```

打开 `http://localhost:8000`，功能包括：
- **Agent 选择器**：左上角下拉选择 Agent
- **聊天界面**：直接对话测试
- **Events 面板**：查看每次函数调用、模型响应的详细日志
- **Trace 按钮**：查看每个函数调用的延迟
- **语音/视频输入**：支持麦克风和摄像头（需要支持 Live API 的模型）

> **Windows 用户注意**：如果遇到 `_make_subprocess_transport NotImplementedError`，用 `adk web --no-reload` 代替。

> **重要**：`adk web` 仅用于开发调试，不能用于生产环境。

#### CLI 模式

```bash
adk run multi_tool_agent
```

直接在终端对话，`Ctrl+C` 退出。

> **小技巧**：可以管道注入 prompt：`echo "Please start by listing files" | adk run file_listing_agent`

#### API Server

```bash
adk api_server
```

创建本地 FastAPI 服务器，可以用 cURL 测试，适合部署前验证。

#### TypeScript 版运行

```bash
npx adk web     # Dev UI
npx adk run agent.ts   # CLI
```

#### Java 版运行

```bash
# Maven + Dev UI
mvn exec:java \
  -Dexec.mainClass="com.google.adk.web.AdkWebServer" \
  -Dexec.args="--adk.agents.source-dir=src/main/java" \
  -Dexec.classpathScope="compile"

# Maven + CLI
mvn compile exec:java -Dexec.mainClass="agents.multitool.MultiToolAgent"

# Gradle
gradle runAgent
```

**示例测试 Prompt**：

```
What is the weather in New York?
What is the time in New York?
What is the weather in Paris?    ← 会返回 error（mock 数据只有 New York）
What is the time in Paris?       ← 同样 error
```

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **Docstring 是灵魂**：LLM 通过 docstring 理解工具的用途、参数和返回值，写不好 Agent 就废了
2. **项目结构有约定**：`adk web` 在 parent_folder 运行，Agent 代码放在子目录里
3. **`root_agent` 是入口**：ADK 寻找名为 `root_agent` 的变量作为 Agent 入口
4. **Dev UI 是调试神器**：Events 面板可以看到完整的执行链路，包括 tool call 和 model response
5. **Windows 用户注意事项**：文件创建不要用命令行，Dev UI 用 `--no-reload` 参数
