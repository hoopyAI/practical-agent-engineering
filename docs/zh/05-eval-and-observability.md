# 05 — 评估与可观测

## 概览

评估 (Eval) 和可观测 (Observability) 是一体两面：

- **评估**回答的问题是：Agent 做得好不好？
- **可观测**回答的问题是：Agent 在干什么、花了多少钱、哪一步出了问题？

没有链路追踪就没法做评估——你压根不知道哪一步挂了；没有评估的追踪也只是一堆数据——你没有判断质量的标尺。好在主流工具（Langfuse、LangSmith、Braintrust）本身就把两者合在一起了。

这是 Agent 开发者最容易忽视的环节。谁先把它做好，谁就有明显的竞争优势。

---

## 第一部分：评估方法论

### Anthropic 的评估框架

Anthropic 在 2026 年 1 月发布的 [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) 是目前最好的入门材料。

**核心术语：**

| 概念 | 含义 |
|---------|---------|
| **Task** | 一条测试用例，包含明确的输入和成功标准 |
| **Trial** | 对某条 Task 的一次尝试（模型输出具有随机性，因此需要多次 Trial） |
| **Grader** | 给 Agent 输出打分的逻辑，可以是规则，也可以是 LLM |
| **Transcript** | 完整的执行记录（输出、工具调用、推理过程、中间结果） |

### 两个评估维度

**轨迹级（过程评估）：** Agent *怎样*得到结果的？

| 指标 | 衡量的内容 | 举例 |
|--------|-----------------|---------|
| 工具选择准确度 | 该搜索时是否搜索了？ | 本该搜索却直接编造答案 = 0 分 |
| 步骤效率 | 完成任务用了几步？ | 3 步能搞定的事，用了 10 步 |
| 推理链质量 | 逻辑是否连贯？ | 中间出现幻觉或前后矛盾 |
| 错误恢复 | 失败后能否自我纠正？ | 某个工具调用失败后更换了策略 |
| Token 效率 | 花了多少 token？ | 同样的结果：方案 A 用 5K，方案 B 用 50K |

**结果级（结果评估）：** 任务*完成*了吗？

| 指标 | 衡量的内容 |
|--------|-----------------|
| 任务完成率 | 是否成功？ |
| 结果正确性 | 答案对不对？ |
| 延迟 | 快不快？ |
| 成本 | 花了多少钱？ |

- 开发 / 调试阶段 -> 关注轨迹（搞清楚*为什么*失败）
- 生产监控阶段 -> 关注结果（衡量*业务价值*）
- 两者缺一不可

### Anthropic 的实操建议

1. **先从 20-50 个简单任务开始。** 早期改动的边际效益最高，小样本就够了。
2. **从真实故障中提取任务，**而非凭空捏造场景。用那些真正出过问题的案例。
3. **评估结果，而非路径。** Agent 可能找到比你预期更好的解法。
4. **实践评估驱动开发。** 先用评估定义目标能力，再反复迭代直到 Agent 达标。

### LLM-as-Judge（自动化评估的核心方法）

用另一个 LLM 来评判 Agent 的输出，是 2026 年最主流的自动化评估方式。

```
Input: task description + Agent output + scoring rubric + optional reference answer
  | Send to Judge LLM
Output: score per dimension (1-5) + justification (Chain-of-Thought)
```

**关键实践：**

| 实践 | 原因 |
|----------|-----|
| 用整数评分 (1-5) | LLM 对离散分值的一致性更高 |
| 为每个分值写明确定义 | "3 = 完成但有小错误"比"3 = 一般"清晰得多 |
| 让 Judge 先写推理再打分 (CoT) | 可靠性提升 10-15% |
| 用强模型做 Judge | GPT-5 级别的模型与人工评判的一致率达 80-90% |
| 先让人类标注几十个样本 | 如果人类自己都意见不一，说明评分标准需要修改 |

**评分标准示例：**

```
Evaluation dimension: Task Completion

5 = Fully correct, nothing missing
4 = Mostly complete, minor imperfections that don't affect usability
3 = Main parts done, but obvious omissions or errors
2 = Only a small portion completed; core requirements unmet
1 = Completely failed or entirely wrong
```

---

## 第二部分：可观测基础

### 全链路追踪

把 Agent 执行的每一步都记录成一棵 trace 树：

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

### 成本监控

Agent 的开销很难预测。一个陷入重试循环的"失控" Agent 能轻松烧掉几十美元。

**三层预算控制：**

| 层级 | 限制 | 触发动作 |
|-------|-------|---------|
| 单次请求 | `max_tokens` | 截断响应 |
| 单个任务 | Token 预算（如 50K） | 暂停 Agent，通知人类 |
| 日/月 | 总花费上限 | 50% 预警、80% 警告、100% 暂停 |

