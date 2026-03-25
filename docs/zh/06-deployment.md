# 06 — 部署

## 概览

Agent 工程师不需要变成 DevOps 专家，但必须具备把 Agent 推上生产环境的能力。Agent 部署比传统 Web 应用复杂得多——长连接、不可预测的成本、沙箱隔离、状态持久化，每一个都是额外的挑战。

## 部署层级

### 第一层：容器化（必修）

最基本的要求。所有 Agent 项目都应该能构建为 Docker 镜像。

```dockerfile
FROM node:22-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production
COPY . .
CMD ["node", "index.js"]
```

无论最终部署到哪里，Docker 都是起点。

### 第二层：部署平台（选一个，用到熟）

| 平台 | 复杂度 | 适合场景 | 费用模式 | 亮点 |
|------|--------|----------|----------|------|
| **Railway / Render** | 低 | 个人项目 | 免费额度 + 按量计费 | `git push` 即部署 |
| **Cloud Run** | 低-中 | 无状态 Agent | 按调用计费 | 自动伸缩，冷启动快 |
| **AWS Lambda** | 中 | 事件驱动型 Agent | 按调用计费 | 有 15 分钟超时限制 |
| **Fly.io** | 中 | 需要全球分发 | 按用量计费 | 边缘部署 |
| **Kubernetes** | 高 | 企业级多 Agent 系统 | 成本较高 | 完全可控 |

**建议：** 第一个项目用 Railway 或 Cloud Run 就好。Kubernetes 等团队确实需要的时候再学。

### 第三层：Agent 特有的部署难点

传统 Web 部署不会遇到的事情：

| 难点 | 为什么特殊 | 应对方式 |
|------|-----------|----------|
| **长连接/长任务** | 单次 Agent 调用可能跑好几分钟 | 异步任务队列（BullMQ、Cloud Tasks） |
| **沙箱隔离** | Agent 生成的代码需要隔离执行 | Docker-in-Docker 或托管沙箱（E2B） |
| **成本控制** | Token 消耗很难预估 | 单请求限额 + 每日预算上限 |
| **状态持久化** | 长任务需要检查点 | 用 Redis / 数据库保存中间状态 |
| **弹性伸缩** | 流量波动大 | Serverless 或 Kubernetes HPA |
| **密钥管理** | Agent 要调各种外部 API | 环境变量 / Secret Manager |

### 第四层：托管 Agent 平台（了解即可）

不想自己管基础设施的人可以考虑：

