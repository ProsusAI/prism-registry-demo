---
name: ai-extraction-failure-observable
description: "Distinguish LLM extraction failures from genuinely-empty results so callers can implement retry and alerting. TRIGGER when: designing error handling for an LLM extraction step, implementing retry logic for an AI-backed processing pipeline, adding monitoring to a pipeline that returns results from an LLM call."
---

# AI Extraction Failure Observability
*Prevent LLM extraction failures from being silently absorbed as empty results.*

## Key decisions

1. Return a typed result that distinguishes extraction failure from extraction empty. Without this, callers receive identical output for "nothing found" and "LLM failed" — retry logic, alerting, and monitoring cannot act on the distinction, and a systematic outage looks identical to a low-content period.

2. Log LLM extraction failures at ERROR level with a structured field for the failure type. Without this, production failures appear as empty results with no log trace at a level that triggers alerts; the only observable signal is a drop in stored-record volume.

3. Design the extraction interface to surface at least two states: `empty` and `error`. Without this, an LLM rate-limit error, a JSON parse failure, and a genuine no-content conversation all return `[]` — callers cannot selectively retry, skip, or alert on any one of them.

## Anti-patterns

- **What**: Wrapping the entire LLM extraction call in a bare `except Exception: return []`
- **Why**: An empty list is valid output for "this conversation had nothing memorable"; it is also returned on timeout, JSON parse error, and rate-limit — the same return value covers all cases
- **Symptom**: After an LLM outage, the system reports zero records stored with no error signals in logs or metrics; the service appears healthy while silently discarding all new content

## Structural template

```
# Extraction result: two failure modes, not one
class ExtractionResult:
    status: Literal["ok", "empty", "error"]
    facts: list[str]
    error: str | None = None

def extract(messages) -> ExtractionResult:
    try:
        raw = llm.call(extraction_prompt, messages)
        facts = parse_json(raw)
        if not facts:
            return ExtractionResult(status="empty", facts=[])
        return ExtractionResult(status="ok", facts=facts)
    except (LLMError, JSONDecodeError) as e:
        log.error("extraction_failed", reason=type(e).__name__, detail=str(e))
        metrics.increment("extraction.error")
        return ExtractionResult(status="error", facts=[], error=str(e))

# Caller decides retry vs skip — not forced to swallow
result = extract(messages)
if result.status == "error":
    schedule_retry(messages)       # or emit alert, increment error counter
elif result.status == "empty":
    pass                           # genuine no-content — no action needed
else:
    store_all(result.facts)
```
