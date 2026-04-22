---
name: agentic-loop-invariant-side-calls
description: "Prevent redundant API calls from side operations whose inputs don't change across loop iterations in a multi-step agentic tool-use loop. TRIGGER when: adding a background API call or prefetch to an agentic tool-use loop, designing context enrichment for a multi-step agent, adding memory retrieval to a streaming tool pipeline."
---

# Agentic Loop Invariant Side Calls
*Fire side calls whose inputs are invariant across loop iterations once per turn entry — not once per iteration — to avoid N redundant API calls in multi-tool turns.*

## Key decisions

1. Identify side calls (prefetch, background enrichment, memory retrieval) whose inputs are derived solely from the user's message and fire them once before the loop starts, not at the top of each iteration. Without this, a 5-tool turn fires the same enrichment call 5 times with identical inputs, each call returning the same result — the cost is invisible in single-tool tests where the loop runs exactly once.

2. Store the side call's result outside the loop and consume it after the tool-use phase completes, alongside the main response. Without this, the prefetch result is consumed at the start of the first iteration and discarded before tool results arrive, meaning the enrichment never informs the model response it was intended to support.

3. Use a lazy-start pattern (fire the side call concurrently with the first model request) rather than awaiting it before starting the loop. Without this, a slow enrichment call (e.g., 1–2s) adds latency to every turn even when the result is rarely used.

## Anti-patterns

- **What**: Call the enrichment function at the top of the tool-use loop so each iteration has the freshest data
- **Why**: Freshness is the natural instinct — iterating in a loop, each pass feels like a new context that deserves a new fetch
- **Symptom**: N identical API calls per multi-tool turn (N = number of tool calls); only visible in telemetry or billing; single-tool test runs hide this because the loop executes once

- **What**: Await the side call result before starting the first model API call
- **Why**: Sequential initialization feels safe — ensures the enrichment is ready before it's needed
- **Symptom**: Every turn incurs the full enrichment latency as a serial dependency, even when the enrichment result is rarely consumed; users experience consistent added latency

- **What**: Consume the side call result immediately at the start of the first iteration
- **Why**: It's available at that point and feels natural to use it right away
- **Symptom**: The enrichment context is injected before tool calls rather than after, meaning it informs the tool dispatch decision but not the final response synthesis — the wrong phase

## Structural template

```
async function processUserTurn(userMessage, state):
    
    # Fire invariant side calls BEFORE the loop — inputs are the user message, invariant across iterations
    using pendingEnrichment = startEnrichmentPrefetch(userMessage, state.context)
    # Runs concurrently with the first model API call — does NOT block loop start
    
    while true:
        # Model call (runs concurrently with pendingEnrichment on first iteration)
        response = await callModel(state.messages)
        
        if response.stopReason !== "tool_use":
            # Tool-use phase complete — NOW consume the enrichment result
            enrichment = await pendingEnrichment   # resolves under model latency (~1-30s)
            yield { response, enrichment }
            break
        
        # Execute tools and continue loop
        toolResults = await executeTools(response.toolUseBlocks)
        state.messages.push(response, toolResults)
        
        # DO NOT re-fire enrichment here — inputs haven't changed
        # DO NOT consume enrichment here — tool-use phase isn't done yet
```
