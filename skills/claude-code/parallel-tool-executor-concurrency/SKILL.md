---
name: parallel-tool-executor-concurrency
description: "Parallel tool executors: how to safely batch concurrent tool calls while
  preserving deterministic context mutation. TRIGGER when: designing a
  tool executor that runs multiple tool calls concurrently, when debugging
  non-deterministic context state after concurrent tool execution, when
  deciding whether a tool call is safe to run concurrently, or when a write
  tool appears in a batch with read tools. Load when writing any code that
  executes multiple tool calls in parallel."
---

# Parallel Tool Executor: Concurrency and Context Safety
*How to run tool calls concurrently without producing non-deterministic state.*

## Who this is for
Teams building tool executors for agentic systems where multiple tool calls may be dispatched in a single model response. These decisions are non-obvious because the failure (non-deterministic context mutation) manifests as flaky behavior, not a hard error.

## Design insight 1: Write tools break concurrent batches — treat any write as a batch boundary

**Behavior:** Tool calls from a single model response should be partitioned into batches. Consecutive read-only (concurrency-safe) tool calls may run in parallel within a batch. Any write tool (one that modifies shared state) breaks the batch: it and all subsequent tools in the response run serially.

**Why it's non-obvious:** It is tempting to run all tool calls concurrently for performance. The failure is subtle: a write tool that runs concurrently with reads that depend on its output produces correct results most of the time (when the write happens to complete first) and incorrect results occasionally (when a read completes before the write). This presents as intermittent test failures.

**What callers must know:** (1) Determine concurrency safety per-tool, not per-call. A tool is concurrency-safe if it only reads shared state; a tool is not concurrency-safe if it modifies any shared state (filesystem, config, session state). (2) If `isConcurrencySafe` throws or fails to evaluate, treat the tool as a write (conservative default). (3) A write tool in the middle of a batch splits the batch: tools before it may run concurrently, the write tool runs serially, tools after it also run serially.

## Design insight 2: Apply context modifiers in original tool order, not completion order

**Behavior:** When concurrent tools complete, their context modifiers (side-effects on session/permission state) must be queued and applied in the original invocation order, not the completion order.

**Why it's non-obvious:** Concurrent execution produces non-deterministic completion order. A context modifier applied in completion order means that tool A's permission grant might be applied after tool B has already run with the old permissions — or before, depending on network/CPU timing.

**What callers must know:** Queue context modifiers as they complete, keyed by tool invocation ID. After all concurrent tools in a batch complete, apply the modifiers in original invocation order. This adds one collection step but ensures deterministic state.

**Evidence:** AI coding assistant CLI — March 2026

## The underlying pattern
Concurrency safety and context mutation are orthogonal concerns that must both be addressed. A tool can be safe to run concurrently (read-only) but still produce a context modifier that must be applied in order. Treating concurrency safety as a binary "run this tool in the parallel pool" decision is correct for the execution model but not for the context mutation model.

## Status
`seed-1-team` — promoted from static code analysis (design seed). Validate earlier than standard: needs a second team confirmation and ideally an incident cluster before treating as established.
