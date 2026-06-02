---
name: context-window-overflow-survival
description: "Agent conversations that span many turns overflow the context window, causing silent truncation, lost task state, or infinite retry loops. TRIGGER when: designing a long-running agent, adding context compression, debugging an agent that appears to restart mid-task or retries indefinitely, or observing model amnesia on extended conversations."
---

## Key Decisions

1. Protect a token-budget-sized tail of recent messages before compressing. Compress only the middle of the conversation, not the most recent turns. Without tail protection, the model loses detail on recent context exactly when it most needs continuity for task completion.

2. Strip verbose tool results before the LLM summarization call. Tool outputs (search results, file dumps) consume summarizer token budget without improving summary quality. Removing them first as a cheap pre-pass means the summarizer focuses entirely on conversational content.

3. On subsequent compressions, update the existing summary rather than discarding and replacing it. Replacing summaries discards information already captured from earlier segments. An iterative chain preserves the full history at increasing levels of abstraction.

4. Frame every compression summary as "handoff reference only — do not re-execute tasks." Without this framing, models re-execute completed tasks described in summaries ("I see you asked me to install X — let me do that now"). The framing prevents this explicitly.

5. Classify connection-reset errors with no HTTP status code as context overflow when token count approaches the context limit. Providers drop connections on oversized requests without returning a structured error. Without a token-count heuristic to distinguish this from genuine network failures, the agent retries indefinitely without compressing.

## Anti-patterns

- **What**: Compressing from the beginning of the conversation, including recent turns.
  **Why**: Recent turns have the highest information density for task completion — the model needs full detail on what just happened.
  **Symptom**: Agent appears confused about decisions made 2–3 turns ago, asks for clarification the user already provided.

- **What**: Discarding the previous summary and writing a fresh one on each compression round.
  **Why**: Each compression pass destroys history captured by the prior pass; information from early turns is permanently lost after two rounds.
  **Symptom**: Agent loses track of multi-step tasks after the third or fourth compression, reverting to initial assumptions.

- **What**: Passing raw tool outputs to the summarizer without stripping them first.
  **Why**: Tool outputs can be orders of magnitude larger than conversational context and contain no summarization-relevant signal.
  **Symptom**: Summarizer hits its own context limit on tasks that called file or search tools heavily; summary omits the actual conversation.

## Structural Template

```
function compress_context(messages, token_budget):
    tail_tokens = min(20_000, token_budget * 0.3)
    tail        = preserve_tail(messages, tail_tokens)
    middle      = messages[1 : -len(tail)]            # exclude system prompt + tail

    pruned_middle    = replace_tool_results_with_placeholder(middle)
    existing_summary = get_current_summary(messages)

    new_summary = llm_summarize(
        prefix   = "HANDOFF REFERENCE ONLY — do NOT re-execute tasks mentioned below.",
        prior    = existing_summary,
        content  = pruned_middle
    )

    return [system_prompt, summary_message(new_summary)] + tail


function classify_disconnect(error, approx_tokens, context_length):
    if error is NetworkReset and error.status_code is None:
        if approx_tokens > context_length * 0.6 or approx_tokens > 120_000:
            return CONTEXT_OVERFLOW   # → compress, then retry
    return TRANSIENT_NETWORK_ERROR    # → backoff retry without compression
```
