# 04 — 安全与沙箱

## 概述

Agent 能运行代码、调用 API、写入文件——同样也能造成破坏。一条精心构造的 prompt injection，就足以让 Agent 把整个数据库删掉。2026 年，OWASP 专门发布了 Agent Security Top 10，可见业界对这一问题的重视程度。

## 三层安全架构

```
+----------------------------------+
|  Layer 1: Code Execution Sandbox |  <-- Agent 生成的代码在此运行
+----------------------------------+
|  Layer 2: Permissions & ACL      |  <-- 定义 Agent 能做什么、不能做什么
+----------------------------------+
|  Layer 3: Human-Agent Gating     |  <-- 哪些操作需要人类审批
+----------------------------------+
```

## 第一层：代码执行沙箱

Agent 生成的代码**绝不能**直接在宿主机上跑。2026 年的行业共识是：仅靠标准 Docker 隔离已经不够了。

### 三种隔离级别

| 技术 | 隔离强度 | 启动时间 | 性能开销 | 谁在用 |
|------------|----------------|-------------|----------|-------------|
| **Docker/runc** | 弱（共享内核） | 秒级 | 最低 | 开发环境 |
| **gVisor** | 中（用户态内核） | 秒级 | I/O +10-30% | Google GKE Agent Sandbox |
| **Firecracker MicroVM** | 强（独立内核） | ~125ms | 内存 <5MB | AWS Lambda |

### 选型建议

- **个人项目/学习用途**：Docker + 断网 + 只读文件系统
- **生产环境**：gVisor
- **最高安全要求**：Firecracker

### 动手：最小沙箱

```bash
# Docker + no network + read-only + limited resources
docker run --rm \
  --network none \
  --read-only \
  --memory 512m \
  --cpus 1 \
  -v /tmp/agent-workspace:/workspace \
  python:3.12-slim \
  python -c "print('hello from sandbox')"
```

### 托管方案（不想自己搭）

