# 02 — Context Engineering

## Overview

Context Engineering is the discipline of systematically managing an Agent's context: deciding what information enters the context window, in what form, and when. In under a year, it went from a niche concept to a core AI engineering discipline in 2026.

**Agent performance is 90% determined by context quality.**

## Why This Is the Hottest Direction in 2026

- The Manus team shared that they rebuilt their Agent framework 4 times. The core challenge every time was context management.
- A single complex tool schema can eat 500+ tokens
- Connecting multiple MCP Servers can consume 50,000+ tokens before reasoning even begins
- MCP has 97M+ monthly downloads, so more tool connections means the context explosion problem keeps getting worse

## Core Areas

### 1. Context Budget Management

You have a fixed token window (e.g. 200K). How do you allocate it?

```
Total budget 200K tokens
|-- System instructions    ~2K    (1%)
|-- Tool schemas           ~10-50K (5-25%)  <-- most likely to explode
|-- Conversation history   ~20-50K (10-25%)
|-- Retrieved knowledge    ~20-50K (10-25%)
+-- Space for reasoning    remainder (at least 50%)
```

Best practice: keep simultaneously exposed tools under 20. Use tool search to lazy-load infrequently used tools.

### 2. Hierarchical Memory Architecture

Like an operating system's memory hierarchy:

| Layer | Analogy | Capacity | Speed | Content | Implementation |
|-------|---------|----------|-------|---------|---------------|
| Working memory | CPU registers | Very small | Fastest | What the current step needs | Directly in the prompt |
| Short-term memory | RAM | Medium | Fast | Current conversation context | Sliding window / summarization |
| Long-term memory | Disk | Very large | Slow | User preferences, history | Vector DB / knowledge graph |

### 3. Tool Schema Compression

- Simplify parameter descriptions. Good enough is good enough; don't write essays.
- Use enums to constrain values: reduces Agent errors and description length
- Grouped loading: only load tool subsets relevant to the current task type
- Cache hot schemas: cache parsing results for high-frequency tools

### 4. Agentic Context Engineering (ACE)

Self-evolving context. The Agent updates its own context templates based on execution feedback. Information that proves useful is retained; redundant information is pruned. Think of it as a continuously self-optimizing playbook.

### 5. Agentic Graph RAG

Agent reasoning + knowledge graphs. Not just vector search, but reasoning and traversal over structured knowledge graphs.

## Memory Framework Comparison (2026)

| Framework | Strengths | Best For |
|-----------|-----------|----------|
| **Mem0** | Most mature: 26% accuracy increase, 91% latency decrease, 90% token decrease | Production-grade long-term memory |
| **Letta** | Mimics OS memory hierarchy (main context = RAM, external storage = disk) | Scenarios requiring fine-grained control |
| **LangChain Memory** | Built-in multiple memory modes | LangChain ecosystem users |
| **Redis** | Fast read/write, ideal for short-term memory | High-frequency interaction scenarios |
| **Zep** | Focused on conversational memory | Chat-based Agents |

## Essential Reading

| Resource | Description |
|----------|-------------|
| [Manus: Context Engineering for AI Agents](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) | **Start here.** Real-world lessons from rebuilding their framework 4 times |
| [Martin Fowler: Context Engineering for Coding Agents](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html) | A software engineering perspective |
| [State of Context Engineering 2026](https://www.newsletter.swirlai.com/p/state-of-context-engineering-in-2026) | Industry overview |
| [arxiv: Context Engineering for AI Agents in OSS](https://arxiv.org/abs/2510.21413) | Academic paper |
| [arxiv: Agentic Context Engineering (ACE)](https://arxiv.org/abs/2510.04618) | ACE methodology paper |
| [Mem0: Production-Ready Long-Term Memory](https://arxiv.org/pdf/2504.19413) | Mem0 technical paper |
| [IBM: What Is AI Agent Memory](https://www.ibm.com/think/topics/ai-agent-memory) | Concept introduction |
| [6 Best AI Agent Memory Frameworks 2026](https://machinelearningmastery.com/the-6-best-ai-agent-memory-frameworks-you-should-try-in-2026/) | Framework comparison |
| [AWS AgentCore: Long-Term Memory Deep Dive](https://aws.amazon.com/blogs/machine-learning/building-smarter-ai-agents-agentcore-long-term-memory-deep-dive/) | AWS production practices |
| [Redis: Build AI Agents with Memory Management](https://redis.io/blog/build-smarter-ai-agents-manage-short-term-and-long-term-memory-with-redis/) | Redis hands-on tutorial |
| [Agent Memory Paper List (GitHub)](https://github.com/Shichun-Liu/Agent-Memory-Paper-List) | Academic paper collection |

## Agentic RAG: Combining Agents with Retrieval

RAG hasn't become obsolete in 2026. It has evolved. Traditional RAG is a fixed pipeline (query -> retrieve -> generate). Agentic RAG lets the Agent autonomously decide when to retrieve, what to retrieve, and whether to do multiple retrieval rounds.

| | Traditional RAG | Agentic RAG |
|---|----------------|-------------|
| Flow | Fixed: query -> retrieve -> generate | Agent decides autonomously |
| Flexibility | Static | Dynamic query rewriting, data source switching, sufficiency checks |
| Suited for | Simple Q&A | Complex multi-step reasoning + knowledge-intensive tasks |

**The thing to learn is not "how to build a RAG pipeline" but "how to make an Agent use retrieval intelligently."**

### Resources

| Resource | Description |
|----------|-------------|
| [IBM: What is Agentic RAG](https://www.ibm.com/think/topics/agentic-rag) | Concept introduction |
| [Weaviate: What Is Agentic RAG](https://weaviate.io/blog/what-is-agentic-rag) | Architecture deep dive |
| [arxiv: Agentic RAG Survey](https://arxiv.org/abs/2501.09136) | Academic survey |
| [ByteByteGo: MCP vs RAG vs AI Agents](https://blog.bytebytego.com/p/ep202-mcp-vs-rag-vs-ai-agents) | Comparison of all three |

## Practice Plan

1. **Week 1**: Read the Manus blog post + Martin Fowler article; internalize core concepts
2. **Week 1**: Calculate the token distribution of a real Agent call; find where most tokens go
3. **Week 2**: Add long-term memory to a project using Mem0 (remember user preferences)
4. **Week 2**: Try tool schema compression. Simplify descriptions and measure impact on Agent accuracy.
5. **Week 3**: Design a context self-evolution mechanism where the Agent automatically prunes useless context
