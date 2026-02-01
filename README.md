# MCP Firewall 🛡️🔥

**A transparent security proxy for the Model Context Protocol.**

## The Problem
AI Agents (using Claude, Codex, etc.) connect to sensitive tools via MCP (Model Context Protocol). Once connected, the Agent has full authority. If the Agent is hallucinating or compromised (Prompt Injection), it can execute dangerous actions (e.g., `delete_file`, `send_money`) without user consent.

## The Solution: MCP Firewall
A "Middle-Man" MCP Server that wraps the real tool.

**Flow:**
`[AI Client]` <--> `[MCP Firewall]` <--> `[Real MCP Server]`

1.  **Mirroring:** The Firewall queries the Real Server for its tools (`tools/list`) and exposes identical definitions to the AI.
2.  **Interception:** When the AI calls `tools/call`, the Firewall catches it.
3.  **Approval:** The Firewall triggers a user interaction (CLI prompt, Desktop Notification, Web UI).
    > *"Agent wants to call `delete_user(id=5)`. Allow?"*
4.  **Forwarding:**
    - If **Approved**: The Firewall calls the Real Server and returns the result.
    - If **Denied**: The Firewall returns a `User denied execution` error to the AI.

## Architecture (Proposed)
- **Language:** TypeScript or Go (needs robust JSON-RPC handling).
- **Transport:** Stdio (Standard Input/Output) to act as a seamless drop-in replacement in `config.json`.
- **UI:** A local lightweight server for approvals.

## Usage
Instead of configuring Claude to use `gmail-mcp`, you configure it to use:
`mcp-firewall --target="gmail-mcp"`
