# 模块六：专业开发模式 — 从脚本到企业系统

> 对应 PDF 第 53-66 页（Ch 16）

---

## 概念讲解

### 1. Claude Code Gap — 为什么需要专业工作流

**核心问题**：Claude Code 擅长快速生成代码，但它经常跳过专业开发中的关键步骤。这对小脚本没问题，但对企业级应用是致命的。

**专业开发者不只是写代码**——他们还要：
- 规划（Plan）
- 设计架构（Architect）
- 设计 UI/UX
- 测试（Test）
- 做安全审查（Secure）

> **简单说**：Claude Code 默认模式 = 快但粗糙。专业开发模式 = 用 Hooks + Agents + Commands 补上"规划→架构→安全→测试"这些被跳过的步骤。

---

### 2. 三个专业工作流阶段命令

#### 阶段一：项目规划与架构

**命令**：`/project:architect-plan "Design a scalable e-commerce platform"`

```markdown
Perform comprehensive project planning:
1. Define system requirements and constraints
2. Create high-level architecture diagrams
3. Design database schema
4. Plan API structure
5. Define security protocols
6. Create testing strategy
7. Estimate scalability needs (users, data, traffic)

Output a structured architecture document with:
- System overview
- Component breakdown
- Data flow diagrams
- Security considerations
- Scalability assessment
```

#### 阶段二：前端开发（TDD 驱动）

**命令**：`/project:frontend-tdd "Implement user dashboard"`

```markdown
Develop frontend with professional standards:
1. Review UI/UX designs or create wireframes
2. Set up component architecture
3. Write tests FIRST (TDD approach)
4. Implement components
5. Ensure accessibility (WCAG compliance)
6. Optimize performance
7. Add proper error handling

Use modern patterns:
- React Query for server state
- Proper state management
- Component composition
- Performance optimization
```

#### 阶段三：后端开发（安全优先）

**命令**：`/project:backend-secure "Build authentication system"`

```markdown
Professional backend development:
1. Design API contracts first
2. Write integration tests
3. Implement with security in mind
4. Add comprehensive error handling
5. Include logging and monitoring
6. Document all endpoints
7. Performance profiling

Follow patterns:
- Repository pattern for data access
- Service layer for business logic
- Proper validation and sanitization
- Rate limiting and security headers
```

> **三个命令覆盖了完整的开发生命周期**：架构 → 前端（TDD）→ 后端（安全优先）。

---

### 3. 三个专业角色 Persona

#### System Architect

```yaml
name: system-architect
description: |
  Enterprise architecture specialist focusing on:
  - Scalable system design
  - Technology selection
  - Integration patterns
  - Performance architecture
  - Security architecture
  Call when planning new systems or major refactors
```

#### Security Engineer

```yaml
name: security-engineer
description: |
  Security specialist for:
  - Vulnerability assessment
  - Security architecture review
  - Penetration testing strategies
  - Compliance requirements
  - Security best practices
  Call for any security-sensitive features
```

#### Performance Engineer

```yaml
name: performance-engineer
description: |
  Performance optimization expert for:
  - Bottleneck identification
  - Query optimization
  - Caching strategies
  - Load testing
  - Scalability planning
```

> **三个角色 = 三个视角**：每个关键决策都应该经过架构师、安全工程师和性能工程师的审视。

---

### 4. 专业故障排除 — Five Whys 方法论

分三步走的系统化故障排除流程：

**Step 1：问题识别**
```
/project:investigate "Production errors increasing"
→ 分析错误日志 → 检查监控 → 审查近期部署 → 识别模式 → 记录症状
```

**Step 2：根因分析（Five Whys）**
```
/project:five-whys "API timeout errors"
→ Why is the API timing out?
  → Why is the database query slow?
    → Why is the index not being used?
      → Why was the index dropped?
        → Why wasn't this caught in testing?
→ 输出：根因 + 修复方案
```

