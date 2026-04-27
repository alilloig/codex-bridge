---
allowed-tools:
  - mcp__codex__codex
  - mcp__codex__codex-reply
description: "Send a prompt to Codex CLI and return the response. Supports multi-turn sessions."
---

# /codex Command

Send the user's prompt to OpenAI Codex via the `mcp__codex__codex` or `mcp__codex__codex-reply` MCP tools.

## Arguments

$ARGUMENTS contains the prompt to send. If empty, ask the user what they want to send to Codex.

## Session Continuity

- If a `threadId` from a previous Codex call exists in this conversation, use `mcp__codex__codex-reply` with that `threadId` to continue the session.
- If no previous `threadId` exists, use `mcp__codex__codex` to start a new session.
- Always remember the `threadId` from each response for subsequent calls.

## Default Parameters

When starting a new session with `mcp__codex__codex`, always include:

```json
{
  "prompt": "<user prompt>",
  "sandbox": "read-only",
  "cwd": "<current working directory>"
}
```

Only change `sandbox` to `"workspace-write"` if the user explicitly asks Codex to write or edit files.

## Execution

1. Determine whether to start a new session or continue an existing one.
2. Call the appropriate MCP tool with the prompt from $ARGUMENTS.
3. Store the returned `threadId` for future `/codex` calls.
4. Present the `content` from Codex's response to the user.
5. If the response references file changes or commands Codex would have run, note that Codex was in read-only mode and offer to apply the changes via Claude Code's own tools if the user wants.

## Output Format

Present Codex's response clearly, prefixed with a brief note:

> **Codex** (thread `<short-id>`):
>
> <response content>
