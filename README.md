# Codex Bridge

Bridge between Claude Code and OpenAI's Codex CLI via the Model Context Protocol (MCP). Ask Codex questions, get second opinions, delegate subtasks, or use Codex to refine planning prompts — all without leaving Claude Code.

## Features

- **`/codex` command** — Send a prompt to Codex directly from Claude Code, with multi-turn session support via `threadId`.
- **`/claudex` command** — Iteratively refine a planning prompt through Claude↔Codex back-and-forth, then enter plan mode with the refined prompt.
- **Read-only by default** — Codex runs in a sandboxed read-only mode unless you explicitly opt in to writes.
- **Automatic skill triggering** — Claude Code recognises natural-language cues like "ask codex" or "claudex this" and invokes the right tool.
- **Zero custom server code** — Delegates entirely to the Codex CLI's built-in `mcp-server` mode.

## Prerequisites

- **Codex CLI** installed and on your `PATH`. On macOS:
  ```bash
  brew install --cask codex
  ```
  Or via npm:
  ```bash
  npm install -g @openai/codex
  ```
- Codex authenticated — run `codex login` (or `codex auth`, depending on your CLI version) once. Credentials are stored in `~/.codex/`.
- Claude Code CLI installed.

## Installation

The recommended install path is via the Contract Hero plugin marketplace:

```
/plugin marketplace add contract-hero/plugin-marketplace
/plugin install codex-bridge@contract-hero
```

Restart your Claude Code session after install — the plugin spawns `codex mcp-server` as a stdio process on session start.

## Usage

### `/codex` — Send a prompt to Codex

```
/codex Explain the authentication flow in src/auth/
```

Claude Code sends the prompt to Codex and presents the response with a thread identifier. Subsequent `/codex` calls in the same conversation automatically continue the thread.

You can also trigger Codex via natural language — Claude Code's skill system recognises intent:

- "Ask codex how the caching layer works"
- "Get a second opinion from codex on this approach"
- "Send this function to codex for review"

### `/claudex` — Refine a planning prompt, then plan

```
/claudex Add real-time notifications to the dashboard
```

`/claudex` runs a 3–5 round Claude↔Codex refinement protocol that produces a high-quality planning prompt, then drops you into Claude Code's plan mode with that refined prompt as the planning task. Use it when you want a thorough investigation-and-planning pass before any code is written.

Natural-language triggers include "plan how to…", "investigate and plan…", "design an approach for…".

## How it works

```
┌─────────────┐       stdio        ┌──────────────────┐       API       ┌─────────┐
│ Claude Code │ ──── MCP ────────▶ │ codex mcp-server │ ─────────────▶ │ OpenAI  │
│  (plugin)   │ ◀─── JSON-RPC ──── │   (Codex CLI)    │ ◀───────────── │   API   │
└─────────────┘                    └──────────────────┘                └─────────┘
```

1. **Plugin registration** — `plugin.json` declares an MCP server entry that runs `codex --dangerously-bypass-approvals-and-sandbox mcp-server` over stdio.
2. **Session start** — Claude Code spawns the Codex MCP server process when a session begins.
3. **Tool exposure** — The server exposes `mcp__codex__codex` (start a session) and `mcp__codex__codex-reply` (continue a session).
4. **Multi-turn flow** — Each response includes a `threadId`. Passing it to subsequent calls maintains conversation context on the Codex side.
5. **Sandbox policy** — Defaults to `read-only`. If Codex suggests edits, Claude Code offers to apply them using its own tools.

### Plugin layout

```
codex-bridge/
├── .claude-plugin/
│   └── plugin.json            # Manifest: MCP server + commands + skills
├── commands/
│   ├── codex.md               # /codex slash command
│   └── claudex.md             # /claudex slash command
└── skills/
    ├── codex-bridge/SKILL.md  # When/how to invoke /codex
    └── claudex/SKILL.md       # When/how to invoke /claudex
```

## MCP tools reference

### `mcp__codex__codex` — Start a new session

Returns `{ threadId, content }`.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prompt` | Yes | The prompt to send to Codex |
| `sandbox` | No | `"read-only"` (default), `"workspace-write"`, or `"danger-full-access"` |
| `model` | No | Model override (e.g., `"gpt-5.3-codex"`, `"o3"`) |
| `cwd` | No | Working directory for the session |
| `developer-instructions` | No | System-level instructions for Codex |
| `approval-policy` | No | `"untrusted"`, `"on-failure"`, `"on-request"`, `"never"` |
| `profile` | No | Configuration profile from Codex's `config.toml` |

### `mcp__codex__codex-reply` — Continue a session

Returns `{ threadId, content }`.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prompt` | Yes | The follow-up prompt |
| `threadId` | Yes | Thread ID from a previous response |

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Tool not found (`mcp__codex__codex`) | Plugin not installed or MCP server failed to start | Re-install the plugin and restart the session. Verify `codex` is on your `PATH`. |
| Auth error | Codex credentials missing or expired | Run `codex login` (or `codex auth`) in your terminal |
| Empty response | Codex rate-limited or API error | Retry after a moment |
| Slow response (>60s) | Complex task or large codebase scan | Normal for Codex; wait or simplify the prompt |

## Notes

- **Cost** — Each Codex call consumes your OpenAI API quota. `/claudex` makes 3–5 calls per invocation.
- **Session isolation** — Codex sessions are independent from Claude Code's conversation. The `threadId` only maintains context on the Codex side.
- **Auth management** — Claude Code does not manage Codex credentials. Authentication is handled entirely by the Codex CLI.
- **Process lifetime** — The MCP server process lives for the duration of the Claude Code session and is terminated when the session ends.

## License

Apache-2.0 — see [LICENSE](./LICENSE).
