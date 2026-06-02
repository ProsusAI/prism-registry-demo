---
name: agent-memory-integrity
description: "Agent memory systems degrade LLM prefix caching, allow prompt injection via persisted content, and create inconsistent state when live writes update both the store and the in-context representation simultaneously. TRIGGER when: designing a memory system for a persistent agent, debugging prompt cache misses after memory writes, or adding external memory providers to an agent."
---

## Key Decisions

1. Build the system prompt once at session start from the memory snapshot and never rebuild it mid-session, even when the agent writes new memories during the turn. Mid-session writes update the on-disk store, but the in-context snapshot stays frozen until the next session. Rebuilding the prefix on every memory write invalidates the LLM's prompt cache on every turn.

2. Maintain two separate representations: a live store (reflected in tool responses, always current) and a session snapshot (frozen at load time, injected into the system prompt). Tool responses show the live state so the agent is never misinformed; the system prompt shows the session-start state so the cache prefix is stable.

3. Restrict the system to at most one active external memory provider. Each provider adds tools to the LLM's visible schema, and concurrent providers can write conflicting entries to the same user representation. Reject a second provider at registration time rather than discovering conflicts at query time.

4. Scan memory content for injection patterns at write time, not read time. Memory entries are injected verbatim into the system prompt at session start — a malicious entry that bypasses write-time scanning becomes a persistent system-prompt injection that activates in every subsequent session. Read-time scanning finds injections already written to disk; the persistent store is already compromised by then.

## Anti-patterns

- **What**: Rebuilding the system prompt prefix after each memory write to keep it current.
  **Why**: LLM providers cache the prompt prefix; any mutation invalidates the cache and forces a full recompute on every token.
  **Symptom**: Input token cost spikes after memory-heavy sessions; latency increases on every turn after the first memory write.

- **What**: Scanning memory content for injection only when reading it back out.
  **Why**: The persistent store is the attack surface; a scan at read time means malicious content was already written and could have been read by other sessions before this one.
  **Symptom**: An adversarially crafted user input causes the agent to write injected instructions to memory that activate in future sessions — days after the initial conversation.

- **What**: Allowing multiple external memory providers to be active simultaneously.
  **Why**: Each provider registers tools that inflate the schema; concurrent writes to overlapping keys corrupt the user model without a clear error.
  **Symptom**: Agent uses the wrong provider for a query; user profile entries conflict and the agent reports inconsistent information about the same person across turns.

## Structural Template

```
class MemoryManager:

    def load_from_disk(self):
        self._live   = load_store()          # mutable — updated by write tools
        self._snapshot = copy(self._live)    # frozen — injected into system prompt

    def build_system_prompt(self):
        return format_prompt(self._snapshot) # always session-start state

    def write_entry(self, content):
        scan_for_injection(content)          # raises on injection — write-time gate
        self._live.append(content)
        persist_to_disk(content)
        # _snapshot intentionally NOT updated; change takes effect next session

    def register_external_provider(self, provider):
        if self._external_provider is not None:
            raise ValueError("Only one external memory provider allowed")
        self._external_provider = provider
```
