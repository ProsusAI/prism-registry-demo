---
name: agent-resource-budgeting
description: "Dual-layer resource budget enforcement for tool-calling agents — context budget via observation-level truncation and monetary budget via per-instance plus global tracking. TRIGGER when: tool outputs risk context overflow, running multiple agents in parallel with a shared cost ceiling, agent aborts on context exceeded with no recovery, cost tracking crashes the agent for unknown models"
---

## Key Decisions

1. Truncate tool outputs at the observation template level (before insertion into the message list), not at the context window level. Use a head+tail strategy (e.g., first 5K chars + last 5K chars) so the LLM sees the beginning of output (command confirmation, headers) and the end (error messages, final results) while the middle is elided. Without this, a single verbose command (large file cat, dependency install log) consumes the entire context budget in one turn, and truncating at the context window level loses the most recent — and most relevant — messages.

2. Implement cost tracking at two layers: per-agent-instance limits (checked before each LLM call) and a global thread-safe aggregate tracker (checked after each call's cost is known). Individual per-agent limits prevent any single task from overspending, but in batch execution with dozens of parallel agents, the aggregate can exceed the intended budget before any single agent hits its limit. Without the global layer, running 100 agents with a $3 per-agent limit can silently spend $300 when you intended $50 total.

## Anti-patterns

- **What**: Relying solely on output truncation without any context window management for the full message history — never pruning, summarizing, or windowing old messages.
  **Why**: Output truncation prevents single-turn blowups, but the message list still grows monotonically. After enough turns, the accumulated history exceeds the context limit regardless of per-message truncation.
  **Symptom**: Agent works reliably on short tasks (5-10 steps) but aborts with an unrecoverable context-exceeded error on longer tasks (30+ steps), after spending significant money on LLM calls that are now lost because the abort is fatal.

- **What**: Cost tracking that throws a fatal error when the cost calculation fails (e.g., model not in the provider's pricing registry) instead of degrading gracefully.
  **Why**: New models, custom deployments, and local models are frequently missing from pricing registries. A fatal error on cost calculation crashes the entire agent run — including expensive partial progress — for a non-critical feature.
  **Symptom**: Agent crashes immediately on the first LLM call with an opaque error about cost calculation, but only when using a recently released or self-hosted model. Users must discover a non-obvious config flag to disable strict cost tracking before they can use the agent at all.

## Structural Template

```
// Layer 1: Observation-level truncation (per-turn budget)
format_observation(raw_output, max_chars) -> string {
  if length(raw_output) <= max_chars:
    return raw_output
  half = max_chars / 2
  return raw_output[:half]
    + "\n[elided " + (length(raw_output) - max_chars) + " chars]\n"
    + raw_output[-half:]
}

// Layer 2a: Per-instance cost limit (checked BEFORE each call)
Agent.before_query() {
  if cost_limit > 0 AND accumulated_cost >= cost_limit:
    raise LimitsExceeded  // recoverable — can prompt for new limit
}

// Layer 2b: Global aggregate cost limit (checked AFTER each call)
GlobalCostTracker {
  lock: mutex
  total_cost: float
  limit: float

  add(cost) {
    with lock:
      total_cost += cost
    if limit > 0 AND total_cost > limit:
      raise GlobalLimitExceeded  // hard stop across all agents
  }
}

// Cost calculation with graceful degradation
calculate_cost(response, model_name) -> float {
  try:
    cost = pricing_registry.lookup(model_name, response.usage)
    if cost <= 0: raise InvalidCost
    return cost
  catch:
    if cost_limits_are_configured:
      log.warning("Cost unknown for model, limits may not trigger")
    return 0.0  // degrade, don't crash
}

// Integration: both layers in the query path
Agent.query(messages) -> Message {
  before_query()                              // Layer 2a: per-instance check
  response = model.call(messages)
  cost = calculate_cost(response, model_name) // graceful degradation
  accumulated_cost += cost
  global_tracker.add(cost)                    // Layer 2b: global check
  return format_response(response)
}
```
