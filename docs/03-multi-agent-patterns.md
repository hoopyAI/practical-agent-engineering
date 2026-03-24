# 03 — Multi-Agent Patterns

## Overview

When a single Agent handles everything, the prompt grows complex and capabilities get diluted. Multi-Agent architectures split tasks across specialized Agents, like going from a monolith to microservices.

But "multi-Agent" is not one pattern. It is **a family of four distinct approaches**. They solve different problems and have different complexity levels. Pick what you need.

## Four Patterns at a Glance

```
Multi-Agent
|-- Subagent              <-- Boss sends employee on a task; employee reports back
|-- Pipeline (Chain)      <-- A finishes, passes to B, B passes to C
|-- Swarm (Handoff)       <-- Dynamic handoff of control; disband when done
+-- Teams                 <-- Peers communicate and coordinate directly
```

| | Subagent | Pipeline | Swarm | Teams |
|---|---------|----------|-------|-------|
| **Communication** | Reports upward only | Sequential output passing | Explicit handoff | Peer-to-peer messaging |
| **Lifecycle** | Destroyed on completion | Follows the pipeline | Dynamically spawned, disbanded when done | Persistent |
| **Coordination** | Main Agent coordinates | Fixed order | Handoff functions | Shared task list |
| **State** | Not shared | Output passed along | Stateless; context passed at handoff | Shared |
| **Complexity** | Low | Low | Medium | High |
| **Typical impl.** | Claude Code Subagent | LangGraph linear graph | OpenAI Swarm/Agents SDK | Claude Code Agent Teams |

## Pattern 1: Subagent (Foundational, Must Learn)

The main Agent spawns a child Agent, gives it a clear task, and the child reports back. Subagents are unaware of each other's existence.

```
Main Agent
|-- spawn -> Subagent A (research)   -> return result
|-- spawn -> Subagent B (write code) -> return result
+-- Aggregate A + B results -> final output
```

**How Claude Code implements it:**
- Subagent runs within the same session
- Has independent context; does not inherit the main Agent's conversation history
- Can only report to the main Agent; cannot communicate with other subagents
- If you use Claude Code, you use this daily. The Agent tool is a Subagent.

**When to use:**
- Tasks that split cleanly into independent subtasks
- Subtasks don't need to coordinate with each other
- The most common and practical multi-Agent pattern

**Learning path:**
- learn-claude-code s04 (Subagent chapter)
- Focus on understanding the isolation mechanism and context passing

## Pattern 2: Pipeline / Chain (Most Practical, Must Learn)

Agents are connected in sequence. A's output becomes B's input. Like a factory assembly line.

```
Research Agent -> Writer Agent -> Review Agent -> final output
     |                |              |
     +-- findings --> +-- draft -->  +-- feedback -> loop or done
```

**LangGraph is the best implementation:**

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

**What it gives you:**
- Stateful: each step's results are saved in a shared State
- Conditional branching: review failure can loop back to write
- Persistence: interrupted tasks can resume from a checkpoint
- Observable: every step has a trace

LangGraph is overkill for simple linear chains. If you don't need conditional branching or checkpointing, just call functions in sequence.

**When to use:**
- Steps have a clear sequence
- Conditional branching is needed (pass/fail/redo)
- Production environments requiring reliability and recoverability

