# Practical Agent Engineering

> A structured, opinionated guide to becoming an AI Agent engineer in 2026.
> Not a link dump — a learning path with architecture patterns, practice plans, and real-world advice.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

**English | [中文](README.zh.md)**

## Why This Guide

Most "awesome" lists give you 200 links and no direction. This guide gives you:
- **A learning path** — what to learn and in what order
- **Opinionated recommendations** — not every tool, just the ones that matter
- **Practice plans** — concrete exercises, not just reading lists
- **Architecture-first thinking** — patterns before frameworks

## Learning Map

<p align="center">
  <img src="assets/learning-map.svg" alt="Learning Map" width="800" />
</p>

## Learning Strategy

**Start with a complete project, then go deep on each topic.**

### Step 1: Build an Agent from Scratch (3-4 weeks)

Two projects that teach you everything at once:

| Project | What You'll Learn | Time |
|---------|------------------|------|
| [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) | 12-lesson course: agent loop, tools, context, subagents, teams, permissions | 2-3 weeks |
| [Nanobot](https://github.com/HKUDS/nanobot) | Read a complete agent in 4,000 lines: memory, tools, multi-model support | 2-3 days |

### Step 2: Go Deep on Each Topic

| # | Topic | Covered by Step 1? | What to learn next | Guide |
|---|-------|--------------------|--------------------|-------|
| 1 | Architecture Patterns | Agent Loop basics | ReAct vs Plan-Execute, structured output, reliability | [EN](docs/01-architecture-patterns.md) · [中文](docs/zh/01-architecture-patterns.md) |
| 2 | Context Engineering | Context isolation | Hierarchical memory, token budgets, Agentic RAG | [EN](docs/02-context-engineering.md) · [中文](docs/zh/02-context-engineering.md) |
| 3 | Multi-Agent Patterns | Subagent, Teams | Pipeline/LangGraph, Swarm/Handoff | [EN](docs/03-multi-agent-patterns.md) · [中文](docs/zh/03-multi-agent-patterns.md) |
| 4 | Security & Sandboxing | Permission governance | Firecracker/gVisor, OWASP, HITL → HOTL | [EN](docs/04-security-sandbox.md) · [中文](docs/zh/04-security-sandbox.md) |
| 5 | Eval & Observability | Not covered | Evals, LLM-as-Judge, tracing, cost monitoring | [EN](docs/05-eval-and-observability.md) · [中文](docs/zh/05-eval-and-observability.md) |
| 6 | Deployment | Not covered | Docker, platforms, vLLM, hybrid routing | [EN](docs/06-deployment.md) · [中文](docs/zh/06-deployment.md) |

### Quick Framework Reference

| Framework | Use For | Docs |
|-----------|---------|------|
| [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) | Agent implementation, sandbox execution | [TS](https://github.com/anthropics/claude-agent-sdk-typescript) / [Python](https://github.com/anthropics/claude-agent-sdk-python) |
| [LangGraph](https://langchain-ai.github.io/langgraph/) | Stateful multi-agent pipelines | [Tutorials](https://langchain-ai.github.io/langgraph/tutorials/) |
| [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) | Quick prototyping, Swarm/Handoff | [GitHub](https://github.com/openai/openai-agents-python) |
| [Langfuse](https://langfuse.com) | Eval + observability (open source) | [Docs](https://langfuse.com/docs) |
| [DeepEval](https://deepeval.com) | Local eval + CI/CD | [Docs](https://docs.confident-ai.com/) |

## Timeline

| Month | What You're Doing | What You'll Have |
|-------|------------------|-----------------|
| **1-2** | learn-claude-code + Nanobot | Understanding of agent internals |
| **2-3** | Architecture deep dive + LangGraph | Your first multi-agent pipeline |
| **3-4** | Security + Eval | Sandbox, permission model, automated evals |
| **5-6** | Deployment + Observability | A deployed agent with monitoring and cost tracking |

## Who This Is For

- Software engineers transitioning into AI/Agent engineering
- Developers who have used AI coding tools (Claude Code, Cursor, Copilot) and want to understand how they work
- Engineers who want to build production-grade agents, not just demos

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). PRs for fixing outdated links, adding resources, and improving explanations are welcome.

## License

[MIT](LICENSE)
