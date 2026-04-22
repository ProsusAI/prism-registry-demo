---
name: llm-call-resilience-and-cost-control
description: "Handles silent LLM failures, model routing trade-offs, and budget enforcement when cost models are approximate. TRIGGER when: agent gets empty or refused responses from LLM, routing requests across multiple models by capability, tracking per-task LLM spend with budget ceilings, LLM silently returns empty completions at low temperature"
---

## Key Decisions

1. **Auto-escalate temperature on silent LLM refusal.** When a model returns an empty response at temperature 0 (a rare but real failure mode with some providers), retry the same request at temperature 1.0 before exhausting the retry budget. Without this, a silent refusal at deterministic temperature will fail identically on every retry attempt, wasting the entire retry budget and aborting the task — the failure is invisible because there's no error, just an empty response.

2. **Use deterministic binary signals for model routing, not semantic classifiers.** Route requests to a primary (expensive) model based on binary, zero-cost checks — presence of images in the message, or secondary model's context window exceeded — rather than an LLM classifier that evaluates task complexity. This eliminates routing latency and cost but cannot distinguish a trivial text lookup from a complex multi-step reasoning task — both go to the cheaper model unless they contain images or exceed context limits. Accept this limitation explicitly rather than adding a routing LLM call that costs more than the savings.

3. **Track budget in-process using the gateway's cost estimate, and treat it as an approximation with overshoot risk.** Accumulate per-call costs reported by the LLM gateway (e.g., completion cost functions) against a per-task budget ceiling. This is not a hard server-side cap — if the gateway's pricing model is stale or inaccurate for a new model, the agent can overshoot the budget before the check trips. Design the budget check to fire before each LLM call (not after), and include a margin buffer to absorb one additional call's cost.

## Anti-patterns

- **What**: Retrying empty LLM responses at the same temperature without escalation.
  **Why**: Some models deterministically return empty completions at temperature 0 for certain inputs; retrying with identical parameters will fail identically every time.
  **Symptom**: Agent exhausts all 5 retries in ~40 seconds, logs show "no response" on every attempt with identical request parameters, task aborts with no useful error — appears as a provider outage but only affects specific prompts.

- **What**: Using an LLM call to classify request complexity for routing between models.
  **Why**: The routing call itself costs tokens and adds latency; for high-volume agent systems, the routing overhead can exceed the savings from using a cheaper model on simple tasks.
  **Symptom**: P50 latency increases by 200-500ms per agent step; cost analysis shows routing calls consuming 5-15% of total LLM spend with negligible improvement in task completion rates.

- **What**: Checking budget after the LLM call completes instead of before.
  **Why**: A single expensive call (long output, expensive model) can overshoot the budget in one step; post-call checking only catches the overshoot after the money is spent.
  **Symptom**: Tasks consistently exceed budget by 20-40% of a single call's cost; budget alerts fire after the fact; finance reports show systematic over-budget tasks despite budget enforcement being "enabled."

## Structural Template

```
// LLM call wrapper with temperature escalation and pre-call budget checks

function resilient_llm_call(messages, config, budget_tracker):
    temperatures = [config.default_temperature, ESCALATION_TEMPERATURE]

    for temp in temperatures:
        for attempt in range(config.max_retries):
            // Pre-call budget check with margin
            estimated_cost = estimate_call_cost(messages, config.model)
            if budget_tracker.accumulated + estimated_cost + MARGIN > budget_tracker.ceiling:
                raise BudgetExhaustedError(budget_tracker.accumulated, budget_tracker.ceiling)

            response = call_provider(messages, temperature=temp, ...)
            cost = compute_actual_cost(response)
            budget_tracker.accumulate(cost)

            if response.content is not empty:
                return response

            if is_transient_error(response):
                backoff(attempt)
                continue
            else:
                break  // non-transient at this temp, escalate

    raise LLMNoResponseError("exhausted all temperatures and retries")

// Model router — zero-cost binary signal routing

function route_request(messages, primary_model, secondary_model):
    if messages_contain_images(messages):
        return primary_model
    if token_count(messages) > secondary_model.context_limit:
        return primary_model
    return secondary_model
    // NOTE: does NOT route by task complexity — accept this limitation
```
