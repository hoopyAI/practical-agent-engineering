# 05 — Evaluation & Observability

## Overview

Evaluation (Eval) and Observability are two sides of the same coin:

- **Eval** answers: Is the Agent any good?
- **Observability** answers: What is the Agent doing, how much is it costing, and where did it go wrong?

You can't do eval without tracing (you won't know which step failed), and you can't do tracing without eval (you'll have a pile of data with no way to judge quality). Most tools (Langfuse, LangSmith, Braintrust) combine both anyway.

This is the area most Agent developers neglect. Getting it right is a strong differentiator.

---

## Part 1: Evaluation Methodology

### Anthropic's Evaluation Framework

Anthropic's January 2026 post [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) is the best introductory material available.

**Core terminology:**

| Concept | Meaning |
|---------|---------|
| **Task** | A test case with clear input and success criteria |
| **Trial** | A single attempt on a given task (model output is stochastic — multiple trials are needed) |
| **Grader** | The logic that scores the Agent's output (can be rule-based or LLM-based) |
| **Transcript** | The complete execution record (output, tool calls, reasoning, intermediate results) |

### Two Evaluation Dimensions

**Trajectory-level (process evaluation)** — *How* did the Agent reach the result?

| Metric | What It Measures | Example |
|--------|-----------------|---------|
| Tool selection accuracy | Did it use search when it should have? | Should have searched but hallucinated = 0 |
| Step efficiency | How many steps to complete? | 3-step job done in 10 steps |
| Reasoning chain quality | Is the logic coherent? | Hallucination or contradiction in the middle |
| Error recovery | Can it self-correct on failure? | Changed strategy after a tool call failed |
| Token efficiency | How many tokens spent? | Same result: plan A used 5K, plan B used 50K |

**Outcome-level (result evaluation)** — *Did* the task get done?

| Metric | What It Measures |
|--------|-----------------|
| Task completion rate | Did it succeed? |
| Result correctness | Was it right? |
| Latency | How fast? |
| Cost | How much money? |

- Development/debugging -> Trajectory (understand *why* it failed)
- Production monitoring -> Outcome (measure *business value*)
- You need both

### Anthropic's Practical Advice

1. **Start with 20-50 simple tasks** — early changes have outsized impact; small samples are enough
2. **Extract tasks from real failures** — not made up; scenarios that actually caused problems
3. **Evaluate results, not paths** — the Agent may find a better approach than you anticipated
4. **Practice eval-driven development** — define the target capability with an eval first, then iterate until the Agent meets it

### LLM-as-Judge (Core Method for Automated Evaluation)

Use another LLM to evaluate the Agent's output. The most mainstream automated evaluation method in 2026.

```
Input: task description + Agent output + scoring rubric + optional reference answer
  | Send to Judge LLM
Output: score per dimension (1-5) + justification (Chain-of-Thought)
```

**Key practices:**

| Practice | Why |
|----------|-----|
| Use integer scores (1-5) | LLMs are more consistent with discrete scores |
| Define each score clearly | "3 = completed with minor errors" beats "3 = average" |
| Judge writes reasoning before scoring (CoT) | Reliability improves 10-15% |
| Use a strong model as Judge | GPT-5 class models agree with human evaluators 80-90% |
| Have humans label a few dozen samples first | If humans disagree among themselves, the rubric needs work |

**Rubric example:**

```
Evaluation dimension: Task Completion

5 = Fully correct, nothing missing
4 = Mostly complete, minor imperfections that don't affect usability
3 = Main parts done, but obvious omissions or errors
2 = Only a small portion completed; core requirements unmet
1 = Completely failed or entirely wrong
```

---

## Part 2: Observability Fundamentals

### Full-Chain Tracing

Record every step of the Agent's execution as a trace tree:

```
Trace: "Write an article about RAG"
|-- Span: LLM reasoning (topic analysis)        [320 tokens, 1.2s]
|-- Span: Tool call (web_search)                 [0 tokens, 2.1s]
|   +-- Span: HTTP request                      [0.8s]
|-- Span: LLM reasoning (content structure)      [1,200 tokens, 3.4s]
|-- Span: Tool call (generate_image)             [0 tokens, 12s]
|   +-- Span: Gemini API call                   [11.5s]
|-- Span: LLM reasoning (quality check)          [800 tokens, 2.1s]
+-- Span: Tool call (save_file)                  [0 tokens, 0.1s]

Total: 2,320 tokens, 21.9s, $0.034
```

### Cost Monitoring

Agent costs are unpredictable — a "runaway" Agent in a retry loop can burn tens of dollars.

**Three-tier budget controls:**

| Level | Limit | Trigger |
|-------|-------|---------|
| Per-request | `max_tokens` | Truncate response |
| Per-task | Token budget (e.g. 50K) | Pause Agent, notify human |
| Daily/monthly | Total spend cap | 50% alert, 80% warning, 100% pause |

**Cost optimization strategies:**

