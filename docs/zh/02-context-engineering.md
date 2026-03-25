# 02 — Context Engineering

## 概览

Context Engineering 是一门系统化管理 Agent 上下文的学科：决定什么信息进入上下文窗口、以什么形态进入、在什么时机进入。不到一年时间，它从一个小众概念变成了 2026 年 AI 工程的核心方向。

**Agent 的表现，90% 取决于上下文质量。**

## 为什么这是 2026 年最热的方向

- Manus 团队分享过，他们的 Agent 框架重建了 4 次，每次的核心挑战都是上下文管理。
- 一个复杂的工具 schema 就能吃掉 500+ token
- 接入多个 MCP Server 的话，还没开始推理就已经消耗 50,000+ token 了
- MCP 月下载量已经超过 9700 万次，接的工具越多，上下文爆炸的问题就越严重

## 核心领域

### 1. 上下文预算管理

你有一个固定的 token 窗口（比如 200K），怎么分配？

```
Total budget 200K tokens
|-- System instructions    ~2K    (1%)
|-- Tool schemas           ~10-50K (5-25%)  <-- most likely to explode
|-- Conversation history   ~20-50K (10-25%)
|-- Retrieved knowledge    ~20-50K (10-25%)
+-- Space for reasoning    remainder (at least 50%)
```

最佳实践：同时暴露的工具控制在 20 个以内。不常用的工具通过搜索机制按需加载。

### 2. 分层记忆架构

和操作系统的存储层次结构一个思路：

| 层级 | 类比 | 容量 | 速度 | 内容 | 实现方式 |
|------|------|------|------|------|---------|
| 工作记忆 | CPU 寄存器 | 极小 | 最快 | 当前步骤所需信息 | 直接放在 prompt 里 |
| 短期记忆 | 内存 | 中等 | 快 | 当前对话上下文 | 滑动窗口 / 摘要 |
| 长期记忆 | 硬盘 | 很大 | 慢 | 用户偏好、历史记录 | 向量数据库 / 知识图谱 |

### 3. 工具 Schema 压缩

- 参数描述写简洁点就行，别写论文。够用就好。
- 用枚举约束取值：减少 Agent 出错的同时缩短描述长度
- 分组加载：只加载当前任务类型相关的工具子集
- 热门 schema 缓存：高频工具的解析结果做缓存

### 4. Agentic Context Engineering (ACE)

自进化的上下文。Agent 根据执行反馈自动更新自己的上下文模板：有用的信息留下，冗余的信息裁掉。可以理解为一本不断自我优化的操作手册。

### 5. Agentic Graph RAG

Agent 推理 + 知识图谱。不只是向量检索，而是在结构化知识图谱上做推理和遍历。

## 记忆框架对比（2026）

| 框架 | 优势 | 适用场景 |
|------|------|---------|
| **Mem0** | 最成熟：准确率提升 26%，延迟降低 91%，token 消耗降低 90% | 生产级长期记忆 |
| **Letta** | 模拟操作系统的存储层级（主上下文 = 内存，外部存储 = 硬盘） | 需要细粒度控制的场景 |
| **LangChain Memory** | 内置多种记忆模式 | LangChain 生态用户 |
| **Redis** | 读写极快，适合短期记忆 | 高频交互场景 |
| **Zep** | 专注对话记忆 | 聊天型 Agent |

## 必读材料

| 资源 | 说明 |
|------|------|
| [Manus: Context Engineering for AI Agents](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) | **从这里开始。** 重建 4 次框架后的实战经验 |
| [Martin Fowler: Context Engineering for Coding Agents](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html) | 软件工程视角的解读 |
| [State of Context Engineering 2026](https://www.newsletter.swirlai.com/p/state-of-context-engineering-in-2026) | 行业全景 |
| [arxiv: Context Engineering for AI Agents in OSS](https://arxiv.org/abs/2510.21413) | 学术论文 |
| [arxiv: Agentic Context Engineering (ACE)](https://arxiv.org/abs/2510.04618) | ACE 方法论论文 |
| [Mem0: Production-Ready Long-Term Memory](https://arxiv.org/pdf/2504.19413) | Mem0 技术论文 |
| [IBM: What Is AI Agent Memory](https://www.ibm.com/think/topics/ai-agent-memory) | 概念入门 |
| [6 Best AI Agent Memory Frameworks 2026](https://machinelearningmastery.com/the-6-best-ai-agent-memory-frameworks-you-should-try-in-2026/) | 框架横评 |
| [AWS AgentCore: Long-Term Memory Deep Dive](https://aws.amazon.com/blogs/machine-learning/building-smarter-ai-agents-agentcore-long-term-memory-deep-dive/) | AWS 生产实践 |
| [Redis: Build AI Agents with Memory Management](https://redis.io/blog/build-smarter-ai-agents-manage-short-term-and-long-term-memory-with-redis/) | Redis 实操教程 |
| [Agent Memory Paper List (GitHub)](https://github.com/Shichun-Liu/Agent-Memory-Paper-List) | 学术论文合集 |

## Agentic RAG：Agent 与检索的结合

RAG 在 2026 年并没有过时，而是进化了。传统 RAG 是固定流水线（查询 -> 检索 -> 生成），Agentic RAG 则让 Agent 自主决定何时检索、检索什么、是否需要多轮检索。

| | 传统 RAG | Agentic RAG |
|---|---------|-------------|
| 流程 | 固定：查询 -> 检索 -> 生成 | Agent 自主决定 |
| 灵活性 | 静态 | 动态改写查询、切换数据源、判断信息是否充分 |
| 适用于 | 简单问答 | 复杂多步推理 + 知识密集型任务 |

**真正要学的不是 "怎么搭一个 RAG 流水线"，而是 "怎么让 Agent 聪明地使用检索"。**

### 相关资源

| 资源 | 说明 |
|------|------|
| [IBM: What is Agentic RAG](https://www.ibm.com/think/topics/agentic-rag) | 概念入门 |
| [Weaviate: What Is Agentic RAG](https://weaviate.io/blog/what-is-agentic-rag) | 架构详解 |
| [arxiv: Agentic RAG Survey](https://arxiv.org/abs/2501.09136) | 学术综述 |
| [ByteByteGo: MCP vs RAG vs AI Agents](https://blog.bytebytego.com/p/ep202-mcp-vs-rag-vs-ai-agents) | 三者对比 |

## 实践计划

1. **第 1 周**：读 Manus 的博客 + Martin Fowler 的文章，把核心概念内化
2. **第 1 周**：拿一个真实的 Agent 调用，算一下 token 分布——看看 token 主要花在哪里
3. **第 2 周**：用 Mem0 给项目加长期记忆（记住用户偏好）
4. **第 2 周**：尝试压缩工具 schema——精简描述，测量对 Agent 准确率的影响
5. **第 3 周**：设计一个上下文自进化机制，让 Agent 自动裁剪无用上下文
