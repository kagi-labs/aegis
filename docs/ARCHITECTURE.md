# Project Aegis Architecture 🛡️

## Overview
Project Aegis is the Universal Security Control Plane. It acts as a MITM proxy for MCP tool calls, enforcing human-in-the-loop approvals and static analysis of skills.

## Human Interface
Aegis does not manage its own bot connections. When a tool call requires human approval:
1. Aegis pauses the tool execution.
2. Aegis sends an approval request to **Project Minato**.
3. Minato routes that request to the user's preferred channel (Discord/Telegram/Web).
4. The user's decision is routed back through Minato to Aegis.

## Scope
- **Proxy Core:** Stdio/SSE interception.
- **Policy Engine:** CEL-based rule evaluation.
- **Skill Auditor:** Static analysis of \`SKILL.md\` files.
- **Local Trust:** Caching verified versions of tools and skills.