**成本优化策略：**

| 策略 | 效果 | 做法 |
|----------|--------|-----|
| **Prompt caching** | 输入成本 -90%，延迟 -75% | 让供应商缓存固定的系统指令 |
| **模型路由** | 总成本 -40-60% | 简单任务用小模型，复杂任务用大模型 |
| **结果缓存** | 重复查询零成本 | 相同输入直接返回缓存结果 |
| **工具 schema 压缩** | 输入 token -30-50% | 精简描述，按需加载工具 |

**给每一笔 token 消耗打上元数据标签：** Agent ID、任务类型、对话线程。

### 质量与异常监控

| 维度 | 做法 |
|-----------|-----|
| **输出质量** | 定期抽样 + LLM-as-Judge 自动打分 |
| **异常检测** | 输出格式变化、token 消耗骤增 |
| **回归检测** | 每次更新后跑评估基线对比 |
| **用户反馈闭环** | 建立标注队列，让领域专家审核 |
| **静默重试** | 成本飙升的头号元凶——Agent 在悄悄地不断重试 |

### OpenTelemetry：行业标准

2026 年 3 月，OpenTelemetry 发布了 **GenAI Semantic Conventions**，统一了 LLM span 的命名和属性规范。用 OTel 做追踪意味着你可以无缝对接任何兼容的后端。

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

## 第三部分：评估驱动开发 + CI/CD

类似 TDD，但对象是 Agent：

```
1. Write eval (define what the Agent should be able to do)
2. Run eval (Agent can't do it yet -> red)
3. Improve Agent (tweak prompt / add tools / change architecture)
4. Run eval (Agent can do it now -> green)
5. Add to CI/CD (every change auto-runs the eval regression)
```

**在 CI/CD 中：**
- 每个 PR 自动跑核心评估套件
- 设置阈值：任务完成率 < 90% -> 阻止合并
- 成本预算检查：单次调用成本 > $X -> 告警
- 每晚：跑完整评估（更多任务、更多 Trial）

---

## 第四部分：工具选型

评估和可观测工具大多已经融合在一起，应当一并选择。

### 综合平台

