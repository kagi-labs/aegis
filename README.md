# Project Aegis 🛡️

**A universal security control plane for AI agents.**

Aegis is the successor to the original **MCP Firewall** concept. It sits between AI clients
(Claude, Hashi, Kaji, or other local agents) and sensitive tools/MCP servers, enforcing
policy before tool calls reach the machine.

## The problem

AI agents can connect to powerful local tools through MCP or direct command execution. Once
connected, the agent may be able to read files, run shell commands, send messages, mutate
repositories, or trigger payments. If the agent hallucinates, gets prompt-injected, or loads a
malicious skill, those actions can happen without meaningful user consent.

## The solution

Aegis is a middle layer that proxies tool calls and applies a simple verdict model:

```text
allow | deny | ask | log
```

**Flow:**

```text
[AI Client] <--> [Aegis Proxy] <--> [Real MCP Server / Local Tool]
```

1. **Mirror:** Aegis discovers the target server's tool definitions (`tools/list`) and exposes compatible definitions to the AI client.
2. **Intercept:** When the AI calls `tools/call`, Aegis buffers the request.
3. **Evaluate:** The policy engine decides whether to allow, deny, ask the user, or only log.
4. **Approve:** For `ask`, Aegis sends an approval request through the configured UI/channel.
5. **Forward:** If approved, Aegis calls the real tool and returns the result. If denied, it returns a safe error.
6. **Audit:** Every decision is written to an audit log and, later, to Kura.

## Proposed architecture

- **Proxy core:** JSON-RPC/MCP stdio proxy first; SSE/WebSocket later.
- **Policy engine:** rule-based `allow`/`deny`/`ask`/`log` decisions from `aegis.yaml`.
- **Human approval:** terminal/web/channel adapters, with Minato as the channel bridge.
- **Audit log:** JSONL records locally; Kura integration once Kura has a storage API.
- **Skill auditor:** static inspection of `SKILL.md` files before agent loading.
- **Trust registry:** verified fingerprints for tools, skills, and policy versions.

## MVP usage target

The first useful version should be stdio-only and boring:

```bash
aegis proxy --policy ~/.aegis/policy.yaml -- target-mcp-command args...
```

or, for direct command wrapping:

```bash
aegis run --policy ~/.aegis/policy.yaml -- gh repo list kagi-labs
```

## Relationship to other Kagi Labs projects

- **Minato:** delivers approval requests and user decisions over Discord/Telegram/web channels.
- **Hashi:** routes flow/task tool execution through Aegis before touching local tools.
- **Kura:** stores audit records, trusted fingerprints, and long-term policy history.
- **Metsuke:** can feed review/security findings into Aegis policies.

## Roadmap

1. **Shield:** stdio proxy + JSON-RPC interceptor + CLI + JSONL audit log.
2. **Approvals:** web UI and Minato-backed Discord/Telegram approval adapters.
3. **Auditor:** static analysis of skill files and prompt safety checks.
4. **Kura integration:** durable audit logs and trusted fingerprints in Project Kura.
