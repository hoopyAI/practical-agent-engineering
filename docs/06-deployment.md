# 06 — Deployment

## Overview

An Agent engineer doesn't need to be a DevOps expert, but does need to be able to deploy Agents to production. Agent deployment is more involved than traditional web apps because of long connections, unpredictable costs, sandboxing requirements, and state persistence.

## Deployment Tiers

### Tier 1: Containerization (Must Learn)

The most fundamental requirement. Every Agent project should be buildable as a Docker image.

```dockerfile
FROM node:22-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production
COPY . .
CMD ["node", "index.js"]
```

No matter where you deploy, Docker is the starting point.

### Tier 2: Deployment Platforms (Pick One and Get Comfortable)

| Platform | Complexity | Best For | Cost | Highlights |
|----------|-----------|----------|------|-----------|
| **Railway / Render** | Low | Personal projects | Free tier + usage-based | `git push` to deploy |
| **Cloud Run** | Low-Medium | Stateless Agents | Per-invocation billing | Auto-scaling, fast cold starts |
| **AWS Lambda** | Medium | Event-driven Agents | Per-invocation billing | 15-minute timeout limit |
| **Fly.io** | Medium | Global distribution needed | Usage-based | Edge deployment |
| **Kubernetes** | High | Enterprise multi-Agent systems | High | Full control |

**Recommendation**: Start with Railway or Cloud Run for a first project. Learn K8s when a team actually needs it.

### Tier 3: Agent-Specific Deployment Concerns

Things traditional web deployment doesn't deal with:

| Concern | Why It's Special | How to Handle |
|---------|-----------------|--------------|
| **Long connections/tasks** | A single Agent call can run for minutes | Async task queue (BullMQ, Cloud Tasks) |
| **Sandbox isolation** | Agent-generated code needs isolation | Docker-in-Docker or managed sandbox (E2B) |
| **Cost control** | Token consumption is unpredictable | Per-request limits + daily budget |
| **State persistence** | Long tasks need checkpointing | Redis / database for intermediate state |
| **Elastic scaling** | Load is unpredictable | Serverless or K8s HPA |
| **Secret management** | Agents call various APIs | Environment variables / Secret Manager |

### Tier 4: Managed Agent Platforms (Awareness Level)

Options for those who don't want to manage infrastructure:

