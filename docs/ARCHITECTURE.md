# MCP Firewall Architecture

## Overview

The MCP Firewall is a transparent MITM (Man-In-The-Middle) proxy written in Go that sits between
AI agents (clients) and MCP tool servers. It intercepts JSON-RPC 2.0 `tools/call` requests,
enforces configurable policies, and gates dangerous operations behind human approval.

**Core Principles:**
- Zero-modification to existing tools. The firewall wraps any MCP server transparently.
- Default-deny for unknown tools. Explicit policy is required to allow tool execution.
- Human-in-the-loop for sensitive operations. Multiple approval adapters (terminal, desktop, Telegram).
- Stateless policy evaluation. No database. Policy is a single YAML file.

**Integration Points:**
- **Hashi:** Hashi spawns all tools via `mcp-firewall run -- <tool-command>`. The firewall is
  invisible to the LLM -- it sees a normal MCP server. Hashi's config specifies the firewall
  binary path and policy file.
- **Sora-Link:** Sora-Link routes every CLI command through the firewall before execution.
  The firewall's Telegram approval adapter can send approval requests directly to the same
  Telegram chat that Sora-Link uses, creating a unified approval experience.

## Directory Structure

```
/
├── cmd/
│   └── mcp-firewall/
│       └── main.go                # Entry point, CLI argument parsing, signal handling
├── pkg/
│   ├── proxy/                     # Core proxy engine
│   │   ├── proxy.go               # Transport-agnostic proxy loop
│   │   ├── stdio.go               # Stdio transport (stdin/stdout pipe)
│   │   ├── sse.go                 # SSE/HTTP transport
│   │   ├── inspector.go           # JSON-RPC stream parser and classifier
│   │   ├── buffer.go              # Request buffering for approval flow
│   │   └── jsonrpc.go             # JSON-RPC 2.0 types, serialization, validation
│   ├── policy/                    # Policy engine
│   │   ├── engine.go              # Policy evaluation logic
│   │   ├── loader.go              # YAML policy loader with schema validation
│   │   ├── rules.go               # Rule types, matching logic, condition evaluation
│   │   └── condition.go           # Safe condition expression evaluator (CEL-based)
│   ├── approval/                  # Human approval adapters
│   │   ├── adapter.go             # Approval adapter interface
│   │   ├── terminal.go            # Local: interactive terminal prompt
│   │   ├── desktop.go             # Local: OS notification with web confirmation page
│   │   ├── telegram.go            # Remote: Telegram inline keyboard approval
│   │   └── timeout.go             # Approval timeout and default-deny logic
│   ├── audit/                     # Audit logging
│   │   ├── logger.go              # Structured audit log writer
│   │   └── types.go               # Audit event types
│   └── health/                    # Health and diagnostics
│       └── check.go               # Startup verification, status endpoint
├── configs/
│   └── firewall.yaml              # Default configuration
├── docs/
│   └── ARCHITECTURE.md
└── tests/
    ├── integration/
    │   ├── stdio_test.go          # End-to-end stdio proxy tests
    │   └── policy_test.go         # Policy evaluation tests
    └── fixtures/
        ├── echo-server/           # Mock MCP server for testing
        └── policies/              # Test policy files
```

## Phased Approach

### Phase 1: Transparent Shield (MVP) -- Current Focus

A transparent wrapper that intercepts tool calls without managing tool installation or discovery.

**Scope:**
- Stdio transport only (most common MCP server transport).
- Single-tool proxying (one firewall instance per tool).
- Stateless policy evaluation from a YAML file.
- Human approval via terminal, desktop notification, or Telegram.
- Audit logging to a local file.

**Registration:** The firewall wraps an existing tool command:
```bash
mcp-firewall run -- npx -y @modelcontextprotocol/server-filesystem /home/user/documents
```

