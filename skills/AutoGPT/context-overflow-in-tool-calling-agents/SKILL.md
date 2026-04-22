---
name: context-overflow-in-tool-calling-agents
description: "Prevents context window overflow in multi-turn tool-calling agents by combining transcript persistence, split compression strategies, and tool output offloading. TRIGGER when: building a multi-turn agent with tool calling, conversation history grows across turns and approaches context limits, tool outputs are large or unpredictable in size, agent repeats tool calls or loses context after long sessions, designing prompt compression strategy for cost-sensitive execution paths."
---

## Key Decisions

1. **Persist full conversation transcripts as serialized files rather than using LLM summarization for multi-turn continuity.** Strip non-essential entries (progress updates, internal status) before persisting, then upload to durable storage and reload on the next turn. LLM summarization loses tool call details that the agent needs for coherent continuation; file-based persistence preserves full context at the cost of storage I/O rather than token budget.

2. **Use truncation-only compression targeting half the context window for cost-sensitive execution paths, and reserve LLM summarization for interactive paths only.** Truncation is free, fast, and deterministic — it never produces hallucinated summaries of prior context. Setting the target at context_window/2 leaves headroom for the current turn's input and output. The split strategy means batch/automated paths never incur extra LLM costs for compression while interactive paths get higher-quality context preservation.

3. **When a tool output exceeds a threshold set below the model's maximum (accounting for message wrapper overhead), persist the full output to external storage and replace it inline with a middle-out truncated preview pointing to the full content.** The threshold must be calculated relative to the model's tool-result limit minus envelope overhead (e.g., 80K when the limit is 100K). Middle-out truncation preserves both the beginning (headers, structure) and end (conclusions, final values) of the output, which are the most informative portions for the agent's next reasoning step.

## Anti-patterns

- **What**: No maximum conversation length enforcement — sessions grow unboundedly until they hit the context window hard limit.
  **Why**: Without proactive compression triggers, the agent silently loses early context when the provider truncates, leading to repeated tool calls or contradictory actions.
  **Symptom**: Agent re-executes a tool it already called 15 turns ago, or contradicts a decision it made earlier in the session, with no error or warning.

- **What**: Using LLM summarization for all compression paths including automated/batch execution.
  **Why**: Each compression call adds latency and cost proportional to context size, and summarization can hallucinate tool results (e.g., inventing a return value the tool never produced).
  **Symptom**: Batch execution costs spike unexpectedly; occasionally an agent acts on a tool result that differs from what the tool actually returned.

- **What**: Setting the tool output truncation threshold at or near the model's stated limit without accounting for message envelope overhead.
  **Why**: The model's limit applies to the raw content, but the SDK/API wraps tool results in metadata (tool call IDs, role markers, JSON structure) that consume additional tokens.
  **Symptom**: Intermittent "tool result too large" API errors on outputs that appear to be under the limit, with the failure rate varying by output content type.

## Structural Template

```
// Conversation persistence layer
function persist_transcript(session, messages):
    stripped = remove_non_essential_entries(messages)  // progress, heartbeats, internal status
    serialized = serialize_to_file(stripped)
    upload_to_storage(session.storage_key, serialized)

function restore_transcript(session):
    serialized = download_from_storage(session.storage_key)
    return deserialize_messages(serialized)

// Compression strategy (split by execution path)
function compress_if_needed(messages, context_window, path_type):
    target = context_window / 2
    if token_count(messages) <= target:
        return messages
    if path_type == INTERACTIVE:
        return llm_summarize(messages, target)
    else:  // BATCH or AUTOMATED
        return truncate_oldest(messages, target)

// Tool output offloading
THRESHOLD = MODEL_TOOL_RESULT_LIMIT - ENVELOPE_OVERHEAD  // e.g., 100K - 20K = 80K

function process_tool_output(output, session):
    if length(output) <= THRESHOLD:
        return output
    storage_ref = persist_to_workspace(session, output)
    preview = middle_out_truncate(output, THRESHOLD)
    return preview + "\n[Full output: " + storage_ref + "]"

// Agent turn loop
function execute_turn(session, user_input):
    messages = restore_transcript(session)
    messages = compress_if_needed(messages, MODEL_CONTEXT_WINDOW, session.path_type)
    messages.append(user_input)

    while agent_wants_to_act(messages):
        action = call_model(messages)
        if action.is_tool_call:
            raw_output = execute_tool(action.tool, action.args)
            output = process_tool_output(raw_output, session)
            messages.append(tool_result(action.id, output))
        else:
            messages.append(action.response)

    persist_transcript(session, messages)
    return messages.last()
```
