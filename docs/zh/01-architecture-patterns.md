# 01 — 架构模式

## 概览

Agent 不是某一种固定的模式，而是一组可组合的架构模式。你应该能在白板上画出每种模式的流程，并说清楚什么场景该用哪种。

## 五种核心模式

### 1. ReAct（推理 + 行动）

最经典的推理循环模式。Agent 在 "思考 -> 行动 -> 观察" 之间反复迭代。

```
User request -> Think (which tool?) -> Act (call API) ->
Observe (get result) -> Think (enough info?) -> ... -> Return answer
```

- **优点**：灵活，适合动态任务
- **缺点**：每轮循环都烧 token，成本和延迟难以预估
- **适用场景**：需要反复试探和修正的探索型任务

### 2. Plan-and-Execute

把 "规划" 和 "执行" 拆成两个独立模块。Planner 用大模型做高层规划，Executor 用小模型逐步执行。

```
User request -> Planner (break into N steps) ->
Executor runs step 1 -> step 2 -> ... ->
Re-planner (adjust based on intermediate results) -> Continue
```

- **数据**：相比纯 ReAct，任务完成率达 92%，速度快 3.6 倍，token 消耗更低
- **适用场景**：步骤可预测的多步任务

### 3. Reflection

Agent 先生成结果，再自我检查。发现问题就重新生成。

```
Generate answer -> Self-check (hallucination? logic correct?) ->
Found issue -> Regenerate -> Check again -> OK
```

- **代价**：token 消耗增加 2-3 倍
- **适用场景**：对质量要求很高的场景

### 4. Routing

根据输入类型把请求路由到不同的专用 Agent，类似微服务的 API Gateway。

- **适用场景**：输入类型可枚举（客服分类、多语言处理等）

### 5. Orchestrator-Workers

一个编排器动态拆解任务，分配给多个 Worker 执行。不需要预定义步骤。

- **Anthropic 的编码 Agent 就是用这个模式处理 GitHub issue 的**
- **适用场景**：复杂度不可预测的任务

## 模式选择指南

| 场景 | 推荐模式 | 原因 |
|------|---------|------|
| 动态探索 | ReAct | 灵活迭代 |
| 步骤清晰的多步任务 | Plan-and-Execute | 效率高 |
| 质量敏感 | Reflection | 自我检查提升质量 |
| 输入分类 | Routing | 专业化处理 |
| 复杂度不可预测 | Orchestrator-Workers | 动态分工 |
| 成本敏感 | Plan-and-Execute + 小模型 Executor | 大模型规划，小模型执行 |

## 源码学习项目

### learn-claude-code：从零复刻 Claude Code 架构

- **仓库**：https://github.com/shareAI-lab/learn-claude-code
- **语言**：有中文文档
- **核心思路**：模型就是 Agent，代码是 Harness
- **12 节课**：
  - s01：Agent Loop
  - s02：Tool Use
  - s03：Context Isolation
  - s04：Subagent
  - s05-s08：任务系统、依赖图、权限治理
  - s09：Agent Teams（多 Agent 协作）
  - s10-s12：Worktree 隔离、并行执行
- **建议**：花 1-2 周时间跟着把每个模块自己动手实现一遍

### Nanobot：4000 行 Python 复刻 OpenClaw 核心

- **仓库**：https://github.com/HKUDS/nanobot
- **Stars**：26,800+
- **亮点**：OpenClaw 有 43 万行代码，Nanobot 只用 4000 行就实现了核心功能
- **包含**：持久化记忆（本地知识图谱）、工具调用、后台 Agent、模型无关
- **建议**：花 2-3 天通读全部源码，搞清楚一个最小可行 Agent 产品的全貌

### 其它参考

| 项目 | Stars | 说明 |
|------|-------|------|
| [ZeroClaw](https://pinggy.io/blog/zeroclaw_lightweight_openclaw_alternative/) | 21K+ | 轻量级 OpenClaw 替代品，MIT 协议 |
| [OpenCode](https://github.com/opencode-ai/opencode) | 112K+ | 最热门的开源 AI 编码 Agent，Go 语言编写 |
| [SuperClaude](https://github.com/SuperClaude-Org/SuperClaude_Framework) | 20K+ | Claude Code 增强框架 |

## 补充：结构化输出

让 Agent 输出可靠的 JSON/schema，而不是自由文本。这是工具调用和多 Agent 通信的基础——如果 Agent 返回格式不靠谱，下游的一切都会崩。

- Anthropic/OpenAI 都支持 `tool_use` / `function_calling` 来强制 JSON 输出
- 复杂场景下，用 Pydantic/Zod schema 约束输出结构
- 和 ReAct 模式结合时：每个 "action" 步骤必须是结构化的工具调用，不能是自由文本

## 补充：错误处理与可靠性

Agent 上生产环境最常踩的坑：

| 问题 | 症状 | 应对策略 |
|------|------|---------|
| **幻觉循环** | 反复执行同一步骤却意识不到 | 步数上限 + 重复检测 |
| **错误级联** | 一步出错，后面全跟着错 | 逐步校验 + 回滚机制 |
| **成本失控** | 重试循环不停烧钱 | Token 预算 + 超时控制 |
| **进度丢失** | 长任务跑到一半崩了 | Checkpoint + 恢复机制（LangGraph 原生支持） |

## 必读材料

| 资源 | 说明 |
|------|------|
| [Anthropic: Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) | 架构模式的圣经级文档 |
| [Anthropic PDF: Architecture Patterns & Implementation](https://resources.anthropic.com/hubfs/Building%20Effective%20AI%20Agents-%20Architecture%20Patterns%20and%20Implementation%20Frameworks.pdf) | 完整 PDF 版 |
| [Google Cloud: Choose a Design Pattern for Agentic AI](https://docs.google.com/architecture/choose-design-pattern-agentic-ai-system) | Google 视角的模式选择 |
| [Redis: AI Agent Architecture Patterns](https://redis.io/blog/ai-agent-architecture-patterns/) | 单 Agent + 多 Agent 模式详解 |
| [ReAct vs Plan-and-Execute Deep Dive](https://louisbouchard.substack.com/p/react-vs-plan-and-execute-the-architecture) | 两种核心模式的深度对比 |
| [15 Agentic AI Design Patterns (2026)](https://aitoolsclub.com/15-agentic-ai-design-patterns-you-should-know-research-backed-and-emerging-frameworks-2026/) | 15 种模式大全 |
| [AI Agent Architecture: Build Systems That Work (Redis)](https://redis.io/blog/ai-agent-architecture/) | 面向生产环境的架构指南 |
| [Anthropic Cookbook: Agent Patterns](https://github.com/anthropics/anthropic-cookbook/tree/main/patterns/agents) | 代码级别的实现参考 |

## 实践计划

1. **第 1 周**：读 Anthropic 的 Building Effective Agents + Google 设计模式指南
2. **第 1-2 周**：跟着 learn-claude-code 的 12 节课，端到端搭建一个 Agent Harness
3. **第 2 周**：通读 Nanobot 源码，理解最小可行 Agent 的全貌
4. **第 3 周**：在已有项目中加入 Reflection 模式（自我检查 prompt 质量）
5. **持续**：每种模式都写成博客或技术笔记
