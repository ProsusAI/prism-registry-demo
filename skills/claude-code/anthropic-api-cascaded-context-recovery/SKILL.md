---
name: anthropic-api-cascaded-context-recovery
description: "Anthropic API agentic streaming loop: how to recover from context-size errors
  (prompt-too-long 413, max-output-tokens) without terminating the session.
  TRIGGER when: designing or implementing any agentic loop that streams
  from the Anthropic API, when debugging sessions that terminate unexpectedly
  on context overflow, when deciding how to wire post-turn hooks, or when a
  team member asks why errors are withheld from the stream rather than surfaced
  immediately."
---

# Anthropic API: Cascaded Context Recovery
*Architectural decisions for recovering from context-size errors in agentic streaming loops — what to choose and why.*

## Who this is for
Teams building agentic loops that stream from the Anthropic API with tool use and long context. These decisions are non-obvious without production experience; each one has a tempting alternative that fails in a specific, identifiable way.

## Key decisions

### 1. Withhold recoverable API errors from stream consumers until all recovery paths are exhausted
When a prompt-too-long (413) or max-output-tokens error arrives mid-stream, do **not** yield it to the downstream consumer. Buffer it internally, attempt recovery, and only surface it if recovery fails completely.

**Why:** SDK consumers and UI layers that subscribe to the stream treat any `error` field as terminal — they close the session. Yielding an intermediate API error that the loop can still recover from kills the session before recovery has a chance to run.

**What breaks without it:** Any downstream consumer that calls your streaming function and handles errors by aborting will terminate the session on the first 413, even if compact + retry would have succeeded. The user loses the entire session context at the first overflow.

### 2. Cascade recovery cheapest-first: collapse → compact → surface
Order recovery attempts by cost: try the cheapest mechanism first, escalate only if it fails.

For context-size errors: (1) drain any staged context collapses (no API call, preserves granular context), (2) run reactive compact (one summarization API call), (3) surface the error.

For max-output-tokens: (1) silent one-shot retry at a higher token cap, (2) multi-turn recovery with explicit instructions (up to 3×), (3) surface.

**Why:** Compact is expensive (API call + tokens). Context collapse is free. Running them in reverse order wastes money and degrades context quality unnecessarily.

**What breaks without it:** Triggering full compaction for every overflow event wastes API budget and discards granular context that collapse could have preserved. Users see more compaction events than necessary.

### 3. Skip post-turn hooks entirely when the last message is an API error
If the model never produced a valid response (the turn ended with a rate-limit, prompt-too-long, or auth error), do not run post-turn hooks.

**Why:** Post-turn hooks commonly inject additional content (summaries, observations, follow-ups) into the message history. If the injection itself is what caused the overflow, re-running the hook after the error creates a death spiral: error → hook injects tokens → same error → hook fires again → infinite loop.

**What breaks without it:** Any hook that appends to the conversation on every turn will cause an infinite retry loop when the context is at the edge of overflow. The spiral burns API quota and never terminates.

### 4. Preserve the `hasAttemptedCompact` guard across stop-hook retries
When stop hooks inject blocking errors that cause the loop to continue, do **not** reset the "compact already attempted" flag. Treat it as sticky for the remainder of the recovery sequence.

**Why:** If compact ran, failed to recover (context still too large), and then a stop hook fires and injects a blocking error — the loop will continue. Resetting the compact guard here means compact fires again on the next iteration, hits the same failure, fires the stop hook again, and loops indefinitely.

**What breaks without it:** A single stop-hook injection after a failed compact causes: compact → still 413 → stop-hook blocking error → reset guard → compact → still 413 → … burning thousands of API calls per session.

## Anti-patterns

- **What:** Yield 413 / max-output-tokens errors directly from the stream loop as they arrive
  **Why it fails:** SDK consumers and UI layers treat any error as terminal and abort the session
  **Symptom:** Sessions terminate at first context overflow even though compact would have recovered them; users see "session ended" with no retry

- **What:** Run post-turn hooks unconditionally (including when last message is an API error)
  **Why it fails:** Hook injection increases context size, which re-triggers the same error, which fires the hook again
  **Symptom:** Session enters infinite retry loop consuming API quota; error count monotonically increases; loop never exits

- **What:** Reset the "compact already attempted" flag when a stop-hook injects a blocking error
  **Why it fails:** The blocking error causes the loop to continue; compact fires again on the still-oversized context; loop repeats
  **Symptom:** Thousands of compact API calls per session, all returning the same prompt-too-long failure

- **What:** Run expensive recovery (compact) before cheap recovery (collapse drain)
  **Why it fails:** Compact is a full summarization API call; collapse drain is free and preserves granular context
  **Symptom:** Unnecessary compaction events; loss of detailed context history that collapse would have preserved

## Structural template

```
// Per-turn loop
while (true) {
  // --- Pre-API: apply recovery steps in cost order ---
  messages = await applyToolResultBudget(messages)      // evict oversized results first
  messages = await snip(messages)                       // remove oldest rounds
  messages = await microcompact(messages)               // cheap inline compression
  messages = await collapseIfNeeded(messages)           // staged collapses (no API)
  messages = await proactiveCompact(messages)           // full summary (API call)

  // --- Stream from model ---
  let withheldError = null
  for await (const message of callModel(messages)) {
    if (isRecoverableError(message)) {
      withheldError = message  // buffer — do NOT yield
    } else {
      yield message
    }
  }

  // --- Post-stream: cascade recovery for withheld errors ---
  if (withheldError) {
    if (isPromptTooLong(withheldError)) {
      // 1. Try collapse drain (free)
      if (!alreadyTriedCollapseDrain) {
        const drained = collapseContext(messages)
        if (drained.committed > 0) { messages = drained.messages; continue }
      }
      // 2. Try reactive compact (API call, one-shot)
      if (!hasAttemptedCompact) {
        const compacted = await reactiveCompact(messages)
        if (compacted) {
          hasAttemptedCompact = true  // set sticky — never reset
          messages = compacted; continue
        }
      }
      // 3. Surface — all recovery exhausted
      yield withheldError
      return { reason: 'prompt_too_long' }
    }
    // ... similar cascade for max_output_tokens
  }

  // --- Post-turn hooks: ONLY run when model produced a valid response ---
  if (!withheldError && !lastMessage.isApiError) {
    const hookResult = await runStopHooks(messages)
    if (hookResult.blockingErrors.length > 0) {
      // Continue loop — but do NOT reset hasAttemptedCompact
      messages = [...messages, ...hookResult.blockingErrors]
      continue
    }
  }

  if (!needsFollowUp) return { reason: 'completed' }
  messages = [...messages, ...toolResults]
}
```

## Underlying principle
The common thread is **error ownership**: the agentic loop owns context errors internally and is responsible for recovery before surfacing them. Downstream consumers (SDKs, UIs, hooks) are not in a position to recover from context-size errors — they don't have access to the message history or the compact machinery. Surfacing errors prematurely delegates a problem the loop can solve to a system that cannot. The same principle applies to hook injection: hooks are observers of successful turns, not participants in error recovery.

## Status
`seed-1-team` — promoted from cross-subsystem design analysis. Validate earlier than standard: needs a second team confirmation before treating as established.
