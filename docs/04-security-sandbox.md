# 04 — Security & Sandboxing

## Overview

Agents can run code, call APIs, and write files — and they can also cause damage. A single prompt injection can make an Agent delete a database. In 2026, OWASP published a dedicated Agent Security Top 10.

## Three-Layer Security Architecture

```
+----------------------------------+
|  Layer 1: Code Execution Sandbox |  <-- Where Agent-generated code runs
+----------------------------------+
|  Layer 2: Permissions & ACL      |  <-- What the Agent can and cannot do
+----------------------------------+
|  Layer 3: Human-Agent Gating     |  <-- Which operations need human approval
+----------------------------------+
```

## Layer 1: Code Execution Sandbox

Agent-generated code **must never** run directly on the host machine. The 2026 consensus: standard Docker isolation is no longer sufficient.

### Three Isolation Levels

| Technology | Isolation Level | Startup Time | Overhead | Who Uses It |
|------------|----------------|-------------|----------|-------------|
| **Docker/runc** | Weak (shared kernel) | Seconds | Lowest | Development environments |
| **gVisor** | Medium (userspace kernel) | Seconds | I/O +10-30% | Google GKE Agent Sandbox |
| **Firecracker MicroVM** | Strong (independent kernel) | ~125ms | Memory <5MB | AWS Lambda |

### Selection Advice

- **Personal projects / learning**: Docker + no network + read-only filesystem
- **Production environments**: gVisor
- **Highest security requirements**: Firecracker

### Hands-On: Minimal Sandbox

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

### Managed Solutions (Don't Want to Self-Host)

| Solution | Type | Highlights |
|----------|------|-----------|
| [E2B](https://e2b.dev) | Managed API | Create a sandbox with one line of code |
| [microsandbox](https://github.com/nicholasgasior/microsandbox) | Self-hosted | Open source, lightweight |
| [Google GKE Agent Sandbox](https://docs.google.com/kubernetes-engine/docs/how-to/agent-sandbox) | K8s-native | gVisor isolation |
| [Modal](https://modal.com) | Managed | Pay-per-use, auto-scaling |

## Layer 2: Permissions & Access Control

### Permission Model Design

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

### Tool-Level Permissions

Each MCP tool gets individually authorized:

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

## Layer 3: Human-Agent Gating

### HITL to HOTL Evolution

| Mode | Full Name | Human Role | Best For |
|------|-----------|-----------|----------|
| **HITL** | Human-in-the-Loop | Confirms every step | High-risk, low-frequency |
| **HOTL** | Human-on-the-Loop | Supervises; intervenes on anomalies | Medium-risk, medium-frequency |
| **Full Auto** | — | Not involved | Low-risk, high-frequency |

Most Agent projects start with HITL (confirmation gating). The next step is learning HOTL — the Agent runs autonomously, and humans are notified only when something looks wrong.

### HOTL Implementation Pattern

```
Agent executes task
  |
  |-- Low-risk operation  -> Execute directly -> Log it
  |
  |-- Medium-risk operation -> Execute -> Notify human (rollback available)
  |
  +-- High-risk operation -> Pause -> Wait for human approval -> Execute only if approved
```

## OWASP Agent Security Top 10 (2026)

| Rank | Risk | Description |
|------|------|-------------|
| 1 | Prompt Injection | Malicious input hijacks Agent behavior |
| 2 | Tool Misuse | Agent calls tools it shouldn't |
| 3 | Privilege Escalation | Agent exceeds its authorized scope |
| 4 | Data Leakage | Agent exposes sensitive information |
| 5 | Insecure Output | Agent output contains malicious content |
| 6 | Supply Chain | Malicious third-party tools/plugins |
| 7 | Denial of Service | Agent is tricked into consuming excessive resources |
| 8 | Model Manipulation | Adversarial samples manipulate Agent decisions |
| 9 | Logging Gaps | Lack of audit trails |
| 10 | Insecure Integration | Insecure connections with external systems |

## Essential Reading

| Resource | Description |
|----------|-------------|
| [OWASP Top 10 for Agentic Applications 2026](https://www.practical-devsecops.com/owasp-top-10-agentic-applications/) | Threat model Top 10 |
| [OWASP Agent Security Risks (Medium)](https://medium.com/@oracle_43885/owasps-ai-agent-security-top-10-agent-security-risks-2026-fc5c435e86eb) | Detailed breakdown |
| [Northflank: How to Sandbox AI Agents](https://northflank.com/blog/how-to-sandbox-ai-agents) | Sandbox technology selection guide |
| [Northflank: Best Code Execution Sandbox 2026](https://northflank.com/blog/best-code-execution-sandbox-for-ai-agents) | Managed solution comparison |
| [Agent Sandboxes: Practical Guide](https://www.vietanh.dev/blog/2026-02-02-agent-sandboxes) | Hands-on tutorial |
| [Firecrawl: AI Agent Sandbox](https://www.firecrawl.dev/blog/ai-agent-sandbox) | Complete security guide |
| [AI Agent Sandboxing: Firecracker, gVisor](https://manveerc.substack.com/p/ai-agent-sandboxing-guide) | Isolation technology deep dive |
| [CodeAnt: Sandbox LLMs & AI Shell Tools](https://www.codeant.ai/blogs/agentic-rag-shell-sandboxing) | Docker + gVisor hands-on |
| [Google GKE Agent Sandbox](https://docs.google.com/kubernetes-engine/docs/how-to/agent-sandbox) | K8s-native solution |
| [Bunnyshell: Sandboxed Environments for AI Coding](https://www.bunnyshell.com/guides/sandboxed-environments-ai-coding/) | Comprehensive guide |
| [Permit.io: HITL Best Practices](https://www.permit.io/blog/human-in-the-loop-for-ai-agents-best-practices-frameworks-use-cases-and-demo) | Human-Agent collaboration design |
| [From HITL to HOTL](https://bytebridge.medium.com/from-human-in-the-loop-to-human-on-the-loop-evolving-ai-agent-autonomy-c0ae62c3bf91) | Autonomy evolution |
| [Security for Production AI Agents 2026](https://iain.so/security-for-production-ai-agents-in-2026) | Production security practices |
| [Strata: Agentic AI Risks 2026](https://www.strata.io/blog/agentic-identity/a-guide-to-agentic-ai-risks-in-2026/) | Risk landscape |
| [OpenClaw Agent Permissions & Safety](https://www.crewclaw.com/blog/openclaw-agent-permissions-safety) | OpenClaw permission model |

## Practice Plan

1. **Week 1**: Read OWASP Top 10 + Northflank sandbox guide
2. **Week 1**: Build a basic Docker sandbox for an Agent project (no network + read-only)
3. **Week 2**: Design a permission model (whiteboard it) — define each tool's permission boundaries
4. **Week 2**: Implement an approval gateway — high-risk operations pause and wait for human confirmation
5. **Week 3**: Try E2B or gVisor as a Docker replacement
6. **Week 3**: Add a complete security layer to a multi-Agent system