| Strategy | Impact | How |
|----------|--------|-----|
| **Prompt caching** | Input cost -90%, latency -75% | Let the provider cache fixed system instructions |
| **Model routing** | Total cost -40-60% | Simple tasks use small models; complex tasks use large models |
| **Result caching** | Zero cost for repeat queries | Same input returns cached result |
| **Tool schema compression** | Input tokens -30-50% | Trim descriptions, lazy-load tools |

**Key principle: tag every token with metadata** — agent ID, task type, conversation thread.

### Quality & Anomaly Monitoring

| Dimension | How |
|-----------|-----|
| **Output quality** | Periodic sampling + LLM-as-Judge auto-scoring |
| **Anomaly detection** | Output format changes, token consumption spikes |
| **Regression detection** | Run eval baseline comparison after updates |
| **User feedback loop** | Annotation queue for domain experts to review |
| **Silent retries** | The number-one cause of cost spikes — Agents retrying quietly |

### OpenTelemetry: The Industry Standard

In March 2026, OpenTelemetry released **GenAI Semantic Conventions** — standardizing LLM span naming and attributes. Using OTel for tracing means you can plug into any compatible backend.

```python
# Conceptual example
with tracer.start_span("agent.run") as root:
    with tracer.start_span("llm.reasoning") as reasoning:
        reasoning.set_attribute("gen_ai.model", "claude-sonnet-4-6")
        reasoning.set_attribute("gen_ai.usage.input_tokens", 1200)
        reasoning.set_attribute("gen_ai.usage.output_tokens", 320)

    with tracer.start_span("tool.web_search") as tool:
        tool.set_attribute("tool.name", "web_search")
        tool.set_attribute("tool.status", "success")
```

---

## Part 3: Eval-Driven Development + CI/CD

Like TDD, but for Agents:

```
1. Write eval (define what the Agent should be able to do)
2. Run eval (Agent can't do it yet -> red)
3. Improve Agent (tweak prompt / add tools / change architecture)
4. Run eval (Agent can do it now -> green)
5. Add to CI/CD (every change auto-runs the eval regression)
```

**In CI/CD:**
- Every PR auto-runs the core eval suite
- Set thresholds: task completion rate < 90% -> block merge
- Cost budget checks: per-call cost > $X -> alert
- Nightly: run full eval (more tasks, more trials)

---

## Part 4: Tool Selection

Evaluation and observability tools are mostly integrated. Select them together.

### Comprehensive Platforms

