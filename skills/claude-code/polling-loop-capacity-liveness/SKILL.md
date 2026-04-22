---
name: polling-loop-capacity-liveness
description: "Polling loops with capacity throttling how to prevent tight-loop failures
  when a polling service enters an at-capacity state. TRIGGER when:
  designing a polling loop that has a 'throttled' or 'at capacity' mode, when
  configuring sleep intervals for capacity-throttled polling, or when debugging
  a service that appears to be looping at HTTP-round-trip speed. Load when
  writing any poll loop that can enter a reduced-activity mode."
---

# Polling Loop: At-Capacity Liveness Invariant
*How to prevent tight-loop failures when a polling service enters a throttled mode.*

## Who this is for
Teams building polling services (job queue workers, bridge polling loops, long-polling consumers) that have a distinct "at capacity" or "throttled" mode where normal polling is reduced. The tight-loop failure is non-obvious because it requires two independent configuration mistakes to occur simultaneously.

## Design insight 1: When a polling loop is at capacity, at least one sleep mechanism must remain active

**Behavior:** A polling loop in at-capacity mode typically has two independent mechanisms that provide backpressure: (1) a reduced poll interval (sleep between polls), and (2) a heartbeat (periodic keep-alive with a sleep). If both are disabled simultaneously — poll interval set to 0 AND heartbeat disabled — the loop exits the capacity throttle code with no sleep and polls at HTTP round-trip speed.

**Why it's non-obvious:** Each mechanism appears to be independently optional: "I'll disable heartbeat since I'm using poll intervals" or "I'll disable poll intervals since I'm using heartbeats." The failure only occurs when both are disabled at the same time, which requires two separate misconfigurations or an operator disabling one without knowing the other was also off. The resulting tight loop is expensive (continuous HTTP requests) and can cause DB path overload.

**What callers must know:** Enforce a schema-level invariant that at least one at-capacity sleep mechanism is active. Reject configs where both are disabled (poll interval = 0 AND heartbeat interval = 0) rather than accepting them silently. For the poll interval field specifically, use `0` to mean "disabled" and reject values 1–99 (values like `10` are likely a unit confusion — the operator meant 10 seconds, not 10ms).

Also: the sleep detection threshold (used to detect system sleep/wake) must be set above the maximum backoff cap. If it equals the cap, a normal max-backoff sleep registers as a sleep event and resets the error budget, masking sustained connection failures indefinitely.

**Evidence:** AI coding assistant CLI — March 2026

## The underlying pattern
Independent sleep mechanisms interact: disabling all of them simultaneously produces a failure mode that no single mechanism's documentation would warn about. The fix is an invariant enforced at config validation time, not at runtime — by the time the loop is running at HTTP-round-trip speed, the damage is already happening.

## Status
`seed-1-team` — promoted from static code analysis (design seed). Validate earlier than standard: needs a second team confirmation and ideally an incident cluster before treating as established.