**Step 3：性能分析**
```
/project:profile "Analyze slow endpoints"
→ 运行性能分析器 → 识别瓶颈 → 分析数据库查询
→ 检查内存使用 → 审查缓存效果 → 提出优化建议
```

---

### 5. 架构分析 Hook 与架构评分系统

#### 架构影响分析 Hook

当代码修改涉及关键文件时自动触发架构审查：

```python
critical_patterns = [
    'schema', 'migration', 'auth', 'security',
    'api/', 'architecture', 'config'
]

if any(pattern in changed.lower() for pattern in critical_patterns):
    print("\n🏗 ARCHITECTURE IMPACT DETECTED")
    print("Consider running: /project:architect-review")
```

> **效果**：修改了 schema、migration、auth 等文件 → 自动提醒你做架构审查。

#### 架构评分系统

自动化的架构质量评分（满分 100）：

| 检查项 | 影响 |
|--------|------|
| 有 ARCHITECTURE.md 文档 | 没有 → -10 |
| 有 src/components + src/services 分离 | 没有 → -5 |
| 有测试覆盖率报告 | 没有 → -20 |
| 有 .env.example（安全实践） | 有 → +5 |
| 使用服务端状态管理（react-query/swr） | 有 → +5 |

```
分数 < 70 → ⚠ 架构需要改进！建议运行 /project:architect-review
```

---

### 6. 质量门（Quality Gates）

在代码编辑前自动检查测试覆盖：

```bash
#!/bin/bash
# .claude/hooks/quality_gate.sh

if [[ "$CLAUDE_FILE_PATHS" =~ \.(ts|js|py)$ ]]; then
    base_name=$(basename "$CLAUDE_FILE_PATHS" | sed 's/\.[^.]*$//')
    test_file=$(find . -name "*${base_name}*test*" -o -name "*${base_name}*spec*" | head -1)

    if [ -z "$test_file" ]; then
        echo "⚠ WARNING: No tests found for $CLAUDE_FILE_PATHS" >&2
        echo "Consider TDD: Write tests before implementation" >&2
        exit 0  # Don't block, just warn
    fi
fi
```

> **设计选择**：这里用 exit 0（警告但不阻止）。如果你想强制 TDD，改为 exit 2 即可阻止无测试的代码修改。

---

### 7. 容量规划矩阵

| 规模级别 | 用户数 | 需求 | 架构变更 |
|----------|--------|------|----------|
| **Prototype** | 1-100 | 单服务器 | 单体架构 OK |
| **Small** | 100-1K | 基本缓存 | 加 Redis |
| **Medium** | 1K-10K | 负载均衡 | 水平扩展 |
| **Large** | 10K-100K | 微服务 | 服务拆分 |
| **Enterprise** | 100K+ | 完整分布式 | 多区域、消息队列 |

配合 Slash Command 使用：

```
/project:scale-assessment "Current system analysis"
→ 当前容量 → 识别瓶颈 → 增长预测 → 扩展建议 → 成本分析 → 输出扩展路线图
```

---

### 8. 专业开发检查清单

每个严肃项目都应该用这个清单：

- [ ] 编码前完成架构文档
- [ ] 完成安全审查
- [ ] 定义 API 契约
- [ ] 测试策略就位
- [ ] 性能基准设定
- [ ] 监控配置完成
- [ ] 文档已更新
- [ ] 代码审查流程确定
- [ ] 部署策略已规划
- [ ] 回滚程序已就绪

---

### 9. 完整专业工作流示例

```bash
# 1. 架构规划
/project:architect-plan "Build a real-time collaboration tool"

# 2. 安全审查
/project:security-review "Review authentication approach"

# 3. 前端 TDD
/project:frontend-tdd "Implement collaborative editor"

# 4. 后端安全开发
/project:backend-secure "Build WebSocket server"

# 5. 性能测试
/project:performance-test "Load test with 1000 concurrent users"

# 6. 部署就绪检查
/project:deploy-checklist "Production deployment readiness"
```

