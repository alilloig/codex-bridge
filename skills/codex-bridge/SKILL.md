---
name: codex-bridge
description: |
  Integration between Claude Code and OpenAI Codex CLI via MCP. Use when:
  (1) the user says "ask codex", "send to codex", "use codex", or "/codex",
  (2) the user wants a second opinion from another AI coding agent,
  (3) the user wants to compare Claude's approach with Codex's approach,
  (4) the user wants to delegate a subtask to Codex while staying in Claude Code.
  Provides multi-turn session support via threadId for persistent conversations
  with Codex within a single Claude Code session.
allowed-tools:
  - mcp__codex__codex
  - mcp__codex__codex-reply
author: alilloig
version: 1.0.0
date: 2026-03-09
---

# Codex Bridge — Claude Code ↔ Codex CLI Integration

## Problem

Users want to leverage OpenAI's Codex agent from within Claude Code without
switching terminals or managing separate sessions manually.

## Solution

The `codex-bridge` plugin registers `codex mcp-server` as an MCP stdio server,
exposing two tools:

### `mcp__codex__codex` — Start a New Session

Starts a fresh Codex conversation. Returns `{ threadId, content }`.

**Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prompt` | Yes | The user prompt to send to Codex |
| `sandbox` | No | `"read-only"` (default), `"workspace-write"`, or `"danger-full-access"` |
| `model` | No | Model override (e.g., `"gpt-5.3-codex"`, `"o3"`) |
| `cwd` | No | Working directory for the session |
| `developer-instructions` | No | System-level instructions for Codex |
| `approval-policy` | No | `"untrusted"`, `"on-failure"`, `"on-request"`, `"never"` |
| `profile` | No | Configuration profile from Codex's config.toml |

### `mcp__codex__codex-reply` — Continue a Session

Sends a follow-up prompt to an existing Codex thread.

**Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prompt` | Yes | The follow-up prompt |
| `threadId` | Yes | Thread ID from a previous `codex` or `codex-reply` response |

## Usage Pattern

### Starting a new session

```
Call mcp__codex__codex with:
{
  "prompt": "Explain the authentication flow in src/auth/",
  "sandbox": "read-only",
  "cwd": "/Users/alilloig/workspace/my-project"
}

Response: { "threadId": "019cd036-...", "content": "The auth flow works as follows..." }
```

### Continuing the conversation

```
Call mcp__codex__codex-reply with:
{
  "prompt": "What about the refresh token handling?",
  "threadId": "019cd036-..."
}

Response: { "threadId": "019cd036-...", "content": "The refresh token is handled by..." }
```

## Default Behavior

- **Sandbox**: Always use `"read-only"` unless the user explicitly requests Codex to write files.
- **Working directory**: Pass the current project's working directory as `cwd`.
- **Model**: Omit to use Codex's default (currently `gpt-5.3-codex`).
- **Thread reuse**: Always store and reuse `threadId` within a conversation for continuity.

## When to Use

- User explicitly requests Codex involvement ("ask codex", "what would codex say", "/codex")
- User wants a second opinion on an implementation approach
- User wants to delegate a specific subtask to Codex
- User wants to compare solutions from different AI agents

## When NOT to Use

- For tasks Claude Code can handle directly (default behavior)
- When the user hasn't expressed interest in using Codex
- For tasks requiring file writes (unless user explicitly overrides sandbox)

## Important Notes

1. **Cost**: Each Codex call uses the user's OpenAI API quota. Don't call Codex unnecessarily.
2. **Read-only default**: Codex runs in read-only sandbox. If it suggests edits, Claude Code should offer to apply them using its own Edit/Write tools.
3. **Session isolation**: Codex sessions are independent from Claude Code's conversation context. The threadId only maintains Codex-side context.
4. **Auth**: Codex uses credentials from `~/.codex/auth.json`. If auth fails, tell the user to run `codex auth` in their terminal.
5. **Timeout**: Complex prompts may take 30-60 seconds. This is normal for Codex.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Tool not found (`mcp__codex__codex`) | Plugin not installed or MCP server not started | Run `claude plugin add <path>` or check `codex` is on PATH |
| Auth error | Codex credentials expired | Run `codex auth` in terminal |
| Empty response | Codex rate limited or API error | Retry after a moment |
| Slow response (>60s) | Complex task or large codebase scan | Normal; wait or simplify prompt |
