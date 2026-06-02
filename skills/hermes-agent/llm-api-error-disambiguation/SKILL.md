---
name: llm-api-error-disambiguation
description: "LLM API error codes and HTTP status codes frequently carry multiple semantically distinct meanings that require different recovery actions. TRIGGER when: implementing retry logic for an LLM API client, debugging agents that loop on recoverable errors, or building failover logic across LLM providers."
---

## Key Decisions

1. Treat certain 400 Bad Request responses as transient and retryable. Stream corruption, token signature mismatches after context switches, and temporary serialization errors can return 400 with format-error-like bodies even though the request itself is valid. Without special-casing these, the agent permanently aborts sessions it could have recovered from by retrying once.

2. Disambiguate billing-related status codes by inspecting the error body, not just the status code. A single code can indicate either a periodic quota reset (retryable — wait for the reset window) or permanent billing exhaustion (not retryable — rotate credential). Treating all instances identically either abandons viable credentials or exhausts the pool retrying against a permanently blocked account.

3. Resolve context length limits through a multi-step fallback chain rather than a single lookup. Local models, custom endpoints, and provider-specific APIs each expose context limits through different mechanisms. A single lookup fails silently for uncovered deployment scenarios, assigning an incorrect context budget that causes either premature compression or invisible overflow.

4. Classify connection-reset errors with no HTTP status code as context overflow when token count is near the limit. Providers can drop connections on oversized requests without returning a structured error body. Without a token-count heuristic, the agent retries indefinitely on what appears to be a network error but is actually a payload-too-large rejection.

## Anti-patterns

- **What**: Mapping all 400 responses to "malformed request — abort permanently."
  **Why**: Some 400s are caused by transient state (stream corruption, token buffer reuse) and are not structural errors in the request.
  **Symptom**: Sessions with extended-output or multi-turn streaming die permanently on rare transient API errors that would resolve on a single retry.

- **What**: Treating all instances of a billing-related status code identically — either all retry or all fail.
  **Why**: The same status code covers both temporary quota resets and permanent billing failure; the correct action is different in each case.
  **Symptom**: Either all sessions fail after a quota reset that clears in minutes, or credential rotation never triggers on a genuinely exhausted account.

- **What**: Looking up model context length from a single source.
  **Why**: Local models, offline deployments, and custom-endpoint models don't all expose limits through the same channel.
  **Symptom**: Locally-served or custom-endpoint models receive an incorrect context budget — either compressing too early or overflowing silently.

## Structural Template

```
function classify_api_error(status, body, approx_tokens, context_length):
    if status == 400:
        if is_transient_stream_error(body):   # e.g. corruption signature in body
            return RETRY_TRANSIENT
        return ABORT_FORMAT_ERROR

    if is_billing_code(status):
        if "try again" in body and "limit" in body:
            return RETRY_QUOTA_RESET          # periodic window — wait, then retry
        if "insufficient" in body or "credits" in body:
            return ROTATE_CREDENTIAL          # permanent — move to next key

    if status is None and is_connection_reset(error):
        if approx_tokens > context_length * 0.6 or approx_tokens > 120_000:
            return CONTEXT_OVERFLOW           # compress, then retry
        return RETRY_TRANSIENT

    return standard_classification(status)


function get_context_length(model_id, endpoint):
    return first_that_succeeds([
        lambda: user_config_override(model_id),
        lambda: persistent_cache(model_id),
        lambda: active_endpoint_metadata(endpoint),
        lambda: local_server_query(endpoint),
        lambda: provider_models_api(model_id),
        lambda: provider_router_metadata(model_id),
        lambda: suffix_pattern_match(model_id),
        lambda: models_directory_lookup(model_id),
        lambda: DEFAULT_128K,
    ])
```
