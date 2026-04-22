---
name: multi-tier-model-routing
description: "Routes different agent pipeline steps to different model tiers based on task complexity, preventing cost explosion on simple steps and quality degradation on complex ones. TRIGGER when: building an agent pipeline with multiple LLM call types, optimizing LLM costs in a multi-step agent, choosing between reasoning and chat models for different stages, adding fallback chains when a model tier fails"
---

## Key Decisions

1. Split pipeline LLM calls into at least three tiers — cheap/fast for classification and routing, mid-tier for structured generation, and reasoning-capable for semantic decomposition. Without tiering, you either overspend 10-50x on classification tasks or get shallow results on decomposition tasks that require multi-step reasoning.

2. Use reasoning-effort-aware models (models with explicit reasoning parameters) for query decomposition and planning steps, not standard chat models. Chat models produce keyword-level decompositions (surface reformulations of the original query), while reasoning models produce semantically distinct research angles that cover different facets of the problem.

3. Build a cross-tier fallback chain for critical pipeline steps: reasoning model → reasoning model with constrained output → mid-tier chat model. Reasoning models have higher refusal rates and format compliance failures than chat models; without a tier-downgrade fallback, a single model refusal halts the entire pipeline despite cheaper models being capable of producing acceptable (if lower quality) output.

4. Implement a message format fallback for the generation step: system+user message → single combined user message. Different model providers and local models handle system messages inconsistently — some ignore them, some merge them poorly. The fallback ensures generation works across all providers without provider-specific branching logic.

## Anti-patterns

- **What**: Cost tracking uses a single provider's pricing table hardcoded at build time, applied to all model tiers regardless of actual provider.
- **Why**: When different tiers use different providers (e.g., cheap tier on a local model, reasoning tier on a cloud API), cost estimates silently diverge from reality because the pricing formula doesn't match the model being called.
- **Symptom**: Budget dashboards show plausible numbers that don't match actual invoices; the discrepancy grows as traffic scales and is only caught during monthly billing reconciliation.

## Structural Template

```
config:
  tiers:
    fast:    {model, temperature, max_tokens, use_for: [classification, routing, simple_extraction]}
    smart:   {model, temperature, max_tokens, use_for: [generation, synthesis, writing]}
    strategic: {model, reasoning_effort, max_tokens, use_for: [decomposition, planning, evaluation]}

function call_with_tier(task_type, messages):
  tier = resolve_tier(task_type, config.tiers)

  # Cross-tier fallback chain for critical steps
  fallback_chain = [tier, tier_with_constrained_output, next_lower_tier]

  for tier_config in fallback_chain:
    response = try_call(tier_config, messages)
    if response.success:
      track_cost(tier_config.model, response.tokens)  # provider-aware pricing
      return response
    if not is_retryable(response.error):
      raise

  raise AllTiersFailed

function try_call(tier_config, messages):
  # Message format fallback for provider compatibility
  try:
    return llm_call(tier_config, messages_as_system_plus_user)
  except SystemMessageUnsupported:
    return llm_call(tier_config, messages_as_single_user)
```
