# 01 — Agent Architecture Patterns

## Overview

An Agent is not a single pattern — it is a set of composable architecture patterns. You should be able to whiteboard each pattern's flow and explain which scenario calls for which pattern.

## Five Core Patterns

### 1. ReAct (Reason + Act)

The standard reasoning-loop pattern. The Agent iterates through a "Think -> Act -> Observe" cycle.

```
User request -> Think (which tool?) -> Act (call API) ->
Observe (get result) -> Think (enough info?) -> ... -> Return answer
```

- **Pros**: Flexible, handles dynamic tasks
- **Cons**: Each loop iteration burns tokens; cost and latency are unpredictable
- **Best for**: Exploratory tasks requiring iterative refinement

### 2. Plan-and-Execute

Separates "planning" and "execution" into two independent modules. The Planner uses a large model for high-level planning; the Executor uses a smaller model to execute step by step.

```
User request -> Planner (break into N steps) ->
Executor runs step 1 -> step 2 -> ... ->
Re-planner (adjust based on intermediate results) -> Continue
```

- **Data**: 92% task completion rate vs pure ReAct, 3.6x faster, fewer tokens
- **Best for**: Multi-step tasks with predictable steps

### 3. Reflection

The Agent generates a result, then self-checks it. If problems are found, it regenerates.

```
Generate answer -> Self-check (hallucination? logic correct?) ->
Found issue -> Regenerate -> Check again -> OK
```

- **Cost**: 2-3x token consumption
- **Best for**: Scenarios with high quality requirements

### 4. Routing

Routes inputs to different specialized Agents based on type. Like a microservice API Gateway.

- **Best for**: Enumerable input types (customer service classification, multilingual processing)

### 5. Orchestrator-Workers

One orchestrator dynamically decomposes tasks and assigns them to multiple Workers. No predefined steps needed.

- **Anthropic's coding agent uses this pattern for handling GitHub issues**
- **Best for**: Tasks with unpredictable complexity

## Pattern Selection Guide

| Scenario | Recommended Pattern | Reason |
|----------|-------------------|--------|
| Dynamic exploration | ReAct | Flexible iteration |
| Multi-step tasks with clear steps | Plan-and-Execute | High efficiency |
| Quality-sensitive | Reflection | Self-check improves quality |
| Input classification | Routing | Specialized handling |
| Unpredictable complexity | Orchestrator-Workers | Dynamic division of labor |
| Cost-sensitive | Plan-and-Execute + small model Executor | Big model plans, small model executes |

## Source Code Study Projects (Key)

### learn-claude-code — Reproduce the Claude Code Architecture from Scratch

- **Repo**: https://github.com/shareAI-lab/learn-claude-code
- **Language**: Has Chinese documentation (有中文文档)
- **Core idea**: The model is the Agent; code is the Harness
- **12 lessons**:
  - s01: Agent Loop
  - s02: Tool Use
  - s03: Context Isolation
  - s04: Subagent
  - s05-s08: Task system, dependency graph, permission governance
  - s09: Agent Teams (multi-Agent coordination)
  - s10-s12: Worktree isolation, parallel execution
- **Recommendation**: Spend 1-2 weeks following along and implementing each module yourself

### Nanobot — 4,000 Lines of Python Reproducing OpenClaw's Core

- **Repo**: https://github.com/HKUDS/nanobot
- **Stars**: 26,800+
- **Highlight**: OpenClaw has 430K lines of code; Nanobot implements the core in 4,000 lines
- **Includes**: Persistent memory (local knowledge graph), tool calling, background Agent, model-agnostic
- **Recommendation**: Read the entire source code in 2-3 days to understand a minimum viable Agent product

### Other References

| Project | Stars | Description |
|---------|-------|-------------|
| [ZeroClaw](https://pinggy.io/blog/zeroclaw_lightweight_openclaw_alternative/) | 21K+ | Lightweight OpenClaw alternative, MIT license |
| [OpenCode](https://github.com/opencode-ai/opencode) | 112K+ | Most popular open-source AI coding agent, written in Go |
| [SuperClaude](https://github.com/SuperClaude-Org/SuperClaude_Framework) | 20K+ | Claude Code enhancement framework |

## Supplement: Structured Output

Making the Agent output reliable JSON/schema instead of free text. This is the foundation for tool calling and multi-Agent communication — if the Agent's return format is unreliable, everything downstream breaks.

- Anthropic/OpenAI both support `tool_use` / `function_calling` to force JSON output
- For complex scenarios, use Pydantic/Zod schemas to constrain output structure
- Combined with the ReAct pattern: each "action" step must be a structured tool call, not free text

## Supplement: Error Handling & Reliability

The most common production pitfalls for Agents:

| Problem | Symptom | Countermeasure |
|---------|---------|---------------|
| **Hallucination loops** | Repeats the same step without realizing | Step count limit + repetition detection |
| **Error cascading** | One wrong step corrupts everything after | Per-step validation + rollback mechanism |
| **Runaway costs** | Retry loops burning money | Token budget + timeout |
| **Lost progress** | Long task crashes mid-way | Checkpoint + recovery (LangGraph supports this natively) |

## Essential Reading

| Resource | Description |
|----------|-------------|
| [Anthropic: Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) | The architecture patterns bible |
| [Anthropic PDF: Architecture Patterns & Implementation](https://resources.anthropic.com/hubfs/Building%20Effective%20AI%20Agents-%20Architecture%20Patterns%20and%20Implementation%20Frameworks.pdf) | Full PDF version |
| [Google Cloud: Choose a Design Pattern for Agentic AI](https://docs.google.com/architecture/choose-design-pattern-agentic-ai-system) | Google's perspective on pattern selection |
| [Redis: AI Agent Architecture Patterns](https://redis.io/blog/ai-agent-architecture-patterns/) | Single Agent + multi-Agent pattern deep dive |
| [ReAct vs Plan-and-Execute Deep Dive](https://louisbouchard.substack.com/p/react-vs-plan-and-execute-the-architecture) | In-depth comparison of the two core patterns |
| [15 Agentic AI Design Patterns (2026)](https://aitoolsclub.com/15-agentic-ai-design-patterns-you-should-know-research-backed-and-emerging-frameworks-2026/) | Full landscape of 15 patterns |
| [AI Agent Architecture: Build Systems That Work (Redis)](https://redis.io/blog/ai-agent-architecture/) | Production-grade architecture guide |
| [Anthropic Cookbook: Agent Patterns](https://github.com/anthropics/anthropic-cookbook/tree/main/patterns/agents) | Code-level implementation reference |

## Practice Plan

1. **Week 1**: Read Anthropic's Building Effective Agents + Google design pattern guide
2. **Week 1-2**: Follow learn-claude-code's 12 lessons to build an agent harness end to end
3. **Week 2**: Read the Nanobot source code; understand a minimum viable Agent
4. **Week 3**: Add a Reflection pattern to an existing project (self-check prompt quality)
5. **Ongoing**: Write up each pattern as a blog post or technical note