> **6 步把 Claude Code 从代码生成器转变为完整的开发平台**。

---

### 10. 五阶段成长路线图

| 阶段 | 时间线 | 内容 |
|------|--------|------|
| **Phase 1: 基础自动化** | Week 1 | 简单通知 Hook、基础日志、首个安全检查、把重复 Prompt 转成 Slash Command |
| **Phase 2: 安全与质量** | Week 2-3 | 命令验证、自动格式化/lint、pre-commit 检查、文件保护、TDD 提醒 |
| **Phase 3: 高级工作流** | Month 2 | 多工具模式匹配、CI/CD 集成、可观测性仪表板、项目级自动化、架构评分 |
| **Phase 4: 多 Agent 系统** | Month 3+ | 首批 Sub-Agent、Agent 链、主动触发、Meta-Agent、专业角色 Persona |
| **Phase 5: 企业开发** | Month 4+ | 完整专业工作流、架构审查自动化、扩展性评估、故障排除链、容量规划 |

---

### 11. 20 条关键要点

| # | 要点 | 类别 |
|---|------|------|
| 1 | **从简单开始**：先基础 Hook，逐步增加复杂度 | 策略 |
| 2 | **安全优先**：始终审查 Hook 代码，使用限制性权限 | 安全 |
| 3 | **社区智慧**：用 Slash Command 代替复制粘贴，新建会话而非 clear | 工具 |
| 4 | **组合工具**：Claude Code + 其他 AI 工具增强学习 | 学习 |
| 5 | **系统思维**：规模化时关注 Context、Model、Prompt 的流动 | 架构 |
| 6 | **读文档**：docs.anthropic.com 是最佳资源 | 知识 |
| 7 | **不跳过规划**：用 Hook/工作流强制架构、安全、测试阶段 | 流程 |
| 8 | **拥抱 TDD**：用 Hook 在实现前提醒写测试 | 质量 |
| 9 | **度量质量**：实施架构评分和质量门 | 质量 |
| 10 | **谨慎扩展**：用瓶颈分析规划增长 | 架构 |
| 11 | **Agent 单一职责**：每个 Agent 只做一件事，效果最好 | Agent |
| 12 | **上下文策略**：Main Claude 管延续，Agent 管全新视角 | Agent |
| 13 | **三者组合**：Hook 管自动化，Agent 管智能，Main Claude 管上下文 | 整合 |
| 14 | **Description 决定一切**：Agent 的 Description 决定自动发现的准确度 | Agent |
| 15 | **链式力量**：多 Agent 工作流解锁复杂自动化 | Agent |
| 16 | **位置有讲究**：个人 Agent 做工具，项目 Agent 做团队统一 | Agent |
| 17 | **概率性是特性**：拥抱 Agent 的概率性选择 | Agent |
| 18 | **超越代码**：Agent 可用于写作、分析、组织等任何工作流 | 通用 |
| 19 | **最小权限**：Agent 只给必要的工具权限 | 安全 |
| 20 | **颜色编码**：用颜色快速识别哪个 Agent 在运行 | UX |

---

## 问答记录

> 待补充（学习后讨论时填写）

---

## 重点标记

1. **Claude Code Gap**：默认模式跳过规划→架构→安全→测试，专业工作流补上这些
2. **三个阶段命令覆盖完整生命周期**：architect-plan → frontend-tdd → backend-secure
3. **三个角色 Persona 提供三个审查视角**：架构 + 安全 + 性能
4. **Five Whys 是系统化故障排除的核心方法**：连问 5 个 Why 找到根因
5. **架构评分自动化**：改了关键文件就触发审查，分数 < 70 就报警
6. **容量规划矩阵**：从 Prototype（1-100 用户）到 Enterprise（100K+），每级有明确的架构变更
7. **5 阶段路线图**：Week 1 的简单通知 → Month 4+ 的企业级系统，循序渐进
8. **20 条要点是全书精华浓缩**：涵盖策略、安全、工具、学习、架构、质量、Agent 等全维度
