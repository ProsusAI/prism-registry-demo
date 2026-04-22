---
name: tool-result-persistence-self-reference
description: "Prevent circular references when persisting large tool results to disk for session resume — tools that read arbitrary paths or URLs must not have their output persisted. TRIGGER when: adding result persistence to a tool system, designing session-resume tool result replay, adding a new tool that reads files or external resources."
---

# Tool Result Persistence Self-Reference Prevention
*Exclude self-reading tools from result persistence — a file-reading tool whose output is persisted as a path reference creates a circular reference the model can follow indefinitely.*

## Key decisions

1. Classify every tool as either self-bounding (reads back its own result, e.g., file-read, URL-fetch, resource-read) or externally-bounded, and hard-exclude self-bounding tools from result persistence. Without this, a file-reading tool's large result is persisted to disk and replaced in-context with `<persisted-output path=/session/tool-results/xyz>` — the model then calls the same tool with that path, reading the persisted result back as if it were user content.

2. Implement the self-bounding exclusion as a hard sentinel checked before any remote override path (feature flags, admin overrides). Without this, a misconfigured override re-enables persistence for a self-reading tool — the override path bypasses the safety check.

3. Scope tool result persistence to sessions that own a stable storage root (e.g., interactive sessions, named agent sessions) and exclude ephemeral forked agents (compact summarizers, memory enrichers). Without this, ephemeral agents persist results to a storage path they don't own — results are orphaned when the ephemeral agent exits, and the session that does own the path may read stale data on resume.

## Anti-patterns

- **What**: Apply the same persistence policy (size threshold) to all tools uniformly
- **Why**: The threshold only accounts for in-context size, not for whether the tool can re-read the persisted path
- **Symptom**: File-read tool results are replaced with `<persisted-output path=...>` tags; on the next turn, the model calls file-read on the persisted path; the cycle repeats and the model's context fills with nested references — only visible in sessions with large file reads

- **What**: Allow admin or feature-flag overrides to enable persistence for any tool
- **Why**: Overrides bypass the classification logic and can re-enable persistence for excluded tools
- **Symptom**: A "safe" override that increases persistence thresholds accidentally enables circular persistence for self-reading tools; the bug only surfaces in long sessions when the model follows a persisted reference

- **What**: Persist tool results for all agent types that share a session ID
- **Why**: Ephemeral forked agents share the session ID for tracing but don't own the session's storage root
- **Symptom**: Ephemeral agent results accumulate in the session storage directory; on resume, the session sees stale tool results from compact or memory agents as if they were from the interactive session

## Structural template

```
# Tool definition
class FileReadTool:
    maxResultSizeChars = Infinity   # hard opt-out from persistence — reads arbitrary paths

class BashTool:
    maxResultSizeChars = 50_000     # eligible for persistence

# Persistence decision
function getEffectivePersistenceThreshold(tool, overrides):
    declared = tool.maxResultSizeChars
    
    # Hard sentinel: check BEFORE any override path
    if not isFinite(declared):
        return declared   # Infinity — skip persistence entirely
    
    # Override path only reached for externally-bounded tools
    override = overrides?.[tool.name]
    if typeof override === "number" and isFinite(override) and override > 0:
        return override
    
    return declared

# Session-scope guard
function shouldPersistForSession(querySource: string) -> bool:
    # Only interactive and named agent sessions own a stable storage root
    return querySource.startsWith("agent:") or querySource.startsWith("repl_main_thread")
    # NOT: compact, session_memory, or other ephemeral forked agents
```
