---
name: claudex
description: |
  Multi-round prompt refinement for plan-mode execution via Claude↔Codex iteration.
  Use when:
  (1) User says "/claudex", "use claudex", or "claudex this"
  (2) User says "plan how to...", "investigate and plan...", "design an approach for..."
  (3) User says "refine this prompt for planning", "help me plan this properly"
  (4) User explicitly wants Codex to help improve their planning prompt before execution
  
  Do NOT trigger for:
  - Simple questions that don't require planning ("what does X do?")
  - Tasks the user wants executed immediately without planning
  - When user says "ask codex" or "/codex" (use codex-bridge skill instead)
  - Trivial tasks that don't benefit from detailed planning
allowed-tools:
  - mcp__codex__codex
  - mcp__codex__codex-reply
  - EnterPlanMode
author: alilloig
version: 1.1.0
date: 2026-04-27
---

# claudex — Multi-Round Planning Prompt Refinement

## Problem

Users want high-quality planning prompts that guide thorough codebase investigation and detailed implementation planning. Writing such prompts manually is tedious and often results in shallow investigation or incomplete plans.

## Solution

The `claudex` skill uses iterative Claude↔Codex refinement to generate comprehensive planning prompts, then enters plan mode with the refined prompt.

## When to Trigger

### Trigger Phrases (High Confidence)
- "/claudex ..."
- "use claudex to plan..."
- "claudex this task"
- "refine and plan..."

### Trigger Phrases (Medium Confidence)
- "plan how to..." (when task is non-trivial)
- "investigate and plan..."
- "design an approach for..."
- "help me plan this properly"
- "I need a detailed plan for..."

### Do NOT Trigger
- "ask codex..." → use `/codex` instead
- "what would codex say..." → use `/codex` instead
- Simple questions or lookups
- Tasks user wants done immediately
- Trivial changes (typo fixes, simple renames)

## Multi-Round Refinement Protocol

### Round 1 — Codex Drafts v0 (Gate G0)
Codex is the first agent to touch the user input.

Call `mcp__codex__codex` with:
- sandbox: "read-only"
- cwd: current working directory
- prompt: the meta-instruction telling Codex to produce a Claude-Code-ready planning prompt from the user's request (see `commands/claudex.md` for the full text)

Store the returned `threadId` — every subsequent Codex call in this skill reuses it via `mcp__codex__codex-reply`. The body of Codex's response is **v0**, the seed planning prompt.

### Round 2 — Claude critiques v0 → v1
No tool call. Tighten v0, fill gaps Codex missed, keep its investigate-before-coding discipline.

### Round 3 — Codex Critique (Gate G1a)
Call `mcp__codex__codex-reply` with:
- threadId: from G0
- prompt: Ask Codex to critique v1 against its own v0 and generate improved v2

### Round 4 — Claude's v3 (Synthesis)
Review Codex's v2 and critique. Generate v3 by:
- Keeping strongest elements from both versions
- Resolving contradictions
- Incorporating new valuable ideas

### Round 5 — Codex Convergence (Gate G1b)
Call `mcp__codex__codex-reply` with:
- threadId: from G0
- prompt: Ask if v3 is converged or needs more work

If response starts with "CONVERGED", proceed. Otherwise, optionally do one more round (Gate G1c, max 4 Codex calls total).

### After Convergence
1. Call `EnterPlanMode` tool
2. Present the refined prompt as the planning task
3. Proceed with investigation and planning

## Usage Examples

### Example 1: Explicit Command
```
User: /claudex Add real-time notifications to the dashboard

Claude: [Executes multi-round refinement]
→ Codex drafts v0 (thread T opened): structured planning prompt for the dashboard notifications task
→ Claude critiques v0 → v1: tightens scope to the existing dashboard runtime
→ Codex critique on v1 (thread T): "Missing WebSocket vs SSE consideration..."
→ Claude synthesizes v3: transport layer analysis added
→ Codex convergence check on thread T → CONVERGED
→ Enters plan mode with refined prompt
```

### Example 2: Natural Language Trigger
```
User: I need to plan how to add user authentication with OAuth

Claude: [Recognizes planning intent, triggers claudex]
→ Executes same multi-round protocol
→ Enters plan mode with comprehensive OAuth planning prompt
```

### Example 3: Non-Trigger (Use /codex Instead)
```
User: Ask codex what testing framework this project uses

Claude: [Does NOT trigger claudex — uses codex-bridge instead]
→ Sends question directly to Codex for answer
```

## Error Handling

- **Codex unavailable**: Retry once, then fall back to Claude drafting the seed planning prompt directly using the same meta-instruction inline (skip G0; quality degrades but the protocol still produces a refined prompt for plan mode)
- **Empty response**: Use best available version, continue protocol
- **Fenced v0**: If Codex wraps v0 in markdown fences despite the "Output only..." instruction, strip the outer fences and continue
- **Thread lost mid-protocol** (context compaction, MCP restart, etc.): Restart from G0 with a fresh `mcp__codex__codex` call — do not attempt to resume the lost thread
- **Max iterations**: Stop after 4 Codex calls regardless of convergence

## Comparison with /codex

| Aspect | /claudex | /codex |
|--------|----------|--------|
| Purpose | Refine prompts for Claude's planning | Execute tasks via Codex |
| Flow | Codex drafts → Claude/Codex iterate → Claude plans | Claude delegates → Codex executes |
| Result | Claude enters plan mode | Codex response returned |
| Use when | Complex tasks needing investigation | Quick Codex consultation |

## Notes

1. **Cost**: Each `/claudex` invocation makes 3-4 Codex API calls
2. **Latency**: Multi-round refinement takes ~30-60 seconds
3. **Quality**: Produces significantly better planning prompts than single-shot
4. **Fallback**: Works with Claude-only if Codex unavailable (reduced quality)