| Platform | Description |
|----------|-------------|
| [Koyeb Sandboxes](https://www.koyeb.com/blog/serverless-ai-infrastructure-going-into-2026) | Serverless Agent sandboxes, on-demand start/stop |
| [Modal](https://modal.com) | GPU pay-per-use, Python-native, auto-scaling |
| [E2B](https://e2b.dev) | Agent code execution sandbox API |
| [DigitalOcean + OpenClaw](https://www.digitalocean.com/blog/openclaw-digitalocean-app-platform) | One-click OpenClaw Agent deployment |
| [Google GKE + ADK](https://docs.google.com/kubernetes-engine/docs/tutorials/agentic-adk-vertex) | K8s-native Agent deployment |

## vLLM and Self-Hosted Inference: Where It Fits in the Learning Path

### What Is vLLM

vLLM is a high-performance LLM inference engine. It lets you run open-source models (Qwen, Llama, Mistral, etc.) on your own GPUs, independent of Anthropic/OpenAI APIs.

Core technology PagedAttention reduces 60-80% of VRAM waste. Throughput is 19x higher than Ollama (793 TPS vs 41 TPS). Used by Meta, Mistral, Cohere, IBM.

### Do You Need to Learn vLLM Now?

**Short answer: not yet.** Here's why:

| Factor | Current State |
|--------|--------------|
| **Using Claude/GPT** | API calls are sufficient, no self-hosting needed |
| **Cost breakeven** | 2M+ tokens per day before self-hosting is worth it |
| **Ops overhead** | 10-20 hours/month maintenance + GPU costs |
| **Model quality** | Open-source models in Agent scenarios (tool calling, multi-step reasoning) still lag behind Claude/GPT |

### When You'll Need It

| Scenario | Why |
|----------|-----|
| **Data privacy** | Company policy requires data stays on-premise |
| **Extreme volume** | 100M+ tokens per day; API costs become prohibitive |
| **Custom models** | Need fine-tuning or specific open-source models |
| **Offline environments** | No internet access |
| **Hybrid routing** | Simple tasks use local small models; complex tasks use API large models |

### 2026 Trend: Hybrid Routing

The most practical approach is neither "all self-hosted" nor "all API." It is **intelligent routing**:

```
User request -> Router
  |
  |-- Simple/repetitive tasks -> Local small model (vLLM + Qwen, etc.) -> Low cost
  |
  +-- Complex reasoning tasks -> Claude/GPT API -> High quality
```

Tools like [LiteLLM](https://docs.litellm.ai) and OpenRouter let you switch model providers with one line of code.

### Resources for the Curious

| Resource | Description |
|----------|-------------|
| [vLLM Official Docs](https://docs.vllm.ai) | Starting point |
| [vLLM Production Deployment](https://introl.com/blog/vllm-production-deployment-inference-serving-architecture) | Production deployment architecture |
| [Coding Agent with Self-hosted LLM (vLLM + OpenCode)](https://cefboud.com/posts/coding-agent-self-hosted-llm-opencode-vllm/) | End-to-end walkthrough |
| [How to Integrate vLLM in OpenClaw](http://www.emzeth.com/2026/03/how-to-integrate-vllm-models-in-openclaw.html) | OpenClaw + vLLM |
| [Self-Host LLM vs API: Cost Breakdown 2026](https://devtk.ai/en/blog/self-hosting-llm-vs-api-cost-2026/) | Cost comparison |
| [Self-Hosted LLM Guide 2026](https://blog.premai.io/self-hosted-llm-guide-setup-tools-cost-comparison-2026/) | Comprehensive guide |
| [vLLM Alternatives 2026](https://kanerika.com/blogs/vllm-alternatives/) | Alternative comparison |

### Inference Framework Quick Comparison

| Framework | Positioning | Best For |
|-----------|------------|----------|
| **vLLM** | High-throughput production inference | Production, high concurrency |
| **Ollama** | Local development/experimentation | Quick local model testing |
| **TGI (HuggingFace)** | Full-featured inference serving | HuggingFace ecosystem |
| **LiteLLM** | Unified API proxy | Multi-model routing (not an inference engine) |
| **SGLang** | High-performance + structured output | Scenarios requiring JSON mode |

## Essential Reading

| Resource | Description |
|----------|-------------|
| [Deploying AI Agents to Production](https://machinelearningmastery.com/deploying-ai-agents-to-production-architecture-infrastructure-and-implementation-roadmap/) | Comprehensive deployment roadmap |
| [Deploy Agentic Workflows with K8s + Terraform](https://thenewstack.io/deploy-agentic-ai-workflows-with-kubernetes-and-terraform/) | K8s deployment walkthrough |
| [Koyeb: Serverless AI Infrastructure 2026](https://www.koyeb.com/blog/serverless-ai-infrastructure-going-into-2026) | Serverless trends |
| [Deploy AI Apps with Minimal Infrastructure](https://www.runpod.io/articles/guides/deploy-ai-apps-minimal-infrastructure-docker) | Docker minimal deployment |
| [System Design in 2026: AI-Native and Serverless](https://dev.to/devin-rosario/the-complete-guide-to-system-design-in-2026-ai-native-and-serverless-1kpb) | System design overview |

## Practice Plan

```
Week 1: Write a Dockerfile for an Agent project; verify it runs locally with docker run
Week 1: Deploy to Railway or Cloud Run; confirm it's accessible via URL
Week 2: Add an async task queue so long tasks don't block the HTTP response
Week 2: Add secret management, no hardcoded API keys
Week 3: (Optional) Run a small model locally with Ollama; compare with API calls on quality and cost
Week 3: (Optional) Set up LiteLLM for basic model routing, simple tasks go local, complex go API
```
