---
name: tool-output-blast-radius
description: "Large tool outputs silently overflow the context window or cause request failures misdiagnosed as prompt-length errors. TRIGGER when: building agents that call file, search, or code-execution tools; debugging context overflow on tool-heavy tasks; handling LLM API errors after tool-rich turns."
---

## Key Decisions

1. When a tool result exceeds a size threshold, write the full content to a temp file and inject a preview plus the file path in its place. Truncating destroys information silently; passing raw content overflows context. The agent can read the file if it needs the full content, so nothing is lost.

2. Build three independent size-catching layers: (1) a per-tool internal cap where each tool self-truncates; (2) a per-result persistence gate that spills results over a byte threshold to disk; (3) a per-turn aggregate budget that spills any overflow results when the turn total exceeds a limit. Each layer catches what the one above it misses — a tool author forgetting to cap, or multiple medium-sized results summing to overflow.

3. When the API signals the output cap is too large rather than the input too long, reduce the output cap for that call rather than compressing the input context. Conflating the two causes unnecessary context compression cycles that add latency and destroy information when the prompt itself is not the problem.

## Anti-patterns

- **What**: Truncating tool results at a fixed character limit before injecting them into context.
  **Why**: Silent truncation means the model acts on partial information without knowing it's incomplete.
  **Symptom**: Agent hallucinates the rest of a file, or misses an error in a log that appeared after the truncation cutoff.

- **What**: A single global size limit applied only at injection time with no layered defense.
  **Why**: Individual tool output caps are bypassed when a tool author omits self-truncation; aggregate overflow is invisible when many medium outputs accumulate in the same turn.
  **Symptom**: Context overflow occurs despite each individual result appearing within the per-tool limit.

- **What**: Treating all API-reported size errors as prompt-length errors and triggering context compression.
  **Why**: Some size errors indicate only that the requested output cap doesn't fit the remaining token budget; the input is fine.
  **Symptom**: Unnecessary compression cycles add latency to large-output requests; information is discarded when the input never needed to shrink.

## Structural Template

```
MAX_SINGLE_RESULT  = 50_000   # chars
MAX_TURN_AGGREGATE = 200_000  # chars

function inject_tool_result(tool_name, raw_output, turn_budget_used):
    # Layer 1: tool self-caps (tool's own responsibility, not shown here)

    # Layer 2: per-result persistence
    if len(raw_output) > MAX_SINGLE_RESULT:
        path = write_temp_file(tool_name, raw_output)
        return preview(raw_output, 500) + f"\n[Full output → {path}]"

    # Layer 3: per-turn aggregate budget
    if turn_budget_used + len(raw_output) > MAX_TURN_AGGREGATE:
        path = write_temp_file(tool_name, raw_output)
        return f"[Result spilled to disk — {path}]"

    return raw_output


function handle_api_size_error(error_body, current_output_cap):
    if "available_tokens" in error_body:
        # Output cap too large — shrink cap, not input
        return retry_with_output_cap(current_output_cap * 0.8)
    else:
        # Prompt too long — compress input context
        return compress_and_retry()
```