**Learning path:**
- [LangGraph Official Tutorials](https://langchain-ai.github.io/langgraph/tutorials/)
- Practice: re-implement a multi-stage workflow using LangGraph

## Pattern 3: Swarm / Handoff (Understand the Concept)

Agents pass control to each other via **explicit handoff functions**. Not a fixed sequence; the next handler is decided dynamically based on context.

```
User -> Triage Agent
         |
         |-- handoff_to(sales_agent)     # if it's a sales question
         |-- handoff_to(support_agent)   # if it's a support question
         +-- handoff_to(tech_agent)      # if it's a technical question
```

**History:**
- OpenAI released [Swarm](https://github.com/openai/swarm) in 2024 (experimental, a few hundred lines of code)
- Core concepts: only **Agent** (instructions + tools) and **Handoff** (control transfer)
- Now superseded by the **OpenAI Agents SDK** (Swarm is no longer maintained)
- Swarm's handoff design was influential. The Agents SDK inherits it.

**Swarm's source code is worth reading.** Minimalist design that explains the handoff pattern in a few hundred lines. Better for learning than the Agents SDK.

**When to use:**
- Customer-service-style routing (route by type to different specialists)
- No parallelism needed, but flexible control flow required
- Similar to Routing, but more dynamic

**Learning path:**
- Read the [OpenAI Swarm source code](https://github.com/openai/swarm) (a few hours)
- Then see how [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) inherited the design

## Pattern 4: Agent Teams (Frontier, Awareness Level)

Multiple Agent instances run persistently, share a task list, and can send messages directly to each other. Like a real team.

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

**Claude Code Agent Teams (released Feb 2026, experimental):**
- 2-16 independent Claude Code instances
- One Leader assigns tasks; Workers claim and execute
- Workers communicate directly (no Leader relay needed)
- Each Worker has an independent context window
- Shared codebase and task list

**How it differs from Subagent:**

| | Subagent | Agent Teams |
|---|---------|-------------|
| Communication | Reports upward only | Peer-to-peer messaging |
| Coordination | Main Agent is the only relay | Workers self-coordinate |
| Best for | Independent subtasks | Parallel tasks needing coordination |

**When to use:**
- Tasks that need multiple Agents to share findings
- Parallel execution requiring coordination
- Warning: high token consumption, not suited for simple tasks

**When NOT to use:**
- Sequential tasks -> use Pipeline
- Independent subtasks -> use Subagent
- Editing the same file -> single Agent is better

**Learning path:**
- [Claude Code Agent Teams Official Docs](https://code.claude.com/docs/en/agent-teams)
- [Addy Osmani: Claude Code Swarms](https://addyosmani.com/blog/claude-code-agent-teams/)
- [Claude Code Agent Teams Complete Guide](https://claudefa.st/blog/guide/agents/agent-teams)

## A Note on the A2A Protocol

A2A (Agent-to-Agent Protocol) is a Google-initiated standard for inter-Agent communication, aiming to let Agents across different platforms discover and communicate with each other.

**Current status (March 2026):**
- Spec is still a draft; breaking changes are possible at any time
- OpenAI and Microsoft have not publicly endorsed it
- Real-world usage is mainly in enterprise supply chain scenarios
- Community take: it adds complexity on top of MCP with insufficient short-term payoff

**Verdict: understand the concept, but don't invest heavily yet.** For now, multi-Agent communication via shared state / files / internal message channels is sufficient. Wait until the spec stabilizes and major vendors support it.

If you want to understand A2A conceptually:
- [Google: Developer's Guide to AI Agent Protocols](https://developers.googleblog.com/developers-guide-to-ai-agent-protocols/)
- [A2A Protocol Stack](https://subhadipmitra.com/blog/2026/agent-protocol-stack/)

## OpenClaw's Multi-Agent Architecture (Reference)

OpenClaw uses a hybrid approach:

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

This is a **Subagent + Routing** combination, using the Gateway for routing and isolation.

References:
- [OpenClaw Multi-Agent Routing](https://docs.openclaw.ai/concepts/multi-agent)
- [OpenClaw Multi-Agent Deep Dive](https://dev.to/leowss/i-built-a-team-of-36-ai-agents-heres-exactly-how-openclaw-works-2eab)

## General Resources

| Resource | Description |
|----------|-------------|
| [Agent Swarm vs Workflows vs LangGraph](https://softmaxdata.com/blog/agent-architectures-compared/) | Three architectures compared |
| [Agent Swarms: Multi-Agent Networks Guide](https://teammates.ai/agent-swarms) | Swarm pattern overview |
| [Google Research: Scaling Agent Systems](https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/) | Multi-Agent scalability research |
| [IBM: What is a Multi-Agent System](https://www.ibm.com/think/topics/multiagent-system) | Concept introduction |
| [Claude Code Teams and Swarms: Architecture](https://decodeclaude.com/teams-and-swarms/) | Claude Code multi-Agent architecture analysis |

## Practice Plan

```
Week 1: learn-claude-code s04 (Subagent) + s09 (Agent Teams)
Week 2: Read OpenAI Swarm source code; understand the handoff concept
Week 2: LangGraph tutorial, build a Pipeline (Research -> Write -> Review)
Week 3-4: Rebuild a multi-stage project as a multi-Agent Pipeline using LangGraph
```
