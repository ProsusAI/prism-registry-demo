---
name: llm-provider-failover-architecture
description: "LLM provider reliability strategies that seem correct — key rotation, backoff with jitter — fail at scale due to credential ordering, correlated retries, and missing circuit breakers. TRIGGER when: implementing retry or failover logic for an LLM gateway, debugging thundering-herd rate-limit storms, or building a multi-provider routing layer."
---

## Key Decisions

1. When a credential fails, exhaust all configured credentials for the same provider before switching providers. Provider failover changes the active model — with associated quality and latency shifts — and is unnecessary for transient key-level failures. Only trigger a provider switch when every key for the current provider has been tried and failed.

2. Seed retry jitter independently for each concurrent session using a cross-combination of wall clock time and a monotonic per-session counter. Seeding jitter only from wall clock synchronizes all sessions hitting the same rate limit at the same moment, creating a thundering herd that re-triggers the limit after every wait interval.

3. Implement a provider-level circuit breaker that stops routing to a provider after a threshold of consecutive failures within a rolling time window. Backoff alone keeps retrying a consistently degraded provider for the entire session duration, consuming the iteration budget without producing output.

## Anti-patterns

- **What**: Failing over to a different provider immediately on any single key error.
  **Why**: Transient key-level errors (rate limit on one key, temporary auth expiry) don't indicate provider-level degradation.
  **Symptom**: Provider switches are frequent and noisy; the active model changes mid-session on routine rate-limit events; users see inconsistent model behavior across turns.

- **What**: Generating retry jitter from a random source seeded only by wall clock time.
  **Why**: Concurrent sessions starting near the same time share the same seed and generate correlated jitter sequences, converging on the same retry moments.
  **Symptom**: After a mass rate-limit event, all sessions retry simultaneously; the synchronized retry wave re-triggers the limit; sessions are trapped in a repeating correlated backoff cycle.

- **What**: Relying on exponential backoff with no upper bound on retries against a failing provider.
  **Why**: Backoff waits and retries regardless of whether the provider has recovered; it has no trip condition.
  **Symptom**: Sessions involving a degraded provider appear to run for hours while producing no output; the agent exhausts its iteration budget entirely on retries.

## Structural Template

```
class CircuitBreaker:
    def __init__(self, failure_threshold=5, window_seconds=300):
        self.failures = defaultdict(deque)   # provider → timestamps

    def record_failure(self, provider: str) -> None:
        self.failures[provider].append(time.monotonic())

    def is_open(self, provider: str) -> bool:
        cutoff = time.monotonic() - self.window_seconds
        recent = [t for t in self.failures[provider] if t > cutoff]
        return len(recent) >= self.failure_threshold


def route_request(providers, credentials, circuit_breaker):
    for provider in providers:
        if circuit_breaker.is_open(provider):
            continue                            # tripped — skip this provider

        for key in credentials[provider]:
            result = try_request(provider, key)
            if result.success:
                return result
            if result.error in KEY_LEVEL_ERRORS:
                continue                        # try next key, same provider
            break                               # non-key error → try next provider

        circuit_breaker.record_failure(provider)

    raise AllProvidersExhausted()


def jitter_delay(base_delay: float, session_counter: int) -> float:
    seed = time.time_ns() ^ (session_counter * 0x9E3779B9)
    return base_delay * (1 + Random(seed).uniform(0, 1))
```
