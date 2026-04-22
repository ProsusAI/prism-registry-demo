---
name: anthropic-api-thinking-blocks
description: "Anthropic API extended thinking: how thinking blocks must be preserved and
  when they cause 400 errors. TRIGGER when: implementing extended thinking
  in a multi-turn agentic loop, when debugging 400 errors with 'thinking blocks
  cannot be modified', when implementing model fallback that switches from a
  thinking-enabled model, or when deciding what to strip before a retry.
  Load when writing any code that handles extended thinking across multiple turns."
---

# Anthropic API: Extended Thinking Block Invariants
*What to expect when using extended thinking in multi-turn agentic loops.*

## Who this is for
Teams using extended thinking (`max_thinking_length > 0`) in multi-turn conversations with tool use. These constraints are not prominently documented and produce confusing 400 errors.

## Design insight 1: Thinking blocks must be preserved through the full assistant trajectory

**Behavior:** The API enforces a 3-part invariant for thinking blocks:
1. A message containing a `thinking` or `redacted_thinking` block must be part of a request where `max_thinking_length > 0`
2. A thinking block may not be the last block in a message
3. Thinking blocks must be preserved for the entire assistant trajectory — this means: the assistant message that contains the thinking block, **plus** the `tool_result` user messages that follow, **plus** the next assistant message that follows those tool_results

**Why it's non-obvious:** Developers naturally think of thinking blocks as belonging to the single message that contains them. The trajectory constraint extends preservation through the tool_result/assistant exchange that follows — an entirely different message in the conversation. Stripping thinking from "old" assistant messages during context management, when those messages are within an active trajectory, triggers a 400.

**What callers must know:** (1) Never strip thinking blocks from assistant messages that are part of an unfinished trajectory. (2) On model fallback (switching from a thinking-enabled model to one that doesn't support thinking), strip all thinking blocks from history before the retry — the fallback model's request will have `max_thinking_length = 0`. (3) On model fallback mid-stream, thinking blocks in already-emitted partial messages have invalid signatures — emit tombstones for those messages and retry cleanly.

**Evidence:** AI coding assistant CLI — March 2026

## The underlying pattern
Extended thinking is a trajectory-level feature, not a message-level feature. The API treats the assistant message, its tool calls, the tool results, and the following assistant response as a single atomic unit from a thinking-block perspective. Any context management (compaction, truncation, fallback) must preserve or remove thinking blocks at trajectory granularity, not at message granularity.

## Status
`seed-1-team` — promoted from static code analysis (design seed). Validate earlier than standard: needs a second team confirmation and ideally an incident cluster before treating as established.
