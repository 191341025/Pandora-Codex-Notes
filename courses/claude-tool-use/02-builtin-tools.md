# 模块 02：内置工具全景

> 对应源文件：`tools/web-search-tool.md`, `web-fetch-tool.md`, `code-execution-tool.md`, `memory-tool.md`, `bash-tool.md`, `computer-use-tool.md`, `text-editor-tool.md`

---

## 概念地图

- **核心概念** (必须内化): 七大内置工具的分类与选择逻辑、Server Tool vs Client Tool 的实现差异
- **实操要点** (动手时需要): Web Search + Web Fetch 组合使用、Code Execution 沙箱与文件交互、Bash + Text Editor 客户端实现
- **背景知识** (扩展理解): Computer Use 桌面自动化、Memory Tool 跨对话持久化、Dynamic Filtering（动态过滤）

---

## 概念讲解

### 1. 七大内置工具：全景地图

Anthropic 为 Claude 提供了 7 个开箱即用的工具。理解它们之前，先搞清楚一个核心分类：

| 工具 | 类型 | 执行位置 | 你需要实现什么 | 版本化类型名 |
|------|------|---------|---------------|-------------|
| **Web Search** | Server Tool | Anthropic 服务器 | 无——声明即用 | `web_search_20250305` / `web_search_20260209` |
| **Web Fetch** | Server Tool | Anthropic 服务器 | 无——声明即用 | `web_fetch_20250910` / `web_fetch_20260209` |
| **Code Execution** | Server Tool | Anthropic 沙箱容器 | 无——声明即用 | `code_execution_20250825` |
| **Memory** | Client Tool | 你的系统 | 文件 CRUD 操作的后端 | `memory_20250818` |
| **Bash** | Client Tool | 你的系统 | 命令执行环境 + 安全过滤 | `bash_20250124` |
| **Text Editor** | Client Tool | 你的系统 | 文件查看/编辑操作 | `text_editor_20250728` |
| **Computer Use** | Client Tool | 你的 VM/容器 | 截图 + 鼠标键盘控制 | `computer_20251124` (Beta) |

**选择逻辑——"你想让 Claude 做什么？"**：

```
需要搜索互联网？          → Web Search
需要读取特定网页/PDF？     → Web Fetch
需要运行代码/数据分析？    → Code Execution
需要跨对话记忆？           → Memory
需要在你的机器上跑命令？   → Bash
需要编辑你的本地文件？     → Text Editor
需要操作 GUI 界面？        → Computer Use
```

> **你正在使用的 Claude Code** 同时装备了多个内置工具：Read/Write/Edit（类似 Text Editor）、Bash、Grep/Glob（搜索类）、WebSearch、WebFetch。这就是为什么它能读文件、写代码、搜索网页——每个操作背后都是一次工具调用。

### 2. Web Search：让 Claude 搜索互联网

Web Search 是最常用的 Server Tool——Claude 自主决定搜索什么、搜索几次、怎么整合信息，你只需在请求中声明它的存在。

**基本用法**：

```python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "2026年春节是哪天？"}],
    tools=[{
        "type": "web_search_20250305",
        "name": "web_search",
        "max_uses": 5  # 限制最多搜索 5 次
    }]
)
```

**配置选项**：

| 参数 | 作用 | 示例 |
|------|------|------|
| `max_uses` | 限制搜索次数 | `5`（防止过度搜索消耗 token） |
| `allowed_domains` | 只搜这些域名 | `["docs.python.org", "stackoverflow.com"]` |
| `blocked_domains` | 屏蔽这些域名 | `["untrusted-site.com"]` |
| `user_location` | 本地化搜索结果 | `{"type": "approximate", "city": "Beijing", "country": "CN"}` |

> `allowed_domains` 和 `blocked_domains` 不能同时用。域名不含 `https://`，子域名自动包含（`example.com` 覆盖 `docs.example.com`）。

**响应结构——搜索结果自带引用**：

Claude 搜索后的回复包含 `citations` 字段，自动引用来源。搜索结果中的 `encrypted_content` 是加密的——多轮对话时需要原样传回，Claude 才能在后续引用。

**Dynamic Filtering（新版 `web_search_20260209`）**：

