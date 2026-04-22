---
name: supervisor-subagent-delegation
description: "Structures how a supervisor agent delegates, constrains, and communicates with parallel sub-agents to prevent redundant work and coordination failures. TRIGGER when: building a supervisor that dynamically delegates to sub-agents, sub-agents produce redundant or overlapping work, supervisor fires all tasks without assessing intermediate results, sub-agents lack context to work independently, agent over-parallelizes simple queries or under-parallelizes complex ones"
---

## Key Decisions

1. Force a deliberate reasoning pause in the supervisor between delegation rounds using an explicit thinking step. Without this, the supervisor fires off all sub-agent tasks at once without assessing intermediate results, leading to redundant and unfocused work across sub-agents.

2. Make delegation visible as tool calls in the supervisor's reasoning trace rather than using direct function invocations. This allows the supervisor to include detailed, structured instructions in the tool call arguments, producing better-scoped delegation. Direct invocations hide delegation intent from the reasoning chain.

3. When sub-agent requests exceed the concurrency limit, return an error message to the supervisor explaining the constraint rather than silently dropping or queuing excess requests. This teaches the model about its operating constraints through in-context feedback, allowing it to self-correct on the next round.

4. Include explicit scaling rules in the supervisor prompt that map query complexity to parallelism levels (e.g., simple fact-finding gets 1 sub-agent, comparative analysis gets 1 per element). Without these rules, the model over-parallelizes trivial queries (wasting resources and producing thin results) and under-parallelizes complex ones (producing incomplete coverage).

5. Give each sub-agent complete, standalone instructions — they cannot see other sub-agents' work or the supervisor's full context. The supervisor prompt must explicitly state this constraint so it doesn't assume implicit context sharing between sub-agents. When this is missing, the supervisor writes instructions like "also check what the other agent found," which the sub-agent cannot fulfill.

## Anti-patterns

- **What**: Supervisor delegates all research in a single burst without intermediate assessment.
  **Why**: Without a forced reasoning step, the model optimizes for perceived efficiency by parallelizing everything upfront, but it hasn't yet seen results that would refine subsequent queries.
  **Symptom**: Sub-agents return overlapping findings on the same subtopic while other subtopics go unresearched; final output has deep coverage in one area and gaps elsewhere.

- **What**: Sub-agent instructions reference context only available to the supervisor (e.g., "expand on the previous finding" or "check what other agents found").
  **Why**: The supervisor assumes sub-agents share its conversation history or can see sibling agents' outputs, because the prompt doesn't explicitly state the isolation boundary.
  **Symptom**: Sub-agents produce off-topic or confused results; debugging requires reading the full delegation trace to discover the context mismatch.

- **What**: Excess sub-agent requests are silently dropped when concurrency limits are exceeded.
  **Why**: The system enforces a limit but doesn't communicate it back to the supervisor model, so the model continues planning as if all delegations succeeded.
  **Symptom**: Final report has unexplained gaps in coverage; the supervisor's plan references research that was never executed but no error appears in logs.

## Structural Template

```
function supervisor_loop(query, config):
    state = initialize(query)

    while not state.is_complete:
        // Force deliberate reasoning before each delegation round
        reasoning = think_step(state.findings_so_far, state.remaining_gaps)

        // Determine parallelism from query complexity
        num_agents = apply_scaling_rules(reasoning.assessed_complexity, config.max_concurrent)

        // Delegate via structured tool calls with standalone instructions
        delegations = []
        for task in reasoning.next_tasks[:num_agents]:
            delegations.append(
                create_delegation(
                    instructions=build_standalone_instructions(task, state.context),
                    constraint="You cannot see other agents' work."
                )
            )

        // Execute with overflow feedback
        results, overflow_errors = execute_concurrent(delegations, config.max_concurrent)

        // Feed overflow errors back to supervisor as messages
        if overflow_errors:
            state.add_feedback("Reduce parallelism: {overflow_errors.count} tasks exceeded limit")

        state.incorporate(results)

    return state.final_output
```
