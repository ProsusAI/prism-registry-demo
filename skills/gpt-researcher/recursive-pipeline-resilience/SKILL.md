---
name: recursive-pipeline-resilience
description: "Makes multi-step and recursive agent pipelines resilient to mid-execution failures, concurrency explosions, and redundant external calls. TRIGGER when: building a pipeline that spawns recursive sub-tasks, long-running agent tasks lose all work on failure, concurrent sub-tasks overwhelm external APIs, shared mutable state is accessed from concurrent pipeline branches"
---

## Key Decisions

1. Share deduplication state (e.g., visited URLs, processed queries) by reference across recursively spawned sub-pipeline instances rather than by copy. Without reference sharing, each recursion level re-processes resources from previous levels, producing duplicate work proportional to breadth x depth — a depth-3, breadth-4 pipeline processes the same top URLs 12 times instead of once.

2. Cache results from external tool calls per-request and reuse them across sub-queries within the same pipeline run. Without caching, each sub-query independently calls the same external APIs, multiplying latency and cost by the number of sub-queries. The cache key should be the tool + query combination, scoped to the current pipeline run (not cross-request).

3. Gate recursive task spawning with a concurrency semaphore sized to your external API rate limits, not to available compute. Recursive breadth x depth pipelines spawn tasks exponentially — a breadth-4, depth-3 pipeline can spawn 84 concurrent tasks. Without a semaphore, these exhaust LLM API rate limits within seconds, causing cascading 429 errors that are retried into further rate limit violations.

4. Combine a concurrency semaphore (controls parallelism) with a rate limiter (controls throughput) for external resource access like web scraping. A semaphore alone allows burst requests that trigger IP bans; a rate limiter alone serializes work unnecessarily. The combination allows N concurrent connections with a minimum inter-request delay, matching the access pattern that most web servers expect.

## Anti-patterns

- **What**: No checkpointing of accumulated context in a multi-step pipeline — failure at any step discards all prior work.
- **Why**: A recursive research run (breadth=4, depth=3) that fails at depth 2 has already completed 15+ minutes of LLM calls, web scraping, and embedding, all of which must be repeated from scratch because no intermediate state was persisted.
- **Symptom**: Users report that deep research "never completes" — it fails intermittently due to transient API errors and always restarts from zero, making completion probability decrease exponentially with pipeline depth.

- **What**: LLM call wrapper contains a retry loop that returns immediately after the first successful response, providing no actual retry on transient failures.
- **Why**: The retry loop was written to handle failures but the control flow returns on success before checking if retry is needed on failure — a common bug in hand-rolled retry logic that's invisible in testing (tests use working APIs).
- **Symptom**: Transient 429/500 errors from the LLM API during report generation crash the entire pipeline despite valid accumulated context; the error rate correlates with API provider load patterns (peaks during business hours) and is not reproducible in dev environments.

- **What**: Pipeline steps return empty strings on failure instead of raising typed exceptions, making error and success indistinguishable to callers.
- **Why**: Each step catches all exceptions and returns a default value (empty string) to prevent crashing the pipeline, but callers interpret empty strings as "no results" rather than "step failed" — the error is swallowed.
- **Symptom**: Users receive blank or truncated reports with no error message; diagnosing whether the cause was empty search results, LLM refusal, or API timeout requires reading server logs because the pipeline reported success.

- **What**: Shared numeric accumulators (e.g., cost tracking) are mutated from concurrent async tasks without synchronization.
- **Why**: Multiple sub-queries complete simultaneously and increment the same counter; under Python's asyncio, a read-modify-write sequence can interleave across await points even without true threading, causing lost updates.
- **Symptom**: Cost tracking underreports actual spend by 10-30% under high concurrency; the discrepancy is only visible when comparing tracked costs against provider invoices at month end.

## Structural Template

```
state:
  visited_resources: shared_set    # passed by reference to all sub-instances
  tool_cache: dict[key, result]    # scoped to current pipeline run
  accumulated_context: list        # checkpointed after each step
  cost_accumulator: locked_counter # synchronized writes

config:
  max_concurrent_tasks: 4          # sized to API rate limits
  max_concurrent_scrapers: 15      # sized to target site tolerance
  rate_limit_delay: 50ms           # minimum inter-request delay
  checkpoint_storage: disk | cache

function run_recursive_pipeline(query, depth, breadth, state):
  if depth == 0: return state.accumulated_context

  sub_queries = generate_sub_queries(query, breadth)

  semaphore = Semaphore(config.max_concurrent_tasks)
  results = parallel_map(sub_queries, lambda q:
    with semaphore:
      process_sub_query(q, state)
  )

  checkpoint(state, depth)  # persist before recursion

  follow_ups = extract_follow_up_queries(results)
  return run_recursive_pipeline(follow_ups, depth - 1, breadth, state)

function process_sub_query(query, state):
  # Check tool cache before calling external tools
  cached = state.tool_cache.get(tool + query)
  if cached: return cached

  resources = search(query)
  new_resources = [r for r in resources if r not in state.visited_resources]
  state.visited_resources.add_all(new_resources)

  content = scrape_with_rate_limit(new_resources)
  compressed = compress_context(content, query)

  state.tool_cache.set(tool + query, compressed)
  state.cost_accumulator.add(cost)  # synchronized
  return compressed

function scrape_with_rate_limit(urls):
  scraper_semaphore = Semaphore(config.max_concurrent_scrapers)
  rate_limiter = TokenBucket(delay=config.rate_limit_delay)
  return parallel_map(urls, lambda url:
    with scraper_semaphore:
      rate_limiter.wait()
      scrape(url)
  )
```
