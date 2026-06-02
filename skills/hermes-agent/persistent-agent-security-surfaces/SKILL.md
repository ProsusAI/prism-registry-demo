---
name: persistent-agent-security-surfaces
description: "Agents that write or modify their own persistent configuration create attack vectors — malicious tasks can plant injections in files that execute in future sessions. TRIGGER when: building an agent that learns or saves reusable skill or config files, implementing capability gates or redaction controls, or supporting multi-user deployments of a shared agent."
---

## Key Decisions

1. Apply the same security scanning to agent-created or agent-edited persistent files as to externally sourced ones. When an agent writes a skill, rule, or config file, scan it for injection patterns, path traversal, and code execution vectors before persisting. Skipping the scan treats agent-written content as trusted when it is as adversarially controllable as anything from outside.

2. Snapshot security-critical flags (redaction enabled, capability scope, tool allowlists) at module initialization time, not at call time. If these flags are read from environment at call time, an LLM-generated shell command can export a modified value and disable protections mid-session. Import-time snapshotting makes these flags immutable for the session's lifetime.

3. Add a review or validation gate before agent-generated content becomes canonical and reusable in future sessions. Agent-written content that is incorrect — wrong procedure, wrong assumption — gets reused across future sessions without expiry. A single bad save propagates silently until a user explicitly patches it.

## Anti-patterns

- **What**: Agent-modified skill or rule files skip the security checks applied to hub-installed content.
  **Why**: The agent is as adversarially controllable as external content — a malicious task can instruct the agent to write injected instructions to a file that loads on future sessions.
  **Symptom**: A planted instruction in a persisted file activates in the next session, executing actions the user never approved.

- **What**: Security-sensitive runtime flags read from environment at call time rather than at import time.
  **Why**: The agent has access to shell or terminal tools and can emit `export FLAG=disabled` before a sensitive call, mutating the environment mid-session.
  **Symptom**: Credentials or sensitive outputs appear in plain text after a session where the agent used a shell tool; the redaction flag is found disabled in logs.

- **What**: All users in a shared deployment receive the same tool scope regardless of identity.
  **Why**: Tool access is determined at agent startup, not per-request; there is no runtime check tied to the requesting identity.
  **Symptom**: A non-admin user successfully calls an administrative tool; access control exists only at the UI layer and is bypassed by direct API calls.

## Structural Template

```
# At module import time — immutable for session lifetime
REDACTION_ENABLED = os.environ.get("REDACT_SECRETS", "true").lower() == "true"
CAPABILITY_LEVEL  = os.environ.get("CAPABILITY_SCOPE", "standard")


class AgentFileGuard:
    INJECTION_PATTERNS = [
        r"ignore\s+previous",
        r"<\|.*?\|>",
        r"system\s*:",
        r"you are now",
    ]

    def validate(self, content: str, path: str) -> None:
        check_no_path_traversal(path)
        for pattern in self.INJECTION_PATTERNS:
            if re.search(pattern, content, re.IGNORECASE):
                raise SecurityError("Injection pattern in agent-written file")
        # Same scanner as externally-sourced files — no separate trust level


def get_tool_definitions(user_context) -> list[Tool]:
    allowed = TOOL_ALLOWLIST.get(user_context.user_id, DEFAULT_SCOPE)
    return [t for t in ALL_TOOLS if t.name in allowed]
    # Per-user scoping at call time, not at startup
```
