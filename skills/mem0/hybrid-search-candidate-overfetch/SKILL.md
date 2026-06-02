---
name: hybrid-search-candidate-overfetch
description: "Hybrid search (semantic + keyword + signal boost) must over-fetch semantic candidates before re-ranking — fetching exactly top-k from the vector store gives de facto pure-semantic results regardless of hybrid scoring. TRIGGER when: implementing hybrid search over a vector store, adding keyword or entity-boost re-ranking on top of semantic retrieval, designing a retrieval pipeline that combines multiple signals."
---

# Hybrid Search Candidate Overfetch
*Fetching exactly top-k from the vector store before hybrid re-ranking pre-filters out the memories that non-semantic signals would have promoted.*

## Key decisions

1. Fetch at least 4× the requested result count (with a floor of 50–60) from the semantic retrieval layer before applying non-semantic signals. Without this, memories ranked outside the top-k semantically never enter the hybrid scoring pool — keyword and entity signals from lower semantic positions are never applied.

2. Apply all non-semantic scoring to the full over-fetched candidate set, then cut to top-k after hybrid re-ranking. Without this, a memory with weak semantic similarity but strong keyword or entity signal that would rank first under hybrid scoring is excluded before scoring begins.

3. Make the over-fetch multiplier configurable. Without this, deep recall queries — where the best result has low semantic similarity — cannot benefit from larger candidate pools without a code change.

## Anti-patterns

- **What**: Passing `limit=top_k` to the vector store search, then applying keyword re-ranking to those top-k results
- **Why**: The semantic pre-filter excludes all candidates outside the top-k; the hybrid layer operates on an already-semantically-filtered set with no candidates to promote from lower semantic ranks
- **Symptom**: Adding keyword and entity-boost scoring to a retrieval system produces no measurable improvement in recall over pure-semantic; disabling individual signals returns identical result sets — because semantic pre-filtering already matched the same top-k before any hybrid signal could act

## Structural template

```python
def search(query: str, top_k: int, threshold: float = 0.05,
           overfetch_multiplier: int = 4, overfetch_floor: int = 60) -> list:

    # Over-fetch: give non-semantic signals a full candidate pool to work on
    candidate_limit = max(top_k * overfetch_multiplier, overfetch_floor)

    candidates = vector_store.search(
        query_vector=embed(query),
        limit=candidate_limit,        # NOT top_k
        threshold=threshold,
    )

    candidate_ids = [c.id for c in candidates]

    # Score non-semantic signals across all candidates
    bm25_scores   = keyword_index.score(query, ids=candidate_ids)
    entity_scores = entity_index.score(query_entities, ids=candidate_ids)

    # Hybrid re-rank the full pool
    for c in candidates:
        c.combined = hybrid_fuse(c.semantic, bm25_scores[c.id], entity_scores[c.id])

    # Cut to top_k only after re-ranking
    return sorted(candidates, key=lambda c: c.combined, reverse=True)[:top_k]
```
