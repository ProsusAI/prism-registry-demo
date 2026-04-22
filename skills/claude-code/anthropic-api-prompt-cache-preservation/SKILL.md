---
name: anthropic-api-prompt-cache-preservation
description: "Anthropic API prompt caching: how to keep server-side cache hits stable across
  a long agentic session. TRIGGER when designing session state models for
  agentic applications, when debugging unexpected cache misses after feature flag
  changes, when deciding whether to re-evaluate eligibility or feature headers
  per-turn, or when configuring compaction to avoid busting the prompt cache.
  Load even before cache misses are observed — these decisions must be made at
  design time."
---

# Anthropic API: Prompt Cache Preservation
*Architectural decisions for keeping prompt cache hits stable across long agentic sessions.*

## Who this is for
Teams building long-running agentic sessions on the Anthropic API with prompt caching enabled. The prompt cache saves 50-70K+ tokens per request at steady state; these decisions protect that saving from being accidentally invalidated by feature state changes that happen within a session.

## Key decisions

### 1. Latch beta headers sticky-on: once sent, never un-sent for the session
When a feature is first activated (fast mode, AFK mode, cache-editing mode, etc.), set a per-session latch. From that point on, always include the corresponding beta header — even if the feature is later disabled or toggled off.

**Why:** The prompt cache key includes the full set of request headers. Sending a beta header on turn 3, removing it on turn 4, then re-adding it on turn 5 generates three distinct cache keys. A 50K-token cached system prompt is re-read and re-billed each time the header set changes.

**What breaks without it:** Feature toggles that look cheap at the application level (a UI button) become expensive at the API level: each toggle busts the cache and forces a full context re-read on the next request.

### 2. Defer compaction boundary messages until after the API response, using actual token counts
When compaction edits the cache (e.g., microcompact deletes cached messages), don't emit the boundary message with a client-side estimated token count. Wait for the API response, read `cache_deleted_input_tokens` from the response, and emit the boundary message with the actual count.

**Why:** The boundary message influences subsequent request construction. If the estimate differs from the actual deletion, the next request's cache control parameters will be built on wrong counts — producing a cache key that doesn't match the server's state, causing a miss.

**What breaks without it:** Client-side estimates are wrong often enough to produce systematic cache misses after every microcompact operation. At scale this becomes a significant cost and latency increase.

### 3. Run compaction pipeline in fixed order: token budget → snip → microcompact → autocompact
Run context reduction steps in this specific order. The order is load-bearing: each step's freed tokens must be visible to the steps that follow.

**Why:** Snip frees tokens that the subsequent blocking-limit check uses. If autocompact runs before snip, it sees the unsnipped token count and may fire unnecessarily, wasting an API call and discarding granular context. The blocking-limit check must subtract snipped tokens or it produces false positives.

**What breaks without it:** Autocompact fires on sessions that snip would have brought under threshold. Granular context is replaced by a summary unnecessarily. Blocking-limit false positives prevent the model from continuing even though the actual usable context is below the limit.

### 4. Latch eligibility flags on first evaluation — don't re-check mid-session
For anything that affects request construction (cache TTL tier, beta header eligibility, quota checks), evaluate once at the start of the session and latch the result. Do not re-evaluate on subsequent turns.

**Why:** Mid-session state changes (e.g., user goes over quota, feature flag rolls out to 50%) would change the request signature. A session that started with 1h cache TTL switching to 5min TTL mid-session generates a new cache key and forces a full context re-read.

**What breaks without it:** Users who go over quota mid-session see a sudden performance degradation (cache miss + full context re-read) that is invisible to them and difficult to debug. A/B rollouts that change feature eligibility mid-session produce the same effect.

## Anti-patterns

- **What:** Re-evaluate feature header eligibility on every turn based on current state
  **Why it fails:** Any state change (quota, feature flag rollout, user settings change) changes the header set, which changes the cache key
  **Symptom:** Prompt cache hit rate drops to near-zero for active users who toggle features; billing spikes without obvious cause

- **What:** Emit compaction boundary messages with client-side estimated token counts
  **Why it fails:** Estimate ≠ actual `cache_deleted_input_tokens` from API; next request built on wrong counts
  **Symptom:** Systematic cache misses after every microcompact operation; intermittent misses that correlate with compaction events

- **What:** Run snip and autocompact in either order
  **Why it fails:** Autocompact's threshold check doesn't account for tokens snip will free; fires unnecessarily
  **Symptom:** More compaction events than expected; granular context replaced by summaries prematurely

## Structural template

```
// Session initialization — evaluate once, latch all
const session = {
  cache1hEligible: await checkCacheEligibility(),  // latched — never re-evaluated
  fastModeHeaderLatched: false,
  afkModeHeaderLatched: false,
}

// Per-turn request construction
function buildRequest(turn) {
  const headers = { ...baseHeaders }

  // Sticky-on: once latched, always sent
  if (turn.fastModeEnabled && !session.fastModeHeaderLatched) {
    session.fastModeHeaderLatched = true
  }
  if (session.fastModeHeaderLatched) headers['anthropic-beta'] += ',fast-mode'
  // ... repeat for each latchable header

  return headers
}

// Context reduction pipeline — order is fixed and load-bearing
async function reduceContext(messages) {
  const { messages: budgeted } = await applyToolResultBudget(messages)
  const { messages: snipped, tokensFreed } = snip(budgeted)
  const { messages: microcompacted, pendingBoundary } = await microcompact(snipped)
  const { messages: collapsed } = await collapseIfNeeded(microcompacted)
  const { messages: compacted } = await autocompactIfNeeded(collapsed, tokensFreed)
  // pendingBoundary: emit AFTER API response with actual cache_deleted_input_tokens
  return { messages: compacted, pendingBoundary }
}

// After API response: emit deferred boundary with actual counts
const response = await callModel(messages)
if (pendingBoundary) {
  const actualDeleted = response.usage.cache_deleted_input_tokens ?? 0
  yield makeBoundaryMessage(actualDeleted)
}
```

## Underlying principle
Prompt cache hits are sensitive to the exact byte signature of the request — headers, message content, and cache control parameters must all be identical across turns. Any mechanism that changes these values in response to in-session state (feature toggles, quota changes, compaction estimates) silently invalidates the cache. The solution is to treat the session as immutable with respect to cache-key-affecting decisions: latch at start, never re-evaluate, and use server-reported values (not estimates) for any measurement that affects subsequent request construction.

## Status
`seed-1-team` — promoted from cross-subsystem design analysis. Validate earlier than standard: needs a second team confirmation before treating as established.
