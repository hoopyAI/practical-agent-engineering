# 03 — 多 Agent 模式

## 概览

一个 Agent 什么都干的时候，prompt 会越来越复杂，能力也被稀释。多 Agent 架构把任务拆给专门的 Agent 去做，有点像从单体服务演进到微服务。

但 "多 Agent" 不是一种模式，而是**四种不同方法的统称**。它们解决的问题不同，复杂度也不同。按需选择就好。

## 四种模式一览

```
Multi-Agent
|-- Subagent              <-- Boss sends employee on a task; employee reports back
|-- Pipeline (Chain)      <-- A finishes, passes to B, B passes to C
|-- Swarm (Handoff)       <-- Dynamic handoff of control; disband when done
+-- Teams                 <-- Peers communicate and coordinate directly
```

| | Subagent | Pipeline | Swarm | Teams |
|---|---------|----------|-------|-------|
| **通信方式** | 只向上汇报 | 按顺序传递输出 | 显式 Handoff | 点对点消息 |
| **生命周期** | 完成即销毁 | 跟随流水线 | 动态创建，完成即解散 | 持久运行 |
| **协调方式** | 主 Agent 统一调度 | 固定顺序 | Handoff 函数 | 共享任务列表 |
| **状态** | 不共享 | 输出逐步传递 | 无状态，Handoff 时传递上下文 | 共享 |
| **复杂度** | 低 | 低 | 中 | 高 |
| **典型实现** | Claude Code Subagent | LangGraph 线性图 | OpenAI Swarm/Agents SDK | Claude Code Agent Teams |

## 模式 1：Subagent（基础，必学）

主 Agent 生成一个子 Agent，给它一个明确的任务，子 Agent 完成后汇报结果。各个 Subagent 之间互不知晓。

```
Main Agent
|-- spawn -> Subagent A (research)   -> return result
|-- spawn -> Subagent B (write code) -> return result
+-- Aggregate A + B results -> final output
```

**Claude Code 的实现方式：**
- Subagent 在同一个 session 中运行
- 有独立的上下文，不继承主 Agent 的对话历史
- 只能向主 Agent 汇报，不能和其他 Subagent 通信
- 如果你用 Claude Code，这是你每天都在用的模式——Agent tool 就是一个 Subagent

**什么时候用：**
- 任务能干净地拆成独立子任务
- 子任务之间不需要协调
- 最常见、也是最实用的多 Agent 模式

**学习路径：**
- learn-claude-code s04（Subagent 章节）
- 重点理解隔离机制和上下文传递

## 模式 2：Pipeline / Chain（最实用，必学）

Agent 按顺序串联。A 的输出是 B 的输入。就像工厂的流水线。

```
Research Agent -> Writer Agent -> Review Agent -> final output
     |                |              |
     +-- findings --> +-- draft -->  +-- feedback -> loop or done
```

**LangGraph 是最好的实现方式：**

```python
graph = StateGraph(State)
graph.add_node("research", research_agent)
graph.add_node("write", writer_agent)
graph.add_node("review", reviewer_agent)

graph.add_edge("research", "write")
graph.add_edge("write", "review")
graph.add_conditional_edges("review", should_revise, {
    "revise": "write",   # Not satisfied -> go back and rewrite
    "done": END,         # Satisfied -> finish
})
```

**这给你带来什么：**
- 有状态：每一步的结果都保存在共享的 State 中
- 条件分支：审查不通过可以回退到写作环节
- 持久化：中断的任务可以从 Checkpoint 恢复
- 可观测：每一步都有 trace

对于简单的线性链，LangGraph 杀鸡用牛刀了。如果不需要条件分支或 Checkpoint，直接按顺序调函数就行。

**什么时候用：**
- 步骤有明确的先后顺序
- 需要条件分支（通过/失败/重做）
- 生产环境需要可靠性和可恢复性