The AI agent's MCP client configuration points to `mcp-firewall` instead of the tool directly:
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "mcp-firewall",
      "args": ["run", "--", "npx", "-y", "@modelcontextprotocol/server-filesystem", "/home/user/documents"]
    }
  }
}
```

### Phase 2: Agentic Hub (Future)

A single aggregation endpoint that multiplexes access to multiple MCP servers.

**Scope:**
- WebSocket endpoint (`ws://localhost:9090`) that agents connect to.
- Tool discovery: agents see a unified tool list from all registered servers.
- Profiles: named policy sets (e.g., "development", "production", "readonly").
- Package manager: install/update MCP servers from a registry.
- Vault: centralized secret injection for tool servers.
- Dashboard: web UI for real-time monitoring, policy editing, and approval history.

**Not designed yet.** Phase 2 is contingent on Phase 1 stability.

## Technical Design (Phase 1)

### 1. Core Proxy (`pkg/proxy`)

The proxy operates as a bidirectional pipe between the AI agent (client) and the MCP tool (server).

**Stdio Transport Flow:**

```
Agent (stdin/stdout) <---> Firewall Proxy <---> Real MCP Server (stdin/stdout)
```

1. The firewall spawns the real MCP server as a child process.
2. The firewall reads JSON-RPC messages from its own stdin (sent by the agent).
3. The Inspector classifies each message:
   - **Pass-through:** `initialize`, `tools/list`, `resources/list`, `notifications/*`, any
     response (`result`/`error`). Forwarded immediately to the server.
   - **Intercept:** `tools/call`. Buffered and sent to the Policy Engine.
4. The Policy Engine evaluates the tool call against the policy rules.
5. Based on the policy decision:
   - **ALLOW:** Forward to the real server immediately.
   - **DENY:** Return a JSON-RPC error response to the agent. Do not forward.
   - **ASK:** Trigger the Approval Adapter. Buffer the request until approval/denial/timeout.
6. The firewall reads responses from the real server's stdout and forwards them to the agent's stdout.

**JSON-RPC Handling:**

- **Batch Requests:** JSON-RPC 2.0 allows batch requests (array of messages). The Inspector
  splits batches and evaluates each `tools/call` individually. Non-tool-call messages in the
  batch are forwarded immediately. The batch response is reassembled.
- **Notifications:** JSON-RPC notifications (no `id` field) are always passed through. They
  cannot be meaningfully blocked since the sender does not expect a response.
- **Malformed Messages:** Invalid JSON or non-JSON-RPC messages are logged as audit events
  and result in a JSON-RPC Parse Error (-32700) response to the sender. They are never
  forwarded to the server.
- **Message Ordering:** The proxy preserves message ordering. When a `tools/call` is buffered
  for approval, subsequent messages from the same direction are queued behind it to prevent
  out-of-order delivery. Messages from the server direction continue flowing (the server may
  send notifications independently).

**Concurrency:**
- Two goroutines handle the bidirectional pipe (agent-to-server, server-to-agent).
- A mutex protects the write side of each pipe to prevent interleaved JSON-RPC messages.
- Buffered tool calls awaiting approval are stored in a `sync.Map` keyed by JSON-RPC request ID.

**Child Process Management:**
- The real MCP server is spawned via `os/exec.Cmd` with stdin/stdout piped.
- The firewall monitors the child process. If it exits unexpectedly:
  1. Log the exit code and stderr output.
  2. Attempt to restart the child (max 3 attempts with 2-second backoff).
  3. If restart fails, send a JSON-RPC InternalError to the agent and exit.
- On firewall shutdown (SIGINT/SIGTERM):
  1. Send SIGTERM to the child process.
  2. Wait up to 5 seconds for graceful exit.
  3. Send SIGKILL if the child has not exited.
  4. Drain any pending approval requests with DENY.

### 2. Policy Engine (`pkg/policy`)

The Policy Engine evaluates tool calls against a set of rules defined in `firewall.yaml`.

**Evaluation Order:**
1. Rules are evaluated top-to-bottom. First matching rule wins.
2. If no rule matches, the `default` action applies.
3. The default action itself defaults to `ask` if not specified (fail-safe).

