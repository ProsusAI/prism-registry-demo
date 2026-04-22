---
name: context-compression-in-rag-pipelines
description: "Manages context accumulation and compression when passing retrieved content to LLMs, preventing context overflow and cost explosion in multi-step retrieval pipelines. TRIGGER when: building a RAG pipeline with multiple retrieval steps, retrieved context exceeds model limits, implementing recursive research with depth-based context accumulation, choosing between embedding filtering and LLM reranking for context relevance"
---

## Key Decisions

1. Use embedding similarity filtering rather than LLM-based reranking as the default context compression strategy. Embedding filtering is 10-100x cheaper per query and fast enough for real-time pipelines. Reserve LLM reranking as an opt-in upgrade for queries requiring nuanced relevance judgment — negation ("not X"), conditional relevance ("X only if Y"), or multi-hop reasoning where embedding cosine similarity fails systematically.

2. Set a compression threshold below which documents skip chunking and embedding entirely. Short documents (under ~8000 characters) should pass through as-is because splitting them into chunks and filtering by similarity risks discarding relevant content that falls below the threshold — a paragraph that's tangentially related in isolation but critical in the context of the full document.

3. Construct each pipeline step's LLM messages from scratch (system + user with accumulated context) rather than maintaining conversation history across steps. Conversation history accumulates tokens geometrically across pipeline steps; stateless calls with explicit context injection keep each step's token budget predictable. The trade-off is that inter-step coherence depends entirely on the quality of the accumulated context string, not on the LLM's memory of prior reasoning.

4. For recursive or depth-based pipelines that accumulate context across levels, enforce a hard context word limit and evict oldest items first. Without a ceiling, recursive depth > 2 accumulates context beyond any model's window, causing silent truncation or API errors. Oldest-first eviction preserves the most specific, recently-gathered context from deeper levels while discarding broader initial context that has already been refined by subsequent queries.

## Anti-patterns

- **What**: Using LLM reranking by default when embedding filtering would suffice, without measuring the relevance improvement against the 10-100x cost increase.
- **Why**: LLM reranking is assumed to be universally better, but for factual/keyword-matchable queries (the majority in research pipelines), embedding similarity captures relevance adequately — the LLM reranker adds cost without measurably improving downstream report quality.
- **Symptom**: Pipeline costs are dominated by the reranking step rather than the final generation step, with no measurable quality improvement in A/B evaluation against embedding-only filtering.

## Structural Template

```
config:
  compression_threshold: 8000  # chars; below this, skip chunking
  similarity_threshold: 0.40   # embedding cosine similarity cutoff
  max_context_words: 25000     # hard ceiling for recursive accumulation
  eviction_strategy: oldest_first

function compress_context(documents, query, embedding_model):
  # Short-circuit: small documents skip chunking entirely
  if total_chars(documents) < config.compression_threshold:
    return documents_as_context_string(documents)

  chunks = split_into_chunks(documents)
  scored_chunks = embed_and_score(chunks, query, embedding_model)
  filtered = [c for c in scored_chunks if c.score >= config.similarity_threshold]
  return join_as_context_string(filtered)

function accumulate_context(existing_context, new_context):
  combined = existing_context + new_context
  if word_count(combined) > config.max_context_words:
    combined = evict(combined, config.eviction_strategy,
                     target=config.max_context_words)
  return combined

function build_step_messages(system_prompt, accumulated_context, step_query):
  # Stateless: no conversation history carried between steps
  return [
    {role: system, content: system_prompt},
    {role: user, content: format(step_query, context=accumulated_context)}
  ]
```
