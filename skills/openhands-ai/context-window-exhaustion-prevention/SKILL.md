---
name: context-window-exhaustion-prevention
description: "Prevents context window exhaustion in multi-turn agents that use tool calling, addressing batched tool dispatch and agent-initiated condensation. TRIGGER when: building a multi-turn agent with tool calling, conversation exceeds context limits, agent repeats actions or loses context, tool outputs inflate history"
---

## Key Decisions

1. **Queue multi-tool responses and dispatch one at a time, accepting intermediate result blindness.** When the LLM returns multiple tool calls in a single response, queue them and execute sequentially on subsequent loop iterations rather than re-calling the LLM per tool. This preserves the LLM's intended call sequence and avoids redundant LLM calls, but the agent cannot observe intermediate tool results between batched calls — a failed early call won't prevent later calls from executing with stale assumptions.

2. **Give the agent an explicit tool to request its own context condensation.** Rather than relying solely on controller-level context size checks, expose a tool the agent can call to trigger condensation proactively. If this tool is disabled or misconfigured, the agent has no self-initiated escape hatch from context overflow — it must wait for an external error-triggered condensation, which may arrive too late (after a failed LLM call).

## Anti-patterns

- **What**: Dispatching all batched tool calls without observing intermediate results.
  **Why**: The LLM planned the calls based on assumptions that may be invalidated by the first call's result; executing the full batch blindly leads to cascading errors on stale state.
  **Symptom**: Agent executes a file edit followed by a test run in a single batch, but the edit failed silently — the test runs against the old code, passes, and the agent reports success on broken code.

- **What**: Relying only on controller-side context size detection to trigger condensation.
  **Why**: The controller detects overflow after the LLM call fails (context exceeded), not before — by the time the error fires, the agent has already wasted a call and may lose the most recent reasoning.
  **Symptom**: Intermittent "context length exceeded" errors mid-conversation that lose the agent's last reasoning step, causing it to repeat or contradict its prior plan.

- **What**: Using broad trigger keywords for knowledge injection without relevance scoring.
  **Why**: A single ambiguous keyword (e.g., "test", "deploy") matches too many documents, injecting hundreds of tokens of irrelevant context that displaces useful history.
  **Symptom**: Agent performance degrades on common-word topics despite having correct knowledge files — the knowledge is present but buried in noise, and the consumed tokens crowd out actual conversation history.

## Structural Template

```
// Agent loop with context-aware tool dispatch and self-managed condensation

function agent_loop(conversation, tools, knowledge_base):
    pending_queue = []

    while not done:
        if pending_queue is not empty:
            next_action = pending_queue.dequeue()
            result = execute_tool(next_action)
            // DECISION POINT: check result before continuing queue
            if result.indicates_failure:
                pending_queue.clear()  // abort remaining batched calls
            conversation.append(observation(result))
        else:
            // Inject knowledge before LLM call
            triggered_knowledge = keyword_match(conversation.last_user_message, knowledge_base)
            triggered_knowledge = rank_and_filter(triggered_knowledge, max_tokens=BUDGET)
            context = build_context(conversation, triggered_knowledge)

            // Check context budget BEFORE calling LLM
            if token_count(context) > CONDENSATION_THRESHOLD:
                context = condense(context)

            response = call_llm(context, tools + [condensation_request_tool])

            if response.tool_calls.length > 1:
                pending_queue.enqueue_all(response.tool_calls[1:])

            if response.requests_condensation:
                conversation = condense(conversation)
            else:
                execute_first_tool_or_respond(response)
```
