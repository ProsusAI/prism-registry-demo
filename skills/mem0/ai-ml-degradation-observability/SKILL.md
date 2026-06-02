---
name: ai-ml-degradation-observability
description: "Make AI/ML dependency degradation (missing model, failed import, unavailable component) visible to operators without interrupting the caller. TRIGGER when: designing fallback behavior for an optional AI/ML dependency, adding a new AI/ML step with graceful degradation, implementing health checks for a system with optional AI components."
---

# AI/ML Degradation Observability
*Graceful degradation is correct — invisible degradation is an operational blind spot.*

## Key decisions

1. Log every AI/ML dependency degradation at WARNING or ERROR level, not DEBUG. Without this, a production system running without entity extraction, BM25, or a reranker looks fully healthy in logs and dashboards; the only signal is degraded output quality, which is hard to attribute.

2. Expose a health endpoint or metrics counter that shows which AI/ML signals are currently active. Without this, operators cannot distinguish a healthy deployment from one where optional components are unavailable; A/B experiments cannot detect whether their test group received full or degraded results.

3. When partial results are returned due to degradation, include a signal in the response metadata — a `signals_active` field, a response header, or a structured log line. Without this, callers running quality evaluations cannot control for or detect degraded modes.

## Anti-patterns

- **What**: Catching `ImportError` or `ModelNotFoundError` and returning `[]` or the original input unchanged with only a DEBUG-level log
- **Why**: The fallback is indistinguishable from the normal path at every level — caller result, log severity, metric label
- **Symptom**: A package update silently removes an NLP model; the AI/ML step disables itself; result quality degrades across all users for days before anyone connects the quality drop to the missing component

## Structural template

```
# At each optional AI/ML integration boundary
def run_with_degradation_tracking(component_name, fn, fallback, *args):
    try:
        result = fn(*args)
        metrics.increment(f"{component_name}.active")
        return result, True   # (result, signal_active)
    except (ImportError, ModelNotFoundError, ComponentUnavailableError) as e:
        log.warning(f"{component_name}.unavailable", reason=str(e))
        metrics.increment(f"{component_name}.degraded")
        return fallback, False

# Health endpoint: expose active signals
def health():
    return {
        "status": "degraded" if _any_signal_missing() else "healthy",
        "signals_active": {
            "entity_extraction": entity_extractor.is_available(),
            "keyword_search":    bm25.is_available(),
            "reranker":          reranker.is_available(),
        }
    }

# At the search call site: expose degradation in response
entities, entity_ok  = run_with_degradation_tracking("entity_extraction", extract, [])
bm25_scores, bm25_ok = run_with_degradation_tracking("bm25", keyword_search, {})
result.metadata["signals_active"] = {
    "entity": entity_ok, "bm25": bm25_ok
}
```
