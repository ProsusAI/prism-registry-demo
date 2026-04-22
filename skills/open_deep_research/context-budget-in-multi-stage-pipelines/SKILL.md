---
name: context-budget-in-multi-stage-pipelines
description: "Manages token budgets across multi-stage agent pipelines to prevent context degradation and token limit failures at each stage. TRIGGER when: building a multi-stage pipeline where outputs from one stage feed the next, agent hits token limits during later pipeline stages, planning quality degrades as accumulated context grows, raw tool outputs inflate context beyond model capacity"
---

## Key Decisions

1. Compress each sub-stage's output with a separate summarization call before returning it to the orchestrating stage. Without compression, the orchestrator's context fills with raw search results and tool outputs, degrading its planning quality and eventually hitting token limits. This is the difference between supporting 2 concurrent sub-stages and 5+.

2. Use a cheaper, smaller model to summarize raw content (e.g., search results, document extracts) into structured summaries before passing them to the expensive reasoning model. This two-model architecture reduces token consumption on the expensive model by an order of magnitude while preserving key information. The risk is that the summarizer drops nuanced details it deems unimportant — mitigate by extracting key excerpts alongside the summary.

3. Implement different token-overflow strategies for different pipeline phases based on what each phase needs. Intermediate stages that build on recent context should drop older messages (preserving recency). Final synthesis stages that need breadth across all findings should truncate individual findings progressively (e.g., 10% reduction per retry) rather than dropping entire findings. A single overflow strategy applied uniformly degrades at least one phase.

## Anti-patterns

- **What**: A catch-all exception handler treats every error as a token limit error, silently terminating the pipeline stage.
  **Why**: Token limit errors are structurally similar to other API errors in many SDKs, and a broad exception catch (or a debug flag left as always-true) routes all failures into the token-overflow recovery path.
  **Symptom**: Transient API errors (rate limits, network timeouts, malformed responses) silently terminate research with partial results and no error indication; debugging requires noticing that the "token limit" recovery path was taken when token counts were well within limits.

- **What**: Raw tool outputs passed directly to the orchestrator without compression.
  **Why**: Developers skip the compression step during prototyping ("the model has 128k context, it'll be fine") and never add it back because it works with 1-2 sub-stages.
  **Symptom**: Pipeline works in testing with 2 sub-stages but fails in production with 4+; the orchestrator's final planning step produces shallow or repetitive analysis because its context is dominated by raw search result text rather than distilled findings.

- **What**: Same overflow strategy applied to all pipeline stages.
  **Why**: Developers implement one token-overflow handler and reuse it everywhere for consistency.
  **Symptom**: Either intermediate stages lose recent context (because the handler preserves breadth) causing the agent to repeat already-completed work, or the final synthesis stage drops entire research threads (because the handler preserves recency) producing a report with unexplained coverage gaps.

## Structural Template

```
function pipeline_stage(inputs, stage_config):
    try:
        result = call_model(inputs, stage_config.model)
        return result
    catch token_limit_error:
        return apply_overflow_strategy(inputs, stage_config.overflow_mode)
    catch other_error:
        // NEVER conflate with token limits — propagate or retry separately
        raise or retry(other_error)

function compress_stage_output(raw_output, summary_model):
    // Use cheaper model to produce structured summary
    return summary_model.generate(
        schema={summary: string, key_excerpts: list[string]},
        input=raw_output
    )

function orchestrator_loop(query):
    findings = []

    for round in research_rounds:
        raw_results = execute_substages(round.tasks)

        // Compress before accumulating into orchestrator context
        for result in raw_results:
            findings.append(compress_stage_output(result, cheap_model))

    // Final synthesis uses breadth-preserving overflow
    return synthesize(
        findings,
        overflow_mode="truncate_each_progressively"
    )

// Overflow strategies per phase:
// - intermediate: drop_oldest_messages (preserve recency)
// - synthesis: truncate_findings_progressively (preserve breadth)
```
