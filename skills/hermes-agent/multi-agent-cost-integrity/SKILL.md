---
name: multi-agent-cost-integrity
description: "Multi-agent systems bleed iteration budgets on programmatic tool calls and create unbounded cost loops through recursive delegation. TRIGGER when: implementing a delegation or subagent system, adding code-execution or batch-processing tools to an agent, or debugging agents that exhaust their iteration budget faster than expected."
---

## Key Decisions

1. Programmatic tool calls made inside a code-execution environment — where the agent script chains tool calls in a loop — should not consume from the interactive iteration budget. Without budget refunds for these programmatic turns, a script that chains 30 tool calls consumes 30 turns from a fixed budget, silently reducing the agent's capacity for interactive work.

2. Exclude the delegation tool from the tool set passed to subagents. Allowing subagents to delegate further creates an unbounded agent tree with no depth limit. Iteration budget tracking becomes unreliable because recursive branches run against the same counter without a consistent accounting point, and cost overruns are only visible after the fact.

## Anti-patterns

- **What**: Counting all tool calls — including those fired inside programmatic loops — against the interactive iteration budget.
  **Why**: Code-execution tasks amplify turn counts in a way the budget model doesn't account for; a single step can consume tens of turns.
  **Symptom**: Tasks that use code execution heavily hit the iteration limit well before interactive work is done; the budget appears exhausted after one code-heavy step.

- **What**: Passing the delegation tool to subagents unchanged.
  **Why**: There is no depth limit; a subagent that delegates creates child agents that may themselves delegate, growing the tree exponentially.
  **Symptom**: A single task triggers cascading agent spawns; total LLM calls grow non-linearly; cost overruns are only discoverable in billing after the task completes.

## Structural Template

```
class IterationBudget:
    def charge(self, source: Literal["interactive", "programmatic"]) -> None:
        if source == "programmatic":
            return                    # refund — programmatic loops don't consume budget
        self.remaining -= 1
        if self.remaining <= 0:
            raise BudgetExhausted()


def get_subagent_tool_set(parent_tools: list[Tool]) -> list[Tool]:
    DELEGATION_TOOLS = {"delegate_task", "spawn_agent", "create_subagent"}
    return [t for t in parent_tools if t.name not in DELEGATION_TOOLS]
    # Delegation is strictly depth-1; subagents execute, they do not orchestrate
```