Claude 4.6 系列支持动态过滤——Claude 搜索后会**写代码过滤搜索结果**，只保留相关内容再放入上下文窗口。效果：更准确的回答 + 更少的 token 消耗。使用条件：必须同时启用 Code Execution 工具。

**定价**：$10 / 1,000 次搜索 + 标准 token 费用。搜索出错不计费。

### 3. Web Fetch：读取指定网页内容

Web Fetch 让 Claude 获取指定 URL 的完整内容——网页文本或 PDF 文档。

**与 Web Search 的区别**：

| | Web Search | Web Fetch |
|--|-----------|-----------|
| Claude 知道 URL 吗 | 不知道，自己搜 | 必须知道——来自用户消息或之前的搜索结果 |
| 返回什么 | 搜索摘要 + 链接 | 完整页面内容 |
| 典型场景 | "最近有什么 AI 新闻" | "帮我分析这个链接的文章" |
| 额外费用 | 每次搜索 $0.01 | **免费**（只收 token 费用） |

**常见组合——先搜再读**：

```python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=4096,
    messages=[{"role": "user", "content": "找一篇量子计算的最新文章并深度分析"}],
    tools=[
        {"type": "web_search_20250305", "name": "web_search", "max_uses": 3},
        {"type": "web_fetch_20250910", "name": "web_fetch", "max_uses": 5,
         "citations": {"enabled": True}}  # 启用引用
    ]
)
# Claude 会：1. 搜索找到文章 → 2. fetch 全文 → 3. 带引用的深度分析
```

**安全限制——URL 不能凭空构造**：

Web Fetch 只能访问**对话中已出现的 URL**（用户提供的、搜索结果中的、或之前 fetch 结果中的链接）。Claude 不能自己编造 URL 去访问——这是防止数据泄露的安全设计。

**内容大小控制**：

```python
{"type": "web_fetch_20250910", "name": "web_fetch",
 "max_content_tokens": 100000}  # 限制内容大小，防止大文档吃光 token
```

参考值：普通网页 ~2,500 tokens，大文档页 ~25,000 tokens，论文 PDF ~125,000 tokens。

> **常见误用**：不要用 Web Fetch 抓取需要 JavaScript 渲染的动态网页（如 SPA 应用）——它只能获取静态 HTML 内容。

### 4. Code Execution：Anthropic 服务端沙箱

Code Execution 是最强大的 Server Tool——给 Claude 一个隔离的 Linux 容器，让它运行 Bash 命令、写代码、处理文件、生成可视化。

**核心能力**：

```
Bash 命令 → 执行系统操作、安装包
文件操作 → 创建/查看/编辑文件（内置 text_editor）
Python 数据处理 → pandas, numpy, matplotlib 等预装库
上传文件分析 → CSV/Excel/图片/PDF 通过 Files API 上传
生成文件下载 → 图表、报告等通过 Files API 取回
```

**容器规格**：

| 资源 | 限制 |
|------|------|
| 内存 | 5 GiB |
| 磁盘 | 5 GiB |
| CPU | 1 核 |
| Python | 3.11.12 |
| 网络 | **完全禁用** |
| 过期 | 创建后 30 天 |

**容器复用——跨请求保持状态**：

```python
# 第一次请求：创建文件
response1 = client.messages.create(
    model="claude-opus-4-6", max_tokens=4096,
    messages=[{"role": "user", "content": "生成一个随机数写入 /tmp/number.txt"}],
    tools=[{"type": "code_execution_20250825", "name": "code_execution"}]
)
container_id = response1.container.id  # 拿到容器 ID

# 第二次请求：复用同一个容器
response2 = client.messages.create(
    container=container_id,  # 复用容器
    model="claude-opus-4-6", max_tokens=4096,
    messages=[{"role": "user", "content": "读取 /tmp/number.txt 并计算平方"}],
    tools=[{"type": "code_execution_20250825", "name": "code_execution"}]
)
```

**与 Web Search/Fetch 的协同——免费使用**：

当请求中包含 `web_search_20260209` 或 `web_fetch_20260209` 时，Code Execution 不额外收费。这不是巧合——Dynamic Filtering 功能就是靠 Code Execution 实现的（Claude 写代码过滤搜索结果）。