| 工具 | 类型 | 评估 | 追踪 | 最强项 | 适合谁 |
|------|------|------|---------|-------------|----------|
| **[Langfuse](https://langfuse.com)** | 开源 | 有 | 有 | 可自部署、MIT 协议、Claude 集成 | 个人 / 需要完全掌控 |
| **[LangSmith](https://langchain.com/langsmith)** | SaaS | 有 | 有 | 零接入成本、百万级 trace | LangGraph 用户 |
| **[Braintrust](https://braintrust.dev)** | SaaS | 有 | 有 | 全生命周期 + 团队协作 | 团队 |
| **[Arize Phoenix](https://arize.com)** | 开源 + SaaS | 有 | 有 | 最深度的 Agent 决策链分析 | 深度分析 |

### 侧重评估

| 工具 | 最强项 |
|------|-------------|
| **[DeepEval](https://deepeval.com)** | 本地评估、内置指标丰富、对 CI/CD 友好 |
| **[Promptfoo](https://promptfoo.dev)** | 红队攻防 + 安全测试 |
| **[Galileo](https://galileo.ai)** | 实时护栏 + 自动评估 |

### 侧重可观测

| 工具 | 最强项 |
|------|-------------|
| **[Helicone](https://helicone.ai)** | 一行代码接入成本追踪 |
| **[AgentOps](https://agentops.ai)** | Agent 执行录制与回放 |
| **[OpenLLMetry](https://github.com/traceloop/openllmetry)** | 开源 OTel GenAI 库 |

### 选型建议

```
Just starting out      -> Langfuse (free, eval + tracing in one)
Only need cost tracking -> Helicone (one line of code)
LangGraph users        -> LangSmith (native integration)
Teams / enterprise     -> Braintrust or LangSmith
CI/CD eval             -> DeepEval + GitHub Actions
Security testing       -> Promptfoo
```

---

## 第五部分：Langfuse 深入介绍（推荐起点）

Langfuse 不属于 LangChain 生态，而是一个独立的开源项目（YC W23），与框架无关。它之所以适合作为起点，是因为免费、支持 Claude 集成、有 TypeScript SDK，并且把评估和可观测合二为一。

### Langfuse + Claude：三层集成

**第一层：Anthropic SDK 直接追踪**

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

**第二层：Claude Agent SDK 集成。** 追踪完整的 Agent 执行链。

**第三层：Claude Code 追踪。** 追踪 Claude Code 自身的调用。

### Langfuse 资源

| 资源 | 说明 |
|----------|-------------|
| [Langfuse + Anthropic JS/TS](https://langfuse.com/integrations/model-providers/anthropic-js) | 从这里开始 |
| [Langfuse + Claude Agent SDK](https://langfuse.com/integrations/frameworks/claude-agent-sdk-js) | Agent SDK 追踪 |
| [Langfuse + Claude Code](https://langfuse.com/integrations/other/claude-code) | Claude Code 追踪 |
| [Langfuse Online Demo](https://langfuse.com/docs/demo) | 免费试用 |
| [langfuse-examples](https://github.com/langfuse/langfuse-examples) | GitHub 示例 |
| [JS/TS SDK Examples](https://langfuse.com/docs/sdk/typescript/example-notebook) | TypeScript 端到端 |
| [LLM-as-a-Judge Integration](https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge) | Langfuse 内置自动评估 |
| [Langfuse GitHub](https://github.com/langfuse/langfuse) | 源码（55K+ stars） |

---

## 延伸阅读

### 评估方法论

| 资源 | 说明 |
|----------|-------------|
| [Anthropic: Demystifying Evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) | **从这里开始。** 评估方法论入门。 |
| [AWS: Evaluating AI Agents at Amazon](https://aws.amazon.com/blogs/machine-learning/evaluating-ai-agents-real-world-lessons-from-building-agentic-systems-at-amazon/) | Amazon 生产环境实战经验 |
| [Confident AI: Definitive Agent Evaluation Guide](https://www.confident-ai.com/blog/definitive-ai-agent-evaluation-guide) | 全面的技术指南 |
| [Microsoft: AI Agent Performance Measurement](https://www.microsoft.com/en-us/dynamics-365/blog/it-professional/2026/02/04/ai-agent-performance-measurement/) | Microsoft 指标体系 |
| [Evidently AI: LLM-as-a-Judge Guide](https://www.evidentlyai.com/llm-guide/llm-as-a-judge) | 完整的 LLM Judge 指南 |
| [Monte Carlo: LLM-as-Judge 7 Best Practices](https://www.montecarlodata.com/blog-llm-as-judge/) | Judge 最佳实践 |

### 可观测与追踪

| 资源 | 说明 |
|----------|-------------|
| [OpenTelemetry: AI Agent Observability](https://opentelemetry.io/blog/2025/ai-agent-observability/) | OTel 官方标准 |
| [OTel GenAI Semantic Conventions (2026.03)](https://earezki.com/ai-news/2026-03-21-opentelemetry-just-standardized-llm-tracing-heres-what-it-actually-looks-like-in-code/) | 最新语义规范 |
| [Trace AI Agent Execution with OTel](https://oneuptime.com/blog/post/2026-02-06-trace-ai-agent-execution-flows-opentelemetry/view) | 实操教程 |
| [Monitor AI Agents in Production](https://oneuptime.com/blog/post/2026-03-14-how-to-monitor-ai-agents-in-production/view) | 生产环境监控 |
| [AI Engineer's Guide to LLM Observability](https://agenta.ai/blog/the-ai-engineer-s-guide-to-llm-observability-with-opentelemetry) | OTel + LLM 完整指南 |

### CI/CD 与评估驱动开发

| 资源 | 说明 |
|----------|-------------|
| [Red Hat: Eval-Driven Development](https://developers.redhat.com/articles/2026/03/23/eval-driven-development-build-evaluate-ai-agents) | 评估驱动开发方法论 |
| [LangSmith CI/CD Pipeline](https://docs.langchain.com/langsmith/cicd-pipeline-example) | CI/CD 参考实现 |
| [CI/CD for Evals: GitHub Actions](https://www.kinde.com/learn/ai-for-software-engineering/ai-devops/ci-cd-for-evals-running-prompt-and-agent-regression-tests-in-github-actions/) | GitHub Actions 教程 |

### 成本优化

| 资源 | 说明 |
|----------|-------------|
| [AI Agent Token Cost Optimization](https://fast.io/resources/ai-agent-token-cost-optimization/) | 完整成本优化指南 |
| [Reduce Agent Spend by 60-80%](https://moltbook-ai.com/posts/ai-agent-cost-optimization-2026) | 优化策略 |
| [Redis: LLM Token Optimization](https://redis.io/blog/llm-token-optimization-speed-up-apps/) | 缓存实战 |

### 基准测试

| Benchmark | 测试内容 | 最新结果 |
|-----------|--------------|----------------|
| GAIA | 多步推理 + 检索 + 执行 | Level 3 最高 61% |
| ARC-AGI | 少样本泛化能力 | AI 远低于人类水平 |
| SWE-bench | 代码修复 | 50%+ |
| FieldWorkArena | 业务场景安全性 | AAAI 2026 新发布 |

---

## 练习计划

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