**Rule Matching:**
- `tool`: Glob pattern matched against the tool name. Supports `*` (any characters) and `?`
  (single character). Examples: `read_*`, `write_file`, `*`.
- `action`: `allow`, `deny`, or `ask`.
- `condition` (optional): A CEL (Common Expression Language) expression evaluated against the
  tool call arguments. CEL is used because it is sandboxed, side-effect-free, and well-specified.

**Condition Evaluation:**

Conditions are evaluated using Google's CEL-Go library. The expression environment exposes:
- `args`: The tool call arguments as a map.
- `tool`: The tool name as a string.

Examples:
```yaml
# Allow read_file only for paths under /home
- tool: "read_file"
  action: allow
  condition: 'args.path.startsWith("/home")'

# Deny write_file to /etc
- tool: "write_file"
  action: deny
  condition: 'args.path.startsWith("/etc")'

# Ask for any tool call with more than 1000 characters of input
- tool: "*"
  action: ask
  condition: 'size(string(args)) > 1000'
```

**Security:** CEL expressions cannot perform I/O, access the network, or execute arbitrary code.
The CEL environment is configured with no custom functions beyond the standard library. Malformed
CEL expressions cause the rule to be treated as a syntax error at load time (fail-fast), not at
evaluation time.

**Policy Validation:**
- On load, the policy file is validated against a JSON Schema.
- All CEL expressions are compiled at load time. Compilation errors are fatal (the firewall
  will not start with an invalid policy).
- The policy file is watched for changes. On modification, the new policy is validated and
  hot-reloaded. If validation fails, the old policy remains active and a warning is logged.

### 3. Approval Adapters (`pkg/approval`)

When a policy rule evaluates to `ask`, the firewall triggers the configured approval adapter.

**Adapter Interface:**
```go
type ApprovalAdapter interface {
    RequestApproval(ctx context.Context, req ApprovalRequest) (ApprovalDecision, error)
}

type ApprovalRequest struct {
    RequestID   string            // JSON-RPC request ID
    ToolName    string            // e.g., "write_file"
    Arguments   map[string]any    // Tool call arguments
    Timestamp   time.Time
}

type ApprovalDecision int
const (
    Approved ApprovalDecision = iota
    Denied
    TimedOut
)
```

**Terminal Adapter:**
- Prints tool name and arguments to stderr (not stdout, which is the JSON-RPC pipe).
- Prompts the user with `[A]llow / [D]eny / [A]llow all for this tool?`
- "Allow all" adds a temporary in-memory rule for the tool name that auto-approves for the
  remainder of the session (not persisted to policy.yaml).

**Desktop Adapter:**
- Sends an OS notification (notify-send / osascript) with a summary.
- Opens a local web page (`http://localhost:<random-port>/approve/<request-id>`) with full
  details and Allow/Deny buttons.
- The web server is ephemeral and only serves approval pages. It binds to localhost only.

**Telegram Adapter:**
- Sends a message to a configured Telegram chat with an inline keyboard (Allow / Deny buttons).
- Uses the Telegram Bot API. The bot token and chat ID are configured in `firewall.yaml`.
- The adapter polls for callback query updates to detect button presses.
- **Security:** The adapter verifies that the callback query comes from an authorized user ID
  (configured allowlist). Unauthorized button presses are ignored.

**Timeout Behavior:**
- All adapters have a configurable timeout (default: 5 minutes).
- If the timeout expires without a decision, the request is **denied** (fail-safe).
- The agent receives a JSON-RPC error: `{\"code\": -32001, \"message\": \"Tool call denied: approval timed out\"}`.
- The timeout is communicated to the user in the approval prompt ("Auto-deny in 4:59...").

**Adapter Fallback:**
- If the configured adapter fails to deliver the approval request (e.g., Telegram API is
  unreachable), the firewall falls back to `deny` and logs an error.
- Future: configurable fallback chain (e.g., try Telegram, fall back to terminal).