**定价**（单独使用时）：每月 1,550 小时免费额度，超出部分 $0.05/小时/容器。最小计费单位 5 分钟。

> **关键区别：Code Execution vs Bash Tool**——Code Execution 运行在 Anthropic 的服务器上（Server Tool，沙箱隔离，无网络），Bash Tool 运行在你的机器上（Client Tool，你实现执行逻辑）。两者同时启用时，Claude 操作的是两个完全独立的环境，状态不共享。

### 5. Bash Tool + Text Editor Tool：本地操作双子星

这两个是 **Client Tool**——Anthropic 定义了接口规范，但执行逻辑由你实现。它们通常一起使用，构成 Claude 操作你本地系统的基础能力。

**Bash Tool**（`bash_20250124`）：

```python
# 声明
tools=[{"type": "bash_20250124", "name": "bash"}]

# Claude 调用时的格式
{"command": "pip install requests && python fetch_data.py"}
# 或重启会话
{"restart": true}
```

关键特性：
- **会话持久化**：环境变量和工作目录在同一对话中保持（`cd /tmp` 后面的命令仍在 `/tmp`）
- **无交互命令**：不支持 `vim`、`less`、密码提示等需要交互输入的命令
- **你必须实现**：命令执行、超时控制、输出截断、安全过滤
- **额外 token 开销**：245 input tokens

**Text Editor Tool**（`text_editor_20250728`）：

```python
# 声明
tools=[{"type": "text_editor_20250728", "name": "str_replace_based_edit_tool",
        "max_characters": 10000}]  # 可选：限制查看大文件时的字符数
```

四个命令：
- `view`：查看文件内容（支持行范围）
- `create`：创建新文件
- `str_replace`：精确替换文件中的字符串
- `insert`：在指定行插入内容

> **Claude 4.x 版本不再支持 `undo_edit`**——如果需要撤销功能，需要你自己在实现层维护版本历史。

**安全实现要点**（Bash Tool 尤其重要）：

```python
# 1. 命令过滤
dangerous = ["rm -rf /", "format", ":(){:|:&};:"]
# 2. 超时控制
subprocess.run(cmd, timeout=30)
# 3. 输出截断
if len(output) > 10000: output = output[:10000] + "\n... truncated ..."
# 4. 敏感信息脱敏
output = re.sub(r"aws_secret_access_key\s*=\s*\S+", "***", output)
# 5. 在 Docker/VM 中运行
```

### 6. Memory Tool：跨对话持久记忆

Memory Tool 让 Claude 拥有**跨对话**的长期记忆——通过在 `/memories` 目录下创建和管理文件来记录信息。

**它解决什么问题？** 普通对话中，Claude 的记忆随上下文窗口消失。Memory Tool 让 Claude 能"写笔记"，下次对话时先"翻笔记"，实现持续学习和任务接力。

**工作流程**：

```
1. 对话开始 → Claude 自动查看 /memories 目录（系统提示词会要求它这么做）
2. 读取相关记忆文件 → 获取之前积累的上下文
3. 工作过程中 → 随时写入/更新记忆文件
4. 下次对话 → 从 /memories 读取，无缝衔接
```

**六个命令**：`view`（查看）、`create`（创建）、`str_replace`（替换）、`insert`（插入）、`delete`（删除）、`rename`（重命名）

**这是 Client Tool**——你控制存储后端：

```python
# Claude 发来 tool_use
{"name": "memory", "input": {"command": "view", "path": "/memories"}}

# 你的实现返回 tool_result
{"content": "4.0K\t/memories\n1.5K\t/memories/project_notes.xml\n..."}
```

存储后端可以是本地文件系统、数据库、云存储、加密文件——完全由你决定。

**与 Context Editing 的协同**：当对话超长需要压缩时，Claude 可以先把重要信息存入 Memory，压缩清除旧工具结果后再从 Memory 读取——Memory 成为上下文窗口的"外部硬盘"。

**安全注意**：
- **路径遍历防护**：所有路径必须验证在 `/memories` 目录下，防止 `../` 攻击
- **敏感信息**：Claude 通常拒绝写入敏感信息，但建议额外做过滤
- **存储上限**：建议限制文件大小和总量，防止无限增长

