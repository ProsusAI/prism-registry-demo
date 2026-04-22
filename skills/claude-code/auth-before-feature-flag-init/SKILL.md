---
name: auth-before-feature-flag-init
description: "Initialize authentication before any feature-flag or remote config system that uses user identity for targeting — otherwise authenticated users are denied on cache miss. TRIGGER when: configuring startup initialization order for a service with user-specific feature flags, adding feature gating to an authenticated endpoint, integrating a feature-flag SDK into an auth-required service."
---

# Auth Before Feature-Flag Initialization
*Complete authentication before evaluating user-targeted feature flags — without auth context, the flag system returns stale or default values, denying legitimate users.*

## Key decisions

1. Always complete the authentication flow before initializing or querying a feature-flag system that uses user identity (user ID, org, roles) for targeting rules. Without this, the flag system has no user context to evaluate targeting — it returns the disk-cached value on cache miss, or `false`/default if the cache is empty, denying authenticated users who would otherwise be granted access.

2. Treat a feature-flag system returning defaults in an authenticated context as a degraded state, not a safe fallback — add a log line or metric when the flag SDK falls back to defaults so you can distinguish "feature is off" from "flag evaluated without user context." Without this, access denials caused by auth/flag ordering bugs are indistinguishable from intentional feature-off states in production logs.

3. If the flag system requires a network call to fetch user-specific rules, do not block startup on it — fetch eagerly after auth completes, but allow the system to serve defaults until the fetch resolves. Without this, a flag system network timeout during startup blocks the entire auth flow.

## Anti-patterns

- **What**: Check feature flags at the top of the request handler, before auth middleware runs
- **Why**: Flag middleware is cheap (memory lookup) vs. auth (token validation, network); developers sequence cheap operations first
- **Symptom**: Authenticated users with valid tokens are denied at the feature gate on the first request after a cache expiry — the flag returns `false` because user context wasn't set yet; only reproduces on cache miss, which is rare in testing

- **What**: Treat feature-flag defaults as a safe fallback for unauthenticated state
- **Why**: Defaults are typically "off" or restrictive, so it feels conservative to return them when context is missing
- **Symptom**: Legitimate users are silently denied with no distinguishable error from an intentional feature-off; debugging requires reconstructing the initialization sequence from logs

- **What**: Initialize the flag SDK once at process startup before any request handling
- **Why**: One-time initialization at startup is simpler than per-request initialization; the SDK is already initialized by the time requests arrive
- **Symptom**: The SDK initializes without a user context; per-user targeting rules are never applied; all users get the default (often "off") regardless of their eligibility

## Structural template

```
# Startup / request entrypoint sequence
async function handleRequest(req):
    
    # Step 1: Auth — must complete before flag evaluation
    authContext = await authenticate(req)
    if not authContext.valid:
        return 401
    
    # Step 2: Initialize flag system with user identity
    # Flag SDK has auth context now — targeting rules can be evaluated correctly
    flagClient.identify({ userId: authContext.userId, org: authContext.org })
    
    # Step 3: Evaluate feature gate
    isEnabled = flagClient.isEnabled("feature-name")
    if not isEnabled:
        logDebug("feature-name disabled for user", { userId: authContext.userId,
                  hadCachedRules: flagClient.hasCachedRules() })  # distinguish off vs. no-context
        return 403
    
    # Step 4: Proceed with request
    return handleAuthorizedRequest(req, authContext)

# Flag client initialization (process-level, not per-request)
# Load disk cache at startup — do NOT evaluate user-specific rules here
flagClient = new FlagClient({ diskCachePath: "..." })
await flagClient.loadCache()    # fast, local; safe before auth
# Do NOT: await flagClient.fetchRules()  — no user context yet
```
