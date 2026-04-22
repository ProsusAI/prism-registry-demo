---
name: anthropic-api-tool-dispatch
description: "Anthropic API tool use: detecting when the model has called a tool and
  dispatching tool execution correctly. TRIGGER when: implementing tool
  dispatch in a streaming API loop, when debugging tool calls that are silently
  missed, or when a team member asks why stop_reason should not be used as the
  tool-dispatch signal. Load when writing any code that processes streaming
  responses for tool use."
---

# Anthropic API: Tool Dispatch
*What to expect when implementing tool use in a streaming Anthropic API loop.*

## Who this is for
Teams implementing tool dispatch on top of the Anthropic streaming API. This failure mode appears early — often in the first integration test — but is subtle because `stop_reason` is present in the API response and appears to be the intended signal.

## Design insight 1: Use content-block detection, not stop_reason, for tool dispatch

**Behavior:** The API's `stop_reason === 'tool_use'` field is not reliably set when the model calls a tool. A streaming response may contain `tool_use` content blocks but have a `stop_reason` of `end_turn` or `null`.

**Why it's non-obvious:** The API documentation describes `stop_reason === 'tool_use'` as the signal that the model wants to use a tool. The natural implementation is to check `stop_reason` at the end of streaming and dispatch if it equals `'tool_use'`. This works most of the time, creating test suites that pass — then fails in production on specific model responses.

**What callers must know:** Detect tool use by inspecting content blocks for `type === 'tool_use'` entries as they arrive during streaming. Set a `needsFollowUp` flag when any such block is seen. Do not use `stop_reason` as the primary or secondary signal for tool dispatch.

**Evidence:** AI coding assistant CLI — March 2026

## The underlying pattern
The API's `stop_reason` is a best-effort hint, not a guarantee. For anything that controls execution flow (tool dispatch, continuation decisions), inspect the message content directly rather than relying on the stop signal. Content is authoritative; `stop_reason` is informational.

## Status
`seed-1-team` — promoted from static code analysis (design seed). Validate earlier than standard: needs a second team confirmation and ideally an incident cluster before treating as established.
