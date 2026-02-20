# Project Aegis (The Shield) 🛡️

**A transparent security proxy for the Model Context Protocol.**

## Overview
Aegis is the **Security Control Plane** of the Kagi ecosystem. It acts as a MITM proxy for all MCP tool calls, enforcing granular security policies and human-in-the-loop approvals.

## The Problem
AI Agents (using Claude, Codex, etc.) connect to sensitive tools via MCP (Model Context Protocol). Once connected, the Agent has full authority. If the Agent is hallucinating or compromised (Prompt Injection), it can execute dangerous actions (e.g., `delete_file`, `send_money`) without user consent.

## The Solution: Aegis
A "Middle-Man" MCP Server that wraps the real tool.

**Flow:**
`[AI Client]` <--> `[Aegis]` <--> `[Real MCP Server]`

1.  **Mirroring:** Aegis queries the Real Server for its tools (`tools/list`) and exposes identical definitions to the AI.
2.  **Interception:** When the AI calls `tools/call`, Aegis catches it.
3.  **Approval:** Aegis triggers a user interaction (CLI prompt, Desktop Notification, Web UI).
    > *"Agent wants to call `delete_user(id=5)`. Allow?"*
4.  **Forwarding:**
    - If **Approved**: Aegis calls the Real Server and returns the result.
    - If **Denied**: Aegis returns a `User denied execution` error to the AI.

## Ecosystem Integration
- Integrates with **Minato** for Human-in-the-Loop approvals via user channels.
- Uses **Kura** to store "Local Trust" records — verified tool versions cached locally for speed and security.
- Performs "Skill Auditing" to ensure third-party tools are safe.

## Architecture (Proposed)
- **Language:** Go (robust JSON-RPC handling, single binary).
- **Transport:** Stdio (Standard Input/Output) to act as a seamless drop-in replacement in `config.json`.
- **Policy Engine:** Rego/OPA or simple JSON rules (Allow/Deny/Ask).
- **UI:** A local lightweight server for approvals.

## Usage
Instead of configuring Claude to use `gmail-mcp`, you configure it to use:
`aegis --target="gmail-mcp"`
