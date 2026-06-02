---
name: multi-tenant-search-scope-bypass
description: "In multi-tenant memory systems, env vars or runtime flags that widen search scope beyond the requesting user must be explicitly audited — they bypass per-user identity isolation and expose cross-tenant data. TRIGGER when: adding a global-search or admin-mode feature to a multi-tenant system, implementing env-var-controlled search scope overrides, designing wildcard or cross-user queries."
---

# Multi-Tenant Search Scope Bypass
*An env-var-controlled wildcard search bypasses per-user identity isolation — a single misconfiguration exposes all tenants' data with no audit trail.*

## Key decisions

1. Log at WARN or AUDIT level every search that uses a wildcard identity filter. Without this, a misconfigured deployment variable silently exposes all tenants' data to every search request; there is no trace in logs to detect or attribute the exposure.

2. Require an explicit opt-in at the API call site, not only at the env-var-read location. Without this, enabling the env var affects every single search in the process, including those that had no intention of querying cross-tenant data.

3. Validate the widened scope at the authorization boundary before issuing the cross-tenant query. Without this, any caller who knows the bypass mechanism — env var name, header name, or flag name — can issue cross-tenant searches without any access check.

## Anti-patterns

- **What**: Reading `GLOBAL_SEARCH=true` from env and replacing all identity filters with a wildcard that matches all tenants
- **Why**: Env vars are process-wide; one misconfiguration (e.g., copied from a dev config into a shared namespace) widens every search for every user without any per-call decision
- **Symptom**: In a shared deployment namespace, the env var set for dev leaks into the production pod; all searches return results from every tenant; no error is raised; the exposure is only visible as unexpected result volume and content in audit logs

## Structural template

```python
def build_search_filters(requesting_user_id: str, request_flags: dict, config: Config):
    global_scope_enabled = config.get("GLOBAL_SCOPE", "false").lower() == "true"
    caller_opted_in      = request_flags.get("global_scope", False)

    if global_scope_enabled and caller_opted_in:
        # Both conditions required: env-var gates the feature, caller must opt in per-call
        audit_log.warning(
            "cross_tenant_search",
            requesting_user=requesting_user_id,
            reason="global_scope requested",
        )
        # Authorization check before issuing the widened query
        if not authorization.can_search_all(requesting_user_id):
            raise PermissionError("cross-tenant search requires elevated permission")
        return {"OR": [{"user_id": "*"}]}

    # Default: always scope to the requesting user
    return {"user_id": requesting_user_id}
```
