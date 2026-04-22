---
name: agent-session-identity-location-split
description: "Agent session design: separating stable session identity from mutable working location. TRIGGER when: designing session storage for agents that support worktrees, remote execution, or mid-session directory changes; when debugging resume failures after a working directory change; when comparing session IDs across API versions or compat layers; or when a team member asks why getCwd()should not be used to derive session file paths."
---

# Agent Sessions: Identity vs. Location Split
*Architectural decisions for maintaining stable session identity when the working directory can change.*

## Who this is for
Teams building agents or CLI tools that persist session state to disk and support mid-session directory changes (worktrees, remote execution, `cd` operations). These decisions are non-obvious without having debugged a resume failure caused by identity/location conflation.

## Key decisions

### 1. Set the session's identity root once at startup — never update it for file operations
At startup, resolve the working directory to its real path (following symlinks, normalizing Unicode). Store it as the session's identity root. When the agent changes directory (e.g., switches to a worktree), update the operational working directory separately. Never overwrite the identity root.

**Why:** Session storage (history, transcripts, skills) is keyed by the identity root. If the identity root drifts with the working directory, a session started in `/project` that switches to `/project/.git/worktrees/feature` will write its history to the worktree path, then be unable to find it on resume (which starts from `/project`).

**What breaks without it:** Session resume fails silently after any mid-session directory change. The history, transcript, and skill state from the session are written to the worktree path and are effectively orphaned from the perspective of the original project directory.

### 2. Use body-only comparison when matching session IDs across compat layers
When your infrastructure has a compat layer that rewrites session ID prefixes (`session_abc123` → `cse_abc123`), compare session IDs by extracting the body (UUID) rather than comparing the full string.

**Why:** The compat layer may return one prefix to one client tier while the infrastructure uses a different prefix internally. Full-string comparison causes a component to reject its own session as "foreign" — a hard-to-diagnose failure that only manifests when the compat gate is active.

**What breaks without it:** The session management component receives its own session back from the infrastructure with a different prefix and concludes it belongs to a different session, dropping the work or refusing to process it.

### 3. Resolve symlinks and normalize Unicode in the identity root at startup
Use the real path (resolve symlinks) and normalize to a canonical Unicode form (NFC on macOS) when setting the identity root. Handle filesystem permission errors on the real-path resolution with a fallback to the raw path.

**Why:** On macOS, the filesystem uses NFD normalization for stored paths but the process working directory may return NFC. Path comparisons between a path recorded at startup and a path obtained later via `getCwd()` can fail due to Unicode normalization mismatch, not an actual difference in the path.

**What breaks without it:** The agent reports a path mismatch for files it actually has access to, or fails to find its own session file because the recorded path and the lookup path differ in Unicode normalization.

## Anti-patterns

- **What:** Use the current working directory (`getCwd()`) to derive session file paths
  **Why it fails:** The CWD changes with worktree switches, `cd` operations, and remote session initialization; session files end up scattered across different directories
  **Symptom:** Resume fails to find the session file; multiple partial session files for the same logical project; session history not found after worktree switch

- **What:** Compare session IDs across API versions or compat layers by full string equality
  **Why it fails:** Compat layers rewrite ID prefixes while preserving the underlying UUID; full-string comparison sees two representations of the same session as different sessions
  **Symptom:** "Session not found" or "foreign session" errors that only appear when the compat gate is active; intermittent failures tied to infrastructure routing rules

- **What:** Re-resolve the identity root on each turn (e.g., calling `realpathSync` every request)
  **Why it fails:** The real path can change (temporary mounts, CloudStorage, network drives) and some filesystems raise permission errors on `realpathSync`; re-resolution introduces both inconsistency and fragility
  **Symptom:** EPERM errors on CloudStorage mounts; identity root drift on network filesystems; session files created at different paths across turns of the same session

## Structural template

```
// Startup — set identity root once
function initSession() {
  const rawCwd = process.cwd()
  let identityRoot: string
  try {
    identityRoot = realpathSync(rawCwd).normalize('NFC')  // canonical form
  } catch {
    // Handle CloudStorage EPERM, network drive lstat failures, etc.
    identityRoot = rawCwd.normalize('NFC')                // fallback: raw + normalize
  }

  return {
    identityRoot,    // NEVER updated — used for: history, transcripts, session files
    cwd: identityRoot,  // updated by file operations and directory changes
  }
}

// Mid-session directory change (worktree, cd, remote init)
function changeDirectory(session, newPath) {
  session.cwd = newPath  // operational CWD updates
  // session.identityRoot is NEVER touched
}

// Session storage: always use identityRoot
function getSessionFilePath(session) {
  return path.join(session.identityRoot, '.session', session.id + '.jsonl')
}

// Session ID comparison across compat layers
function sameSession(idA: string, idB: string): boolean {
  if (idA === idB) return true
  // Extract body: everything after the last underscore
  const bodyA = idA.slice(idA.lastIndexOf('_') + 1)
  const bodyB = idB.slice(idB.lastIndexOf('_') + 1)
  // Guard: require minimum body length to avoid false matches on short IDs
  return bodyA.length >= 8 && bodyA === bodyB
}
```

## Underlying principle
Session identity and working location serve different masters: identity is stable across the session lifetime and is used for storage and retrieval; location is mutable and is used for file operations. Conflating them means that any feature which changes location (worktrees, remote sessions, subagent spawning in different directories) silently corrupts the storage model. Decoupling them costs one extra field in the session struct and prevents a class of resume failures that are extremely difficult to reproduce and debug.

## Status
`seed-1-team` — promoted from cross-subsystem design analysis (codebase-tagged pattern with generalizable principle). Validate earlier than standard: needs a second team confirmation before treating as established.