### 4. Audit Logging (`pkg/audit`)

Every intercepted tool call is logged to an append-only audit file.

**Log Format:** JSON Lines (one JSON object per line).

**Audit Event Fields:**
```json
{
  "timestamp": "2026-02-07T14:30:00Z",
  "event": "tool_call",
  "tool": "write_file",
  "arguments": {"path": "/home/user/test.txt", "content": "..."},
  "policy_action": "ask",
  "matched_rule": 2,
  "approval_decision": "approved",
  "approval_adapter": "telegram",
  "approval_latency_ms": 12340,
  "forwarded": true,
  "result_status": "success",
  "duration_ms": 450
}
```

**Behaviors:**
- Arguments that match secret patterns (API keys, tokens) are redacted in the audit log.
- The audit file is rotated daily and compressed (gzip). Retention: 30 days by default.
- Audit logging is synchronous (blocking). If the audit file cannot be written, the firewall
  logs to stderr and continues (audit failure does not block tool execution).

## Configuration

```yaml
# configs/firewall.yaml
version: "1"

proxy:
  transport: stdio              # stdio | sse (Phase 1: stdio only)
  child_restart_attempts: 3
  child_restart_backoff: 2s
  shutdown_grace_period: 5s

policy:
  default: ask                  # allow | deny | ask
  rules:
    - tool: "read_*"
      action: allow

    - tool: "write_file"
      action: ask
      condition: '!args.path.startsWith("/etc")'

    - tool: "delete_*"
      action: deny

    - tool: "execute_command"
      action: ask

    # Catch-all: anything not matched above uses the default action
  hot_reload: true              # Watch policy file for changes

approval:
  method: terminal              # terminal | desktop | telegram
  timeout: 5m
  telegram:
    bot_token: ${FIREWALL_TELEGRAM_BOT_TOKEN}
    chat_id: ${FIREWALL_TELEGRAM_CHAT_ID}
    authorized_users:           # Telegram user IDs allowed to approve
      - 123456789
  desktop:
    bind: "127.0.0.1:0"        # Random port on localhost

audit:
  enabled: true
  file: ~/.mcp-firewall/audit.jsonl
  rotation: daily
  retention_days: 30
  redact_secrets: true

logging:
  level: info                   # debug | info | warn | error
  output: stderr                # stderr (stdout is reserved for JSON-RPC)
```

## Error Handling Summary

| Scenario | Behavior |
|---|---|
| Malformed JSON from agent | Return Parse Error (-32700), log, do not forward |
| Unknown JSON-RPC method | Pass through (may be a custom extension) |
| Policy file missing at startup | Fatal error, refuse to start |
| Policy file invalid (bad CEL) | Fatal error, refuse to start |
| Policy file changes to invalid | Keep old policy, log warning |
| Approval adapter unreachable | Deny the request, log error |
| Approval timeout | Deny the request |
| Child process crashes | Restart up to 3 times, then exit |
| Child process hangs (no response) | Per-request timeout (60s default), return timeout error |
| Firewall receives SIGTERM | Drain pending approvals (deny all), terminate child, exit |
| Audit file unwritable | Log to stderr, continue operation |
| Concurrent tool calls | Each handled independently, keyed by request ID |

## Security Considerations

- **stdout is sacred:** The firewall NEVER writes non-JSON-RPC data to stdout. All diagnostic
  output goes to stderr. Violations would corrupt the agent-server communication.
- **No privilege escalation:** The firewall runs with the same privileges as the agent. It does
  not require root.
- **Condition injection:** CEL evaluation is sandboxed. There is no `eval()` or shell execution
  in condition processing.
- **Telegram approval spoofing:** The Telegram adapter validates `callback_query.from.id` against
  the authorized user list. An attacker who knows the bot token but is not in the authorized list
  cannot approve tool calls.
- **Localhost binding:** The desktop approval web server binds to `127.0.0.1` only. It is not
  accessible from the network.
