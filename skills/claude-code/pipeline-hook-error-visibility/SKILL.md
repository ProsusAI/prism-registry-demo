---
name: pipeline-hook-error-visibility
description: "Ensure optional post-step hooks in a processing pipeline surface failures rather than silently swallowing them. TRIGGER when: adding a hook invocation to a processing pipeline, designing optional post-processing callbacks, wiring post-sampling or post-tool hooks into an agentic loop."
---

# Pipeline Hook Error Visibility
*Prevent optional post-step hooks from silently swallowing failures — fire-and-forget is the natural default but produces invisible persistent failures in production.*

## Key decisions

1. Never invoke a hook with bare `void hookFn()` or an unhandled `Promise`; always attach `.catch(logError)` at minimum. Without this, any exception thrown inside the hook is swallowed by the runtime — no log line, no user-visible warning, no observable signal that the hook is broken.

2. Maintain a consecutive-failure counter per hook; after a threshold (e.g., 3 consecutive failures), surface a system warning to the operator or user rather than continuing to silently skip. Without this, a misconfigured or externally-broken hook fails on every invocation with no circuit-break — there is no way to distinguish a transient error from a permanently broken hook.

3. Treat hook failure as a non-fatal degraded state, not as a hard error — the pipeline must continue even when a hook fails. Without this, a single flaky hook blocks every downstream step; the hook is optional enrichment, not a correctness dependency.

## Anti-patterns

- **What**: `void executeHook(ctx)` — fire and forget with no error handler
- **Why**: The hook runs asynchronously; without `.catch`, the rejected Promise disappears into the event loop with no surface area
- **Symptom**: Hook failures are invisible at runtime — only discovered via full audit of logs or when a downstream consumer notices missing enrichment data, typically days after the bug was introduced

- **What**: No failure counter — treat each hook invocation as independent
- **Why**: Without tracking consecutive failures, there is no signal that a hook is consistently broken vs. occasionally flaky
- **Symptom**: A hook that fails on every call continues to be invoked indefinitely; operators see no warning; the cost (latency, resource use for the failed call) accumulates silently

- **What**: Propagate hook errors back to the caller and fail the pipeline step
- **Why**: Hooks are optional enrichment; making the pipeline's correctness depend on them inverts their intended role
- **Symptom**: A transient hook failure (network blip, external service down) kills the pipeline step for the user; the hook's optional nature is violated

## Structural template

```
# Hook invocation wrapper
const consecutiveFailures: Map<string, number> = new Map()
const FAILURE_THRESHOLD = 3

async function invokeHook(hookName: string, hookFn: () => Promise<void>, ctx):
    try:
        await hookFn(ctx)
        consecutiveFailures.set(hookName, 0)   # reset on success
    catch (err):
        const count = (consecutiveFailures.get(hookName) ?? 0) + 1
        consecutiveFailures.set(hookName, count)
        logError(`Hook "${hookName}" failed (${count} consecutive)`, err)
        if count >= FAILURE_THRESHOLD:
            emitSystemWarning(`Hook "${hookName}" has failed ${count} times in a row — check configuration`)
        # do NOT rethrow — hook is optional; pipeline continues

# Call site
await invokeHook("post-sampling", executePostSamplingHooks, { response, ctx })
# NOT: void executePostSamplingHooks(response, ctx)
```