| Tool | Type | Eval | Tracing | Strongest At | Best For |
|------|------|------|---------|-------------|----------|
| **[Langfuse](https://langfuse.com)** | Open source | Yes | Yes | Self-hostable, MIT, Claude integration | Individuals / full control |
| **[LangSmith](https://langchain.com/langsmith)** | SaaS | Yes | Yes | Zero overhead, million-scale traces | LangGraph users |
| **[Braintrust](https://braintrust.dev)** | SaaS | Yes | Yes | Full lifecycle + team collaboration | Teams |
| **[Arize Phoenix](https://arize.com)** | Open source + SaaS | Yes | Yes | Deepest Agent decision-chain analysis | Deep analysis |

### Eval-Focused

| Tool | Strongest At |
|------|-------------|
| **[DeepEval](https://deepeval.com)** | Local eval, rich built-in metrics, CI/CD friendly |
| **[Promptfoo](https://promptfoo.dev)** | Red teaming + security testing |
| **[Galileo](https://galileo.ai)** | Real-time guardrails + automated evaluation |

### Observability-Focused

| Tool | Strongest At |
|------|-------------|
| **[Helicone](https://helicone.ai)** | One-line cost tracking |
| **[AgentOps](https://agentops.ai)** | Agent execution recording and replay |
| **[OpenLLMetry](https://github.com/traceloop/openllmetry)** | Open-source OTel GenAI library |

### Selection Recommendations

```
Just starting out      -> Langfuse (free, eval + tracing in one)
Only need cost tracking -> Helicone (one line of code)
LangGraph users        -> LangSmith (native integration)
Teams / enterprise     -> Braintrust or LangSmith
CI/CD eval             -> DeepEval + GitHub Actions
Security testing       -> Promptfoo
```

---

## Part 5: Langfuse Deep Dive (Recommended Starting Point)

Langfuse is not part of the LangChain ecosystem — it is an independent open-source project (YC W23), framework-agnostic. It is a strong starting choice because it is free, has Claude integration, a TypeScript SDK, and combines eval + observability.

### Langfuse + Claude: Three Integration Layers

**Layer 1: Anthropic SDK Direct Tracing**

```typescript
// npm install @anthropic-ai/sdk @arizeai/openinference-instrumentation-anthropic @langfuse/otel
import Anthropic from "@anthropic-ai/sdk";
// After configuring Langfuse OTel, all Claude calls are automatically traced

const anthropic = new Anthropic();
const message = await anthropic.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 1000,
  messages: [{ role: "user", content: "Hello, Claude!" }],
});
// -> Automatically records trace, token usage, and latency
```

**Layer 2: Claude Agent SDK Integration** — Trace the complete Agent execution chain

**Layer 3: Claude Code Tracing** — Trace Claude Code's own calls

### Langfuse Resources

| Resource | Description |
|----------|-------------|
| [Langfuse + Anthropic JS/TS](https://langfuse.com/integrations/model-providers/anthropic-js) | Start here |
| [Langfuse + Claude Agent SDK](https://langfuse.com/integrations/frameworks/claude-agent-sdk-js) | Agent SDK tracing |
| [Langfuse + Claude Code](https://langfuse.com/integrations/other/claude-code) | Claude Code tracing |
| [Langfuse Online Demo](https://langfuse.com/docs/demo) | Free trial |
| [langfuse-examples](https://github.com/langfuse/langfuse-examples) | GitHub examples |
| [JS/TS SDK Examples](https://langfuse.com/docs/sdk/typescript/example-notebook) | TypeScript end-to-end |
| [LLM-as-a-Judge Integration](https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge) | Automated evaluation inside Langfuse |
| [Langfuse GitHub](https://github.com/langfuse/langfuse) | Source code (55K+ stars) |

---

## Essential Reading

### Evaluation Methodology

| Resource | Description |
|----------|-------------|
| [Anthropic: Demystifying Evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) | **Most important** — evaluation methodology primer |
| [AWS: Evaluating AI Agents at Amazon](https://aws.amazon.com/blogs/machine-learning/evaluating-ai-agents-real-world-lessons-from-building-agentic-systems-at-amazon/) | Amazon production experience |
| [Confident AI: Definitive Agent Evaluation Guide](https://www.confident-ai.com/blog/definitive-ai-agent-evaluation-guide) | Comprehensive technical guide |
| [Microsoft: AI Agent Performance Measurement](https://www.microsoft.com/en-us/dynamics-365/blog/it-professional/2026/02/04/ai-agent-performance-measurement/) | Microsoft metrics framework |
| [Evidently AI: LLM-as-a-Judge Guide](https://www.evidentlyai.com/llm-guide/llm-as-a-judge) | Complete LLM Judge guide |
| [Monte Carlo: LLM-as-Judge 7 Best Practices](https://www.montecarlodata.com/blog-llm-as-judge/) | Judge best practices |

### Observability & Tracing

| Resource | Description |
|----------|-------------|
| [OpenTelemetry: AI Agent Observability](https://opentelemetry.io/blog/2025/ai-agent-observability/) | OTel official standard |
| [OTel GenAI Semantic Conventions (2026.03)](https://earezki.com/ai-news/2026-03-21-opentelemetry-just-standardized-llm-tracing-heres-what-it-actually-looks-like-in-code/) | Latest semantic conventions |
| [Trace AI Agent Execution with OTel](https://oneuptime.com/blog/post/2026-02-06-trace-ai-agent-execution-flows-opentelemetry/view) | Hands-on tutorial |
| [Monitor AI Agents in Production](https://oneuptime.com/blog/post/2026-03-14-how-to-monitor-ai-agents-in-production/view) | Production monitoring |
| [AI Engineer's Guide to LLM Observability](https://agenta.ai/blog/the-ai-engineer-s-guide-to-llm-observability-with-opentelemetry) | OTel + LLM complete guide |

### CI/CD & Eval-Driven Development

| Resource | Description |
|----------|-------------|
| [Red Hat: Eval-Driven Development](https://developers.redhat.com/articles/2026/03/23/eval-driven-development-build-evaluate-ai-agents) | Eval-Driven methodology |
| [LangSmith CI/CD Pipeline](https://docs.langchain.com/langsmith/cicd-pipeline-example) | CI/CD reference implementation |
| [CI/CD for Evals: GitHub Actions](https://www.kinde.com/learn/ai-for-software-engineering/ai-devops/ci-cd-for-evals-running-prompt-and-agent-regression-tests-in-github-actions/) | GitHub Actions tutorial |

### Cost Optimization

| Resource | Description |
|----------|-------------|
| [AI Agent Token Cost Optimization](https://fast.io/resources/ai-agent-token-cost-optimization/) | Complete cost optimization guide |
| [Reduce Agent Spend by 60-80%](https://moltbook-ai.com/posts/ai-agent-cost-optimization-2026) | Optimization strategies |
| [Redis: LLM Token Optimization](https://redis.io/blog/llm-token-optimization-speed-up-apps/) | Caching in practice |

### Benchmarks

| Benchmark | What It Tests | Latest Results |
|-----------|--------------|----------------|
| GAIA | Multi-step reasoning + retrieval + execution | Level 3 max 61% |
| ARC-AGI | Generalization from few examples | AI far below human performance |
| SWE-bench | Code repair | 50%+ |
| FieldWorkArena | Business scenario safety | New at AAAI 2026 |

---

## Practice Plan

```
Week 1: Read Anthropic's eval guide + AWS blog
        Sign up for Langfuse; add tracing to an Agent project
Week 2: Define 5 eval tasks (extract from real failures)
        Run 10 tasks; analyze token and latency distribution in Langfuse
Week 3: Implement LLM-as-Judge auto-scoring
        Identify the biggest performance bottleneck; try prompt caching
Week 4: Add eval + cost checks to GitHub Actions
        Compare different prompt strategies on score + cost
```