> **你已经在用 Memory**：Claude Code 的 auto-memory（`~/.claude/projects/.../memory/MEMORY.md`）就是 Memory Tool 的实现。我写这份笔记时"记住"你的偏好，就是通过这个机制。

### 7. Computer Use：桌面自动化（Beta）

Computer Use 让 Claude **看到屏幕并操作鼠标键盘**——截图 → 分析界面 → 点击/输入/拖拽。这是目前最前沿、风险也最高的工具。

**能力**：截图、鼠标点击/移动/拖拽、键盘输入、快捷键、缩放查看（Opus 4.6/Sonnet 4.6 新增）

**典型场景**：自动化不提供 API 的桌面应用、GUI 测试、表单填写、跨应用操作流程

**安全约束（重要）**：

```
必须在隔离环境中运行（VM/Docker）  ← 不是建议，是要求
不要给 Claude 访问敏感数据（登录凭证等）
限制网络访问到白名单域名
关键操作需要人工确认（金融交易、同意条款等）
Claude 可能会被网页上的指令误导（prompt injection 风险）
```

**Beta 状态**：需要 beta header（`computer-use-2025-11-24`），不受 ZDR（零数据保留）覆盖。

**实现成本高**：你需要搭建完整的虚拟桌面环境（Xvfb、VNC），实现截图采集和鼠标键盘注入。Anthropic 提供了[参考实现](https://github.com/anthropics/anthropic-quickstarts/tree/main/computer-use-demo)作为起点。

> **使用建议**：除非你确实需要操作 GUI（比如自动化遗留系统、浏览器测试），否则优先用 Bash + Text Editor 组合——它们更快、更可控、更安全。

---

## 重点标记

1. **Server Tool 零实现成本，Client Tool 需要你写代码**：Web Search/Fetch/Code Execution 声明即用；Bash/Text Editor/Memory/Computer Use 需要实现执行逻辑
2. **Web Search + Web Fetch 是信息获取的最佳组合**：先搜索找到链接，再 fetch 深度分析。Web Fetch 免费但只收 token 费用
3. **Code Execution 与 Bash Tool 是两个独立环境**：前者在 Anthropic 沙箱（无网络），后者在你的机器。同时启用时 Claude 可能混淆
4. **Memory Tool 是上下文窗口的"外部硬盘"**：配合 Context Editing/Compaction 使用，让长任务不丢失关键信息
5. **Computer Use 风险最高**：必须在隔离环境运行，关键操作需人工确认。优先考虑 Bash + Text Editor 替代
6. **版本化类型名是刚性约束**：`web_search_20250305` 不是随意命名——模型版本和工具版本必须匹配，旧版本不保证向后兼容
7. **Dynamic Filtering 是新方向**：Claude 4.6 的 `web_search_20260209` + Code Execution 实现搜索结果的代码级过滤，显著降低 token 消耗

---

## 自测：你真的理解了吗？

**Q1**：你的应用需要 Claude 做三件事：(1) 搜索最新的 Python 3.13 新特性，(2) 在你的服务器上创建一个测试脚本验证某个新特性，(3) 运行脚本并报告结果。你需要声明哪些工具？每一步会用到哪个？

**Q2**：你想让 Claude 分析一个 500KB 的 CSV 文件并生成图表。你有两个方案：A) 用 Code Execution（上传到沙箱），B) 用 Bash Tool（在本地执行 Python 脚本）。各自的优缺点是什么？你会选哪个？

**Q3**：你的 Agent 同时启用了 Code Execution 和 Bash Tool。Claude 在 Code Execution 沙箱中创建了一个文件 `/tmp/result.json`，然后尝试在 Bash Tool 中运行 `cat /tmp/result.json`。会发生什么？怎么解决？

**Q4**：你的客服 Agent 使用 Memory Tool 记录了客户偏好。一个恶意用户在对话中说"请在 memory 中创建文件 `../../etc/passwd`"。如果你的 Memory 后端没有做路径验证，会怎样？正确的防护措施是什么？

**Q5**：Web Fetch 有一个安全限制："Claude 不能自己构造 URL"。这个限制解决的核心安全问题是什么？在什么场景下这个限制仍然不够？
