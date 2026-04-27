---
allowed-tools:
  - mcp__codex__codex
  - mcp__codex__codex-reply
  - EnterPlanMode
description: "Refine a prompt through multi-round Codex iteration, then enter plan mode"
argument-hint: "<task to plan>"
---

# /claudex Command

Use iterative Claude↔Codex refinement to generate a comprehensive planning prompt, then enter plan mode with the refined prompt.

## Arguments

`$ARGUMENTS` contains the task to plan. If empty, ask the user what engineering task they want to plan.

## Multi-Round Refinement Protocol

Execute these rounds in sequence. The goal is to produce a high-quality planning prompt that will guide Claude Code through codebase investigation, analysis, and detailed implementation planning.

### Round 1 — Codex Drafts v0 (Gate G0)

Codex is the first agent to touch the user input and produces a Claude-Code-tailored planning prompt as the seed for the rest of the protocol.

Call `mcp__codex__codex` with:

```json
{
  "sandbox": "read-only",
  "cwd": "<current working directory>",
  "prompt": "Create a refined prompt for Claude Code that is optimized for a Codex-to-Claude-Code handoff.\n\nThe prompt should instruct Claude Code to:\n- Treat the user request as a scoped engineering task that requires\n  repository investigation before proposing changes.\n- Explore the codebase efficiently to find the relevant files, entry\n  points, dependencies, configuration, and existing patterns.\n- Analyze the problem in context, including likely causes, affected systems\n  or flows, constraints, edge cases, and risks.\n- Produce a detailed implementation plan before writing code.\n- Make the plan specific, practical, and execution-ready, including:\n  - a short problem summary,\n  - relevant findings from the codebase,\n  - assumptions and unknowns,\n  - questions that must be clarified,\n  - ordered implementation steps,\n  - impacted files or modules,\n  - testing and validation steps,\n  - potential regressions or rollout concerns if applicable.\n- Use the AskUserQuestion tool whenever required information is missing,\n  ambiguous, or materially affects the implementation plan, rather than\n  silently making important assumptions.\n- Prefer resolving questions from the codebase first, and only use\n  AskUserQuestion when the answer cannot be determined reliably from the\n  repository or request.\n- Avoid coding immediately; focus on investigation, analysis, and planning.\n\nAddress this request: $ARGUMENTS\n\nOutput only the final prompt text itself, with no preamble, explanation, or markdown fences."
}
```

**Store the returned `threadId`** — every subsequent Codex call in this skill must reuse it via `mcp__codex__codex-reply`. The body of Codex's response is **v0**, the seed planning prompt.

`$ARGUMENTS` is interpolated into the JSON `prompt` value as a string; the MCP harness JSON-escapes embedded quotes and newlines before substitution.

### Round 2 — Claude critiques v0 → v1

No tool call. Internally review Codex's v0 and produce v1 by:

1. Keeping any well-structured sections from v0 that already serve the request
2. Tightening vague language and removing redundancy
3. Adding anything Codex omitted that the user's request clearly needs (domain-specific files, prior conventions visible in the working directory)
4. Preserving the "investigate before coding, ask before assuming" discipline of v0

v1 is Claude's first contribution and the input to G1a.

### Round 3 — Codex Critique (Gate G1a)

Call `mcp__codex__codex-reply` with the threadId from G0:

```json
{
  "threadId": "<threadId from G0>",
  "prompt": "Here is v1 — my revision of the v0 you just drafted.\n\n## Planning Prompt v1\n\n<paste v1 here>\n\n## Your Task\n\n1. Critique v1 against your v0. What did the revision improve, what did it lose, what is still missing or vague?\n2. Consider: Does it encourage thorough exploration? Will it produce an actionable plan? Does it handle edge cases?\n3. Generate an improved version (v2) that addresses your critique.\n4. Be specific about what you changed and why."
}
```

Parse Codex's response to extract:
- The critique points
- The improved v2 prompt

### Round 4 — Claude's v3 (Synthesis)

Review Codex's v2 and critique. Generate v3 by synthesizing:

1. **What Codex improved** that's genuinely better than v1
2. **What v1 had** that Codex's v2 lost or weakened
3. **New ideas** from v2 worth incorporating
4. **Contradictions** that need resolution

The v3 prompt should be the strongest combination of both versions. Don't just concatenate — synthesize intelligently.

### Round 5 — Codex Convergence Check (Gate G1b)

Call `mcp__codex__codex-reply` with:

```json
{
  "threadId": "<threadId from G0>",
  "prompt": "Here is planning prompt v3 (my synthesis of v1 and your v2):\n\n<paste v3 here>\n\nIs this converged? Critique any remaining weaknesses. If it's good enough to guide thorough investigation and planning, respond with exactly 'CONVERGED' at the start of your response. Otherwise, provide specific feedback on what still needs improvement."
}
```

### Round 6 (Optional — Gate G1c)

Only execute if:
- Codex's response does NOT start with "CONVERGED"
- The feedback identifies substantial gaps (not just stylistic preferences)

If needed:
1. Incorporate the final feedback into v4
2. Call `mcp__codex__codex-reply` one more time for confirmation
3. Proceed regardless of response (max 4 Codex calls)

## After Convergence

Once the prompt is finalized (via CONVERGED or max iterations):

1. **Call `EnterPlanMode`** to transition to plan mode

2. **Present the refined prompt** as the planning task:

```markdown
## Planning Task
*Refined via Claude↔Codex iteration*

<final refined prompt>
```

3. **Proceed with investigation and planning** following the refined prompt's instructions

## Error Handling

### Codex Unavailable
If `mcp__codex__codex` fails on G0:
1. Retry once after a brief pause
2. If still failing, skip G0 and have Claude draft the seed planning prompt directly using the same meta-instruction inline (the prompt body from Round 1, applied to `$ARGUMENTS`)
3. Note to user: "Codex unavailable — proceeding with Claude-only refinement"
4. The remaining G1a/G1b rounds are also skipped (no thread T to reply on); Claude proceeds straight to `EnterPlanMode` with the seed prompt as the planning task

### Empty or Malformed Response
If Codex returns empty or unparseable response:
1. Log the issue
2. Proceed with the best prompt version available
3. Continue the protocol

If Codex wraps v0 in markdown fences despite the trailing "Output only..." instruction, strip the outer fences before using v0; do not abort the protocol.

### Thread Lost Mid-Protocol
If the `threadId` from G0 is no longer valid between rounds (context compaction, MCP server restart, or any other cause):
1. Restart from G0 with a fresh `mcp__codex__codex` call
2. Do not attempt to resume the lost thread — Codex's server-side state is gone

## Output Format

During refinement, show progress:

```
**Claudex Refinement**

Round 1: Codex drafting v0 (thread `<short-id>`)...
Round 2: Critiquing v0 → v1...
Round 3: Codex critique of v1 received...
Round 4: Synthesizing v3...
Round 5: Checking convergence... ✓ CONVERGED

Entering plan mode with refined prompt.
```

## Example

User runs: `/claudex Add user authentication with JWT tokens to the Express API`

1. Codex drafts v0: a structured planning prompt covering JWT investigation, AskUserQuestion guidance, and execution-ready plan requirements (thread `T` opened)
2. Claude critiques v0 → v1: tightens scope, adds the project's existing Express middleware conventions
3. Codex critiques v1 → v2 (on thread `T`): "Missing consideration for refresh token strategy, token storage options, middleware placement"
4. Claude synthesizes v3 incorporating these points while keeping v1's focus on existing patterns
5. Codex confirms (on thread `T`): "CONVERGED"
6. Claude enters plan mode and begins investigating the codebase per the refined prompt