**学习路径：**
- [LangGraph Official Tutorials](https://langchain-ai.github.io/langgraph/tutorials/)
- 练习：用 LangGraph 重新实现一个多阶段工作流

## 模式 3：Swarm / Handoff（了解概念即可）

Agent 之间通过**显式的 Handoff 函数**传递控制权。不是固定顺序，而是根据上下文动态决定下一个处理者。

```
User -> Triage Agent
         |
         |-- handoff_to(sales_agent)     # if it's a sales question
         |-- handoff_to(support_agent)   # if it's a support question
         +-- handoff_to(tech_agent)      # if it's a technical question
```

**历史脉络：**
- OpenAI 在 2024 年发布了 [Swarm](https://github.com/openai/swarm)（实验性质，几百行代码）
- 核心概念就两个：**Agent**（指令 + 工具）和 **Handoff**（控制权转移）
- 现在已经被 **OpenAI Agents SDK** 取代（Swarm 不再维护）
- Swarm 的 Handoff 设计影响很大，Agents SDK 继承了这一理念

**Swarm 的源码值得一读。** 极简设计，几百行代码就把 Handoff 模式讲透了。比 Agents SDK 更适合学习。

**什么时候用：**
- 客服式路由（按类型分配给不同专员）
- 不需要并行，但需要灵活的控制流
- 和 Routing 类似，但更动态

**学习路径：**
- 读 [OpenAI Swarm 源码](https://github.com/openai/swarm)（几个小时）
- 然后看 [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) 如何继承了这套设计

## 模式 4：Agent Teams（前沿，了解即可）

多个 Agent 实例持久运行，共享任务列表，可以直接互相发消息。就像一个真正的团队。

```
Leader Agent (assigns tasks)
|-- Worker A (claims task 1, executes)
|-- Worker B (claims task 2, executes)
|-- Worker C (claims task 3, executes)
|
Workers can message each other:
  A -> B: "I found a bug, check your side too"
  B -> A: "Confirmed, same issue here"
```

**Claude Code Agent Teams（2026 年 2 月发布，实验性质）：**
- 2-16 个独立的 Claude Code 实例
- 一个 Leader 分配任务，Worker 认领并执行
- Worker 之间直接通信（不需要 Leader 中转）
- 每个 Worker 有独立的上下文窗口
- 共享代码库和任务列表

**和 Subagent 的区别：**

| | Subagent | Agent Teams |
|---|---------|-------------|
| 通信方式 | 只向上汇报 | 点对点消息 |
| 协调方式 | 主 Agent 是唯一中转 | Worker 之间自行协调 |
| 适用场景 | 独立子任务 | 需要协调的并行任务 |

**什么时候用：**
- 任务需要多个 Agent 共享发现
- 并行执行但需要协调
- 注意：token 消耗很高，简单任务不划算

**什么时候不该用：**
- 顺序型任务 -> 用 Pipeline
- 独立子任务 -> 用 Subagent
- 编辑同一个文件 -> 单 Agent 更好

**学习路径：**
- [Claude Code Agent Teams Official Docs](https://code.claude.com/docs/en/agent-teams)
- [Addy Osmani: Claude Code Swarms](https://addyosmani.com/blog/claude-code-agent-teams/)
- [Claude Code Agent Teams Complete Guide](https://claudefa.st/blog/guide/agents/agent-teams)

## 关于 A2A 协议

A2A（Agent-to-Agent Protocol）是 Google 牵头的 Agent 间通信标准，目标是让不同平台的 Agent 能互相发现和通信。

**现状（2026 年 3 月）：**
- 规范还是草案状态，随时可能有破坏性变更
- OpenAI 和 Microsoft 都没有公开表态支持
- 实际落地主要在企业供应链场景
- 社区看法：在 MCP 之上又加了一层复杂度，短期回报不够

**判断：了解概念就行，不值得重度投入。** 目前多 Agent 通信通过共享状态 / 文件 / 内部消息通道就够了。等规范稳定、主要厂商跟进之后再说。

如果想了解 A2A 的概念：
- [Google: Developer's Guide to AI Agent Protocols](https://developers.googleblog.com/developers-guide-to-ai-agent-protocols/)
- [A2A Protocol Stack](https://subhadipmitra.com/blog/2026/agent-protocol-stack/)

## OpenClaw 的多 Agent 架构（参考）

OpenClaw 用的是混合方案：

```
Gateway (daemon process)
|-- Persistent Agent A (long-lived, bound to a Slack channel)
|-- Persistent Agent B (long-lived, bound to Discord)
|-- Sub-agent C (spawned by Agent A for a temporary task)
+-- Sub-agent D (spawned by Agent B for background search)

Each Agent is fully isolated:
- Independent workspace, session, permissions
- Independent auth (credentials are not shared)
- Independent model selection (can use different models)
```

这是 **Subagent + Routing** 的组合，用 Gateway 做路由和隔离。

参考链接：
- [OpenClaw Multi-Agent Routing](https://docs.openclaw.ai/concepts/multi-agent)
- [OpenClaw Multi-Agent Deep Dive](https://dev.to/leowss/i-built-a-team-of-36-ai-agents-heres-exactly-how-openclaw-works-2eab)

## 通用资源

| 资源 | 说明 |
|------|------|
| [Agent Swarm vs Workflows vs LangGraph](https://softmaxdata.com/blog/agent-architectures-compared/) | 三种架构横向对比 |
| [Agent Swarms: Multi-Agent Networks Guide](https://teammates.ai/agent-swarms) | Swarm 模式概览 |
| [Google Research: Scaling Agent Systems](https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/) | 多 Agent 可扩展性研究 |
| [IBM: What is a Multi-Agent System](https://www.ibm.com/think/topics/multiagent-system) | 概念入门 |
| [Claude Code Teams and Swarms: Architecture](https://decodeclaude.com/teams-and-swarms/) | Claude Code 多 Agent 架构分析 |

## 实践计划

```
Week 1: learn-claude-code s04 (Subagent) + s09 (Agent Teams)
Week 2: Read OpenAI Swarm source code; understand the handoff concept
Week 2: LangGraph tutorial, build a Pipeline (Research -> Write -> Review)
Week 3-4: Rebuild a multi-stage project as a multi-Agent Pipeline using LangGraph
```