| 方案 | 类型 | 亮点 |
|----------|------|-----------|
| [E2B](https://e2b.dev) | 托管 API | 一行代码创建沙箱 |
| [microsandbox](https://github.com/nicholasgasior/microsandbox) | 自托管 | 开源，轻量 |
| [Google GKE Agent Sandbox](https://docs.google.com/kubernetes-engine/docs/how-to/agent-sandbox) | K8s 原生 | gVisor 隔离 |
| [Modal](https://modal.com) | 托管 | 按量计费，自动扩容 |

## 第二层：权限与访问控制

### 权限模型设计

```
Agent Permission Model:

Within workspace (/workspace/)
|-- /output/     -> free read/write
|-- /logs/       -> free write
|-- /temp/       -> free read/write
+-- everything else -> read-only

Outside workspace
|-- Filesystem     -> forbidden (requires approval)
|-- Network calls  -> whitelist only
|-- Database writes -> requires approval
+-- External APIs  -> authorized per-tool

High-risk operations
|-- Delete data        -> mandatory human confirmation
|-- Send messages      -> mandatory human confirmation
|-- Payments/transfers -> mandatory human confirmation
+-- Publish/deploy     -> mandatory human confirmation
```

### 工具级权限

每个 MCP tool 单独授权，粒度可以细到表和行数：

```json
{
  "tool": "database_query",
  "permissions": {
    "read": true,
    "write": false,
    "delete": false,
    "tables": ["users", "products"],
    "max_rows": 1000
  }
}
```

## 第三层：人机协作门控

### 从 HITL 到 HOTL

| 模式 | 全称 | 人类角色 | 适用场景 |
|------|-----------|-----------|----------|
| **HITL** | Human-in-the-Loop | 每一步都确认 | 高风险、低频操作 |
| **HOTL** | Human-on-the-Loop | 监督为主，异常时介入 | 中风险、中频操作 |
| **全自动** | — | 不参与 | 低风险、高频操作 |

大多数 Agent 项目一开始都采用 HITL 模式，逢操作必确认。下一步演进方向是 HOTL——Agent 自主执行，只有出现异常时才通知人类介入。

### HOTL 实现模式

```
Agent executes task
  |
  |-- Low-risk operation  -> Execute directly -> Log it
  |
  |-- Medium-risk operation -> Execute -> Notify human (rollback available)
  |
  +-- High-risk operation -> Pause -> Wait for human approval -> Execute only if approved
```

## OWASP Agent Security Top 10（2026）

| 排名 | 风险 | 说明 |
|------|------|-------------|
| 1 | Prompt Injection | 恶意输入劫持 Agent 行为 |
| 2 | Tool Misuse | Agent 调用了不该调用的工具 |
| 3 | Privilege Escalation | Agent 突破授权范围 |
| 4 | Data Leakage | Agent 泄露敏感信息 |
| 5 | Insecure Output | Agent 输出中包含恶意内容 |
| 6 | Supply Chain | 恶意第三方工具或插件 |
| 7 | Denial of Service | Agent 被诱导消耗大量资源 |
| 8 | Model Manipulation | 对抗样本操纵 Agent 决策 |
| 9 | Logging Gaps | 缺乏审计追踪 |
| 10 | Insecure Integration | 与外部系统的连接不安全 |

## 推荐阅读

| 资源 | 说明 |
|----------|-------------|
| [OWASP Top 10 for Agentic Applications 2026](https://www.practical-devsecops.com/owasp-top-10-agentic-applications/) | 威胁模型 Top 10 |
| [OWASP Agent Security Risks (Medium)](https://medium.com/@oracle_43885/owasps-ai-agent-security-top-10-agent-security-risks-2026-fc5c435e86eb) | 逐条详解 |
| [Northflank: How to Sandbox AI Agents](https://northflank.com/blog/how-to-sandbox-ai-agents) | 沙箱技术选型指南 |
| [Northflank: Best Code Execution Sandbox 2026](https://northflank.com/blog/best-code-execution-sandbox-for-ai-agents) | 托管方案对比 |
| [Agent Sandboxes: Practical Guide](https://www.vietanh.dev/blog/2026-02-02-agent-sandboxes) | 动手实操教程 |
| [Firecrawl: AI Agent Sandbox](https://www.firecrawl.dev/blog/ai-agent-sandbox) | 完整安全指南 |
| [AI Agent Sandboxing: Firecracker, gVisor](https://manveerc.substack.com/p/ai-agent-sandboxing-guide) | 隔离技术深度解析 |
| [CodeAnt: Sandbox LLMs & AI Shell Tools](https://www.codeant.ai/blogs/agentic-rag-shell-sandboxing) | Docker + gVisor 实操 |
| [Google GKE Agent Sandbox](https://docs.google.com/kubernetes-engine/docs/how-to/agent-sandbox) | K8s 原生方案 |
| [Bunnyshell: Sandboxed Environments for AI Coding](https://www.bunnyshell.com/guides/sandboxed-environments-ai-coding/) | 综合指南 |
| [Permit.io: HITL Best Practices](https://www.permit.io/blog/human-in-the-loop-for-ai-agents-best-practices-frameworks-use-cases-and-demo) | 人机协作设计 |
| [From HITL to HOTL](https://bytebridge.medium.com/from-human-in-the-loop-to-human-on-the-loop-evolving-ai-agent-autonomy-c0ae62c3bf91) | 自主性演进 |
| [Security for Production AI Agents 2026](https://iain.so/security-for-production-ai-agents-in-2026) | 生产环境安全实践 |
| [Strata: Agentic AI Risks 2026](https://www.strata.io/blog/agentic-identity/a-guide-to-agentic-ai-risks-in-2026/) | 风险全景 |
| [OpenClaw Agent Permissions & Safety](https://www.crewclaw.com/blog/openclaw-agent-permissions-safety) | OpenClaw 权限模型 |

## 实践计划

1. **第 1 周**：通读 OWASP Top 10 和 Northflank 沙箱指南
2. **第 1 周**：为一个 Agent 项目搭建基础 Docker 沙箱（断网 + 只读）
3. **第 2 周**：设计权限模型（画白板），定义每个工具的权限边界
4. **第 2 周**：实现审批网关，高风险操作暂停等待人类确认
5. **第 3 周**：试用 E2B 或 gVisor 替代纯 Docker 方案
6. **第 3 周**：为一个多 Agent 系统加上完整的安全层