| 平台 | 说明 |
|------|------|
| [Koyeb Sandboxes](https://www.koyeb.com/blog/serverless-ai-infrastructure-going-into-2026) | Serverless Agent 沙箱，按需启停 |
| [Modal](https://modal.com) | GPU 按用量计费，Python 原生，自动伸缩 |
| [E2B](https://e2b.dev) | Agent 代码执行沙箱 API |
| [DigitalOcean + OpenClaw](https://www.digitalocean.com/blog/openclaw-digitalocean-app-platform) | 一键部署 OpenClaw Agent |
| [Google GKE + ADK](https://docs.google.com/kubernetes-engine/docs/tutorials/agentic-adk-vertex) | Kubernetes 原生 Agent 部署 |

## vLLM 与自托管推理：在学习路径中的定位

### vLLM 是什么

vLLM 是一个高性能 LLM 推理引擎，能让你在自有 GPU 上运行开源模型（Qwen、Llama、Mistral 等），完全不依赖 Anthropic/OpenAI 的 API。

核心技术 PagedAttention 可以减少 60-80% 的显存浪费。吞吐量是 Ollama 的 19 倍（793 TPS vs 41 TPS）。Meta、Mistral、Cohere、IBM 等公司都在用。

### 现在需要学 vLLM 吗？

**简短的回答：还不需要。** 原因如下：

| 因素 | 当前状况 |
|------|----------|
| **用 Claude/GPT** | API 调用就够了，不需要自托管 |
| **成本临界点** | 日均 200 万+ token 以上，自托管才划算 |
| **运维开销** | 每月 10-20 小时维护 + GPU 费用 |
| **模型质量** | 开源模型在 Agent 场景（工具调用、多步推理）的表现仍落后于 Claude/GPT |

### 什么时候会用到

| 场景 | 原因 |
|------|------|
| **数据隐私** | 公司政策要求数据不出内网 |
| **极大流量** | 日均 1 亿+ token，API 成本扛不住 |
| **定制模型** | 需要微调或使用特定的开源模型 |
| **离线环境** | 没有互联网 |
| **混合路由** | 简单任务走本地小模型，复杂任务走 API 大模型 |

### 2026 趋势：混合路由

最实用的方案既不是"全部自托管"，也不是"全部走 API"，而是**智能路由**：

```
User request -> Router
  |
  |-- Simple/repetitive tasks -> Local small model (vLLM + Qwen, etc.) -> Low cost
  |
  +-- Complex reasoning tasks -> Claude/GPT API -> High quality
```

LiteLLM（[文档](https://docs.litellm.ai)）和 OpenRouter 这类工具可以让你一行代码切换模型供应商。

### 想深入了解的资源

| 资源 | 说明 |
|------|------|
| [vLLM Official Docs](https://docs.vllm.ai) | 入门起点 |
| [vLLM Production Deployment](https://introl.com/blog/vllm-production-deployment-inference-serving-architecture) | 生产环境部署架构 |
| [Coding Agent with Self-hosted LLM (vLLM + OpenCode)](https://cefboud.com/posts/coding-agent-self-hosted-llm-opencode-vllm/) | 端到端实战演练 |
| [How to Integrate vLLM in OpenClaw](http://www.emzeth.com/2026/03/how-to-integrate-vllm-models-in-openclaw.html) | OpenClaw + vLLM 集成 |
| [Self-Host LLM vs API: Cost Breakdown 2026](https://devtk.ai/en/blog/self-hosting-llm-vs-api-cost-2026/) | 成本对比分析 |
| [Self-Hosted LLM Guide 2026](https://blog.premai.io/self-hosted-llm-guide-setup-tools-cost-comparison-2026/) | 综合指南 |
| [vLLM Alternatives 2026](https://kanerika.com/blogs/vllm-alternatives/) | 替代方案对比 |

### 推理框架速览

| 框架 | 定位 | 适合场景 |
|------|------|----------|
| **vLLM** | 高吞吐生产推理 | 生产环境、高并发 |
| **Ollama** | 本地开发/实验 | 快速试跑本地模型 |
| **TGI (HuggingFace)** | 全功能推理服务 | HuggingFace 生态用户 |
| **LiteLLM** | 统一 API 代理 | 多模型路由（本身不是推理引擎） |
| **SGLang** | 高性能 + 结构化输出 | 需要 JSON mode 的场景 |

## 延伸阅读

| 资源 | 说明 |
|------|------|
| [Deploying AI Agents to Production](https://machinelearningmastery.com/deploying-ai-agents-to-production-architecture-infrastructure-and-implementation-roadmap/) | 全面的部署路线图 |
| [Deploy Agentic Workflows with K8s + Terraform](https://thenewstack.io/deploy-agentic-ai-workflows-with-kubernetes-and-terraform/) | Kubernetes 部署实战 |
| [Koyeb: Serverless AI Infrastructure 2026](https://www.koyeb.com/blog/serverless-ai-infrastructure-going-into-2026) | Serverless 趋势 |
| [Deploy AI Apps with Minimal Infrastructure](https://www.runpod.io/articles/guides/deploy-ai-apps-minimal-infrastructure-docker) | Docker 最小化部署 |
| [System Design in 2026: AI-Native and Serverless](https://dev.to/devin-rosario/the-complete-guide-to-system-design-in-2026-ai-native-and-serverless-1kpb) | 系统设计全景 |

## 练习计划

```
Week 1: Write a Dockerfile for an Agent project; verify it runs locally with docker run
Week 1: Deploy to Railway or Cloud Run; confirm it's accessible via URL
Week 2: Add an async task queue so long tasks don't block the HTTP response
Week 2: Add secret management, no hardcoded API keys
Week 3: (Optional) Run a small model locally with Ollama; compare with API calls on quality and cost
Week 3: (Optional) Set up LiteLLM for basic model routing, simple tasks go local, complex go API
```
