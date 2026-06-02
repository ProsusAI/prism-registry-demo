---
name: hybrid-search-threshold-calibration
description: "In hybrid search systems (semantic + keyword + signal boost), similarity thresholds must be calibrated to the combined score distribution — standard semantic defaults cause recall collapse. TRIGGER when: configuring a similarity threshold for hybrid search results, tuning recall vs precision in a system that combines semantic and keyword signals, migrating from pure-semantic to hybrid retrieval."
---

# Hybrid Search Threshold Calibration
*In adaptive hybrid scoring, the combined score distribution shifts far below standard semantic similarity cutoffs — a "reasonable" threshold of 0.5 collapses recall.*

## Key decisions

1. Calibrate the similarity threshold against the actual score distribution of the hybrid system, not a "standard" semantic similarity value. Without this, setting a threshold of 0.5 (a typical cosine similarity cutoff) causes recall collapse because most relevant results in a hybrid system score 0.05–0.4 under an adaptive-divisor fusion.

2. Use the threshold as a coarse pass-through gate before hybrid re-ranking — not as a quality filter after it. Without this, the threshold excludes memories that have strong keyword or entity signals before those signals are applied, making the hybrid layer ineffective.

3. When the hybrid scoring system uses an adaptive divisor (score normalized by the set of active signals), document the expected score range for API callers. Without this, callers infer the threshold operates on raw similarity (0.0–1.0) and set it 3–5× too high.

## Anti-patterns

- **What**: Setting `threshold=0.5` because it is the documented "good" cosine similarity cutoff for semantic search
- **Why**: In a system that fuses semantic + keyword + entity signals with an adaptive divisor, the score range compresses; the same "relevant" memory that scores 0.8 in pure-semantic search may score 0.25 in the hybrid system because max_possible is 2.5 instead of 1.0
- **Symptom**: After a config "cleanup" that normalizes the threshold to a standard value, search recall drops sharply; disabling individual signals does not help because the threshold already blocked the relevant candidates

## Structural template

```python
# Hybrid score uses adaptive divisor based on active signals
def hybrid_score(semantic, bm25, entity_boost, active_signals):
    max_possible = {
        frozenset({"semantic"}):                     1.0,
        frozenset({"semantic", "bm25"}):             2.0,
        frozenset({"semantic", "bm25", "entity"}):   2.5,
        frozenset({"semantic", "entity"}):           1.5,
    }[frozenset(active_signals)]
    return (semantic + bm25 + entity_boost) / max_possible
    # Result range: ~0.05–0.9 even for top-relevant memories

# Threshold must reflect this distribution — NOT semantic similarity norms
DEFAULT_THRESHOLD = 0.05   # permissive gate; ranking does the real work

def search(query, top_k, threshold=DEFAULT_THRESHOLD):
    # Pass nearly everything through; let ranking decide quality
    candidates = vector_store.search(query_vector=embed(query),
                                     limit=max(top_k * 4, 60),
                                     threshold=threshold)
    for c in candidates:
        c.score = hybrid_score(c.semantic, c.bm25, c.entity, active_signals)

    return sorted(candidates, key=lambda c: c.score, reverse=True)[:top_k]

# Document: threshold=0.05 is a pass-through gate for this system.
# Score range: 0.05–0.9. Do NOT raise to 0.3+ without profiling score distribution.
```
