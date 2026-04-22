---
name: shared-resource-contention-in-concurrent-agent-execution
description: "Prevents race conditions and security holes when agent tasks execute in parallel over shared credentials, network, and filesystem. TRIGGER when: building agents that execute multiple tool calls or tasks concurrently, agents share OAuth credentials or API keys across parallel executions, agent code execution runs on shared multi-tenant infrastructure, agents need both temporary scratch space and persistent storage across turns, designing workspace isolation for agent sessions."
---

## Key Decisions

1. **Acquire an exclusive lock on each credential set for the duration of a tool execution, not just for the API call itself.** OAuth tokens may need mid-execution refresh, and two parallel tasks refreshing the same token simultaneously will invalidate each other's results. The lock must cover the full execute-maybe-refresh-retry cycle. The trade-off is reduced parallelism when multiple tasks share credentials — accept this or provision per-task credential clones.

2. **Block the agent's default code execution tool and replace it with a network-isolated variant using kernel-level isolation, rather than relying on application-level network restrictions.** Application-level URL allowlists can be bypassed through DNS rebinding, redirect chains, or IP-literal URLs. Kernel-level network namespace isolation (removing the network interface entirely from the execution context) is the only reliable prevention for LLM-generated code making arbitrary outbound requests. This is critical in multi-tenant environments where a prompt injection in one session could reach another tenant's services.

3. **Maintain two explicit storage tiers — an ephemeral working directory and a persistent workspace — and document the distinction in the agent's system prompt as a first-class instruction.** LLMs cannot infer storage tier semantics from filesystem paths alone. Without explicit instruction, the agent will save important outputs to the ephemeral directory (lost between turns) or treat the persistent workspace as scratch space (accumulating garbage). The system prompt must state: which paths are ephemeral and when they are cleared, which paths persist and for how long, and the explicit instruction to save important files to the persistent tier.

## Anti-patterns

- **What**: Relying on credit balance or billing systems as the sole concurrency throttle for agent execution, with no per-user execution rate limit.
  **Why**: A user with high credit balance (or during a free trial) can spawn enough concurrent executions to exhaust the worker pool, starving all other users. A compromised API key has the same effect.
  **Symptom**: Execution latency spikes to minutes across all users while one user's dashboard shows dozens of concurrent runs; no rate-limit errors are returned because the billing system sees sufficient balance.

- **What**: No cross-session episodic memory — each agent session starts with zero context about previous sessions beyond static profile fields.
  **Why**: Users must re-explain context, preferences, and constraints every session. The agent cannot learn which tool sequences succeeded or failed in past sessions for similar tasks.
  **Symptom**: Users repeatedly paste the same instructions at session start; support tickets say "the agent forgot everything we discussed yesterday"; successful patterns are never reused automatically.

- **What**: Sharing a mutable execution context object across concurrent agent tasks without copying.
  **Why**: When parallel tasks write to the same context (updating current task ID, accumulating results, tracking execution state), they corrupt each other's state non-deterministically.
  **Symptom**: Intermittent wrong-task-ID in logs, execution results attributed to the wrong task, occasional null pointer errors that cannot be reproduced — all symptoms that appear under load and disappear in single-task testing.

## Structural Template

```
// Credential locking layer
function execute_with_credentials(task, credential_set):
    lock = acquire_exclusive_lock(credential_set.id, timeout=EXECUTION_TIMEOUT)
    try:
        result = execute_task(task)
        if result.needs_token_refresh:
            refresh_token(credential_set)  // safe — we hold the lock
            result = retry_task(task)
        return result
    finally:
        release_lock(lock)

// Network-isolated code execution
function create_sandboxed_executor(session):
    sandbox = create_execution_context(
        network: ISOLATED,          // kernel-level: no network interface
        filesystem: session.ephemeral_dir,
        timeout: MAX_EXECUTION_TIME
    )
    return sandbox

function execute_code_tool(code, session):
    sandbox = create_sandboxed_executor(session)
    return sandbox.run(code)  // no network access possible

// Dual storage tier with LLM education
function initialize_session(session_id):
    ephemeral_dir = create_temp_directory(session_id)
    persistent_dir = resolve_workspace(session_id)

    storage_instruction = """
    FILE STORAGE RULES:
    - {ephemeral_dir}: Scratch space. Cleared after each turn/session.
      Use for: intermediate files, temp downloads, working copies.
    - {persistent_dir}: Permanent workspace. Survives across sessions.
      Use for: final outputs, user-requested files, important results.
    Always save important files to {persistent_dir}.
    """
    inject_into_system_prompt(storage_instruction)

    return Session(ephemeral_dir, persistent_dir)

// Per-task context isolation
function execute_parallel_tasks(tasks, shared_context):
    results = parallel_map(tasks, lambda task:
        task_context = deep_copy(shared_context, overrides={
            task_id: task.id,
            execution_id: generate_id()
        })
        return execute_task(task, task_context)
    )
    return merge_results(results)
```
