---
name: claude-code-hook-block-exit-code
description: "Claude Code PreToolUse hooks must exit 2 to block tool execution — exit 1 signals hook failure but allows the tool to proceed. TRIGGER when: writing a Claude Code PreToolUse hook that should prevent a tool from executing, implementing access control or guardrails in a hook, debugging a hook that runs but does not block the tool."
---

# Claude Code Hook Block Exit Code
*In the Claude Code hook protocol, exit 2 means "block this tool"; exit 1 means "hook itself failed" — the tool still executes on exit 1.*

## Key decisions

1. Use `exit 2` (not `exit 1`) to block tool execution from a `PreToolUse` hook. Without this, `exit 1` — the conventional Unix error code — signals hook failure to the harness but allows the tool to proceed as if no hook ran.

2. Write the block reason to stderr before `exit 2`. Without this, Claude receives no explanation for why the tool was blocked and may retry the same action or behave unpredictably; the stderr message is surfaced to the model as the block reason.

3. End all non-blocking hooks in `2>/dev/null || true` to ensure hook errors never prevent a tool from running. Without this, a transient failure in the hook (e.g., missing binary, Python error) blocks a tool that was meant to be allowed.

## Anti-patterns

- **What**: Exiting with `exit 1` from a `PreToolUse` hook when validation fails and the tool should be blocked
- **Why**: The Claude Code hook protocol distinguishes hook failure (exit 1) from an intentional block decision (exit 2); only exit 2 prevents the tool from running
- **Symptom**: The hook script runs without shell error; the validation logic executes; but the tool it was designed to block proceeds — the only clue is that the intended block had no effect

## Structural template

```bash
#!/bin/bash
# PreToolUse hook: block writes to protected paths

TOOL_INPUT=$(cat)   # tool parameters as JSON on stdin
TARGET_PATH=$(echo "$TOOL_INPUT" | jq -r '.path // ""')

if [[ "$TARGET_PATH" == *"/protected/"* ]]; then
    # Write block reason to stderr — shown to the model
    echo "Blocked: writes to /protected/ must use the designated API." >&2
    exit 2   # 2 = block; tool does NOT execute
fi

# Validation passed — allow the tool
exit 0

# For best-effort (never-block) hooks:
# end in "2>/dev/null || true" so any hook error doesn't block the tool
# example: some_script.py "$TOOL_INPUT" 2>/dev/null || true
```
