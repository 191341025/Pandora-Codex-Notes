# 模块五：Agent 设计与规模化

> 对应 PDF 第 42-49 页

---

## 概念讲解

### 1. 分步定义一个 LLM Agent

白皮书用一个完整案例贯穿：**Software Bug Assistant**（软件缺陷助理），帮客服团队分类处理 bug 报告。

#### Step 1：定义 Agent 身份

三个关键参数：

| 参数 | 必需？ | 说明 | 示例 |
|------|--------|------|------|
| **name** | 必需 | 唯一标识符，用于多 Agent 间委派 | `software_bug_triage_agent` |
| **description** | 推荐 | 能力摘要，其他 Agent 据此决定是否把任务转给它 | "分析新 bug 报告，分类优先级，分配给对应工程团队" |
| **model** | 必需 | 底层 LLM | `gemini-2.5-flash` |

#### Step 2：编写指令（Instructions）

**这是塑造 Agent 行为最关键的部分。**

`instruction` 参数告诉 Agent：核心任务是什么、扮演什么角色、有什么约束、怎么用工具。

**有效指令应该**：
- 清晰具体，描述期望结果
- 提供示例（few-shot prompting）
- 指导工具使用时机和原因
- 可以用 `{variable}` 语法注入动态数据

> **Pro tip（白皮书反复强调）**：
> "你对 Agent 的整个定义就是一个 prompt。"
>
> LLM 会按字面意思理解 Agent 的每个配置——name、description、工具的名称和描述。
> 模糊、不清晰或互相矛盾的命名会导致**"上下文中毒"（Context Poisoning）**——Agent 会困惑、追求错误目标、或错误使用工具。

#### Step 3：配置工具

Bug 助理需要的工具示例：
- `get_user_details(user_id)` — 获取报告 bug 的用户信息
- `search_codebase(file_name)` — 搜索代码库相关文件
- `create_jira_ticket(...)` — 在项目管理系统中创建工单

**LLM 根据工具的名称、docstring 和参数 schema 来决定调用哪个工具。**

> **Pro tip**：工具名和描述要"简洁且独特"。工具越多，模型越容易因歧义而混乱，导致循环或选错工具。

#### Step 4：完成开发周期

测试 → 评估 → 部署

**关键警告**：
> "Agent 系统是非确定性的，会表现出涌现行为。标准单元测试是不够的。"
>
> 必须严格评估两个方面：
> 1. **推理轨迹**：Agent 的一步步逻辑和工具使用是否合理
> 2. **最终输出质量**：准确性、有用性、是否基于事实

---

### 2. Google Agentspace：管理 Agent 舰队

当你从"一个 Agent"发展到"一组 Agent"时，新的挑战出现了：
- 怎么管理它们？
- 非技术人员怎么用？
- 怎么控制数据访问权限？

**Agentspace 的三大功能**：

| 功能 | 说明 |
|------|------|
| **统一企业数据** | 用开箱即用的连接器对接 SharePoint、Google Workspace、Jira 等 |
| **全员自动化** | 产品、市场、运营人员可以用无代码的 Agent Designer 构建自己的 Agent |
| **治理 Agent 舰队** | Agent Gallery 作为中央门户，管理和部署所有 Agent（ADK 自建 + 无代码 + 合作伙伴） |

---

### 3. 其他构建选项

#### Gemini CLI — 终端里的 Agent

**适合**：快速实验、个人开发者

| 特点 | 说明 |
|------|------|
| 免费 | 100 万 token 上下文，每分钟 60 次查询 |
| 开源 | Apache 2.0，可审计、修改、嵌入 |
| 即用 | 直接在终端操作，不需要搭环境 |

#### Firebase Studio — 全栈开发平台

Agent 只是后端，要做完整产品还需要前端、数据库、托管。Firebase Studio 提供一站式方案：

- **App Prototyping Agent**：用自然语言/截图/原型图创建新项目
- **Gemini in Firebase**：AI 辅助编码、调试、测试、重构
- **协作**：团队共享工作区，URL 预览
- **部署**：一键发布到 Firebase App Hosting 或 Cloud Run

**完整工具链**：
```
ADK（Agent 后端） + Agent Starter Pack（生产基础设施） + Firebase Studio（全栈应用）
= 端到端的 Agent 产品
```

---

### 4. Section 2 核心要点速查

| 你的目标 | 最佳方案 |
|---------|---------|
| 用代码构建多 Agent 系统 | ADK（开源） |
| 部署、扩展、管理生产环境 | Vertex AI Agent Engine |
| 给 Agent 长期记忆 | Memory Bank（Vertex AI Agent Engine 内） |
| 让 Agent 发现和对话其他 Agent | A2A 协议 |
| 用 prompt 构建完整 AI 应用 | Firebase Studio |
| 快速在终端实验 | Gemini CLI |

---

### 5. 实际案例

#### Box：用 ADK + A2A 加速内容管理
- 用 Gemini + ADK 构建 A2A 启用的 Agent
- 连接 Box 智能内容云
- 用户用自然语言提问，从海量文档中获取即时答案
- 下一步：支持电子签名和审批的事务型 Agent

#### Zoom：用 A2A 自动调度会议
- Zoom AI Companion 与 Google Agentspace 集成
- 自动从 Gmail 上下文调度 Zoom 会议、更新 Google Calendar
- 消除手动排会议的来回沟通

---

## 问答记录

> 待补充

---

## 重点标记

1. **Agent 定义的每个字符串都是 prompt**：name、description、tool 描述都会被 LLM 解读
2. **Context Poisoning 是常见陷阱**：模糊的命名会让 Agent 迷失方向
3. **标准测试不够**：必须评估推理轨迹和输出质量
4. **Agentspace 解决"多 Agent 管理"问题**：非技术人员也能构建和管理 Agent
5. **ADK + Agent Starter Pack + Firebase Studio = 完整产品线**
6. **Gemini CLI 适合快速实验**：免费、开源、即用
