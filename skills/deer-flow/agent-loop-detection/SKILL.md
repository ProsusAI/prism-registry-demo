---
name: agent-loop-detection
description: "Detects and breaks agent execution cycles when the model repeatedly issues identical tool calls without making progress. TRIGGER when: agents repeat the same tool calls across turns, a model gets stuck in a self-reinforcing pattern, infinite agent loops exhaust quota, a hard termination signal is needed without crashing the run or losing conversation state."
---

## Key Decisions

1. Use two detection thresholds — a warn threshold and a hard-stop threshold — rather than a single termination point. The warn threshold injects a signal that a loop may be forming, giving the model a chance to self-correct before the hard intervention. A single-threshold design terminates without attempting recovery; a two-threshold design separates the self-correction opportunity from the hard termination.

2. At the hard-stop threshold, strip tool_calls from the last AIMessage rather than raising an exception. Stripping forces the model to produce a text-only response on the next turn by removing the pending tool invocations from message history — it is a structural change to the message, not an error. Raising an exception propagates up the agent framework's call stack, potentially skipping checkpointing and losing conversation state built during the loop.

3. Use deterministic hash comparison on the tool call list, not semantic similarity. Hash comparison is O(1), requires no additional model call, and produces zero false positives. The accepted trade-off is that semantically equivalent but syntactically different calls are not detected — this is acceptable since the goal is stopping exact-repeat cycles, not all forms of unproductive behavior.

## Anti-patterns

- **What**: Relying on a prompt instruction ("don't repeat tool calls") as the only loop prevention. **Why**: Instructions compete with the task objective mid-run; under context pressure and deep into a task, the model is already inside the pattern it was told to avoid. **Symptom**: Loops run to quota exhaustion or timeout; costs appear as anomalously high-token runs with no signal that a loop was the cause.

- **What**: Raising an exception to terminate the loop rather than modifying message state. **Why**: Exception propagation through agent framework event loops may skip the final checkpoint write, leaving the thread with no terminal state. **Symptom**: On hard-stop, the thread shows no final user-visible result and the run cannot be resumed or inspected from the last valid checkpoint.

- **What**: Applying a single hard-stop threshold with no prior warning to the model. **Why**: The model has no opportunity to recognize and exit the loop itself, which it can do successfully in most cases once the pattern is surfaced. **Symptom**: Runs terminate abruptly on the first opportunity that could have self-resolved; user experience degrades for recoverable loops that a warning would have broken.

## Structural Template

```
loop_state:
    call_hash_counts: dict[hash, int]
    warn_threshold: 3
    stop_threshold: 5

function on_ai_message(message):
    if not message.tool_calls:
        return message, CONTINUE
    
    call_hash = hash(canonical_form(message.tool_calls))
    loop_state.call_hash_counts[call_hash] += 1
    count = loop_state.call_hash_counts[call_hash]
    
    if count >= stop_threshold:
        message.tool_calls = []  # strip — force text response next turn
        return message, HARD_STOP
    
    if count >= warn_threshold:
        # Inject a warning into the message history before model call
        inject_system_message("Repeated tool call detected. Reconsider your approach.")
        return message, WARN
    
    return message, CONTINUE

# canonical_form: normalize tool call list for stable hashing
# (sort by tool name + args, strip non-deterministic fields like call_id)
```
