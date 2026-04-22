---
name: structured-output-across-providers
description: "Gets reliable structured output from LLM calls when supporting multiple providers with inconsistent tool-calling support, preventing silent data corruption from malformed responses. TRIGGER when: building an agent that must work across multiple LLM providers, parsing JSON from LLM text responses without native tool calling, LLM classification or selection steps produce unparseable output, adding fallback for structured output extraction"
---

## Key Decisions

1. Use JSON string parsing with repair (rather than native function calling / structured outputs) when you need to support 10+ LLM providers, many of which lack tool-calling support. Native tool calling gives schema validation but locks you to providers that support it; JSON-in-text works universally but produces silent data corruption when the repair library "fixes" malformed JSON into structurally valid but semantically wrong objects (e.g., a missing closing bracket causes two list items to merge into one). Mitigate by validating the repaired output against the expected schema before using it.

2. For LLM-based classification or selection steps (e.g., selecting an agent persona, choosing tools from a list), always implement a deterministic fallback that activates when JSON parsing fails — not just when the LLM call fails. The LLM may return valid JSON that doesn't match the expected structure, or the repair may produce unexpected keys. A two-stage approach (LLM selection → pattern-based or default fallback) ensures the pipeline never blocks on a parsing failure in a non-critical selection step.

3. When using LLM classification to dynamically select a persona, role, or configuration for downstream steps, inject the selected result into all subsequent LLM calls as part of the system prompt — not as a one-time context note. Without persistent injection, later pipeline steps revert to generic behavior because each step constructs messages from scratch. The classification cost is amortized across all downstream calls only if its output is consistently applied.

## Anti-patterns

- **What**: Trusting JSON repair output without schema validation, treating "parseable" as "correct."
- **Why**: JSON repair libraries prioritize producing valid JSON over preserving semantic intent — a missing comma between array elements can cause two items to merge, a truncated response can repair to a valid but incomplete structure, and extra whitespace in string values can repair to empty strings.
- **Symptom**: Pipeline produces plausible but wrong results intermittently (e.g., 3 sub-queries instead of 5, agent persona with wrong role text); the error rate correlates with model temperature and output length, and is only caught by manually inspecting LLM outputs in logs.

- **What**: Single-layer fallback that catches LLM call failure but not output format failure, leaving a gap between "call succeeded" and "output is usable."
- **Why**: The LLM returns HTTP 200 with valid JSON that doesn't match the expected schema (wrong keys, wrong types, unexpected nesting), and the code proceeds to use the malformed structure because the error handling only catches exceptions from the API call itself.
- **Symptom**: Downstream pipeline steps crash with KeyError or TypeError on attributes that "should always exist" — the error appears to be in the consuming code rather than in the LLM output, making it hard to trace back to the parsing step.

## Structural Template

```
function extract_structured_output(llm_response, expected_schema, fallback_fn):
  # Stage 1: Parse and repair
  parsed = json_repair(llm_response.text)

  # Stage 2: Schema validation (the critical step most pipelines skip)
  if validates_against(parsed, expected_schema):
    return parsed

  # Stage 3: Deterministic fallback
  log_warning("LLM output parsed but failed schema validation", parsed)
  return fallback_fn(llm_response.text)

function classify_and_inject(query, classification_prompt, downstream_steps):
  # LLM-based classification with fallback
  try:
    raw_response = llm_call(classification_prompt, query)
    classification = extract_structured_output(
      raw_response,
      schema={name: string, role_prompt: string},
      fallback_fn=default_classification
    )
  except LLMCallFailed:
    classification = default_classification()

  # Inject into ALL downstream steps, not just the next one
  for step in downstream_steps:
    step.system_prompt = combine(classification.role_prompt, step.base_prompt)

function select_from_options(query, options, selection_prompt):
  # Two-stage selection: LLM → pattern-based fallback
  try:
    raw = llm_call(selection_prompt, query, options)
    selected = extract_structured_output(
      raw,
      schema={selected_items: list[string]},
      fallback_fn=lambda _: pattern_match_select(query, options)
    )
    return selected.selected_items
  except LLMCallFailed:
    return pattern_match_select(query, options)

function pattern_match_select(query, options):
  # Deterministic fallback: keyword matching against option descriptions
  scores = [(option, keyword_overlap(query, option.description)) for option in options]
  return top_k(scores, k=3)
```
