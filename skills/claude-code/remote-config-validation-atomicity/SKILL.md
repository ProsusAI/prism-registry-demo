---
name: remote-config-validation-atomicity
description: "Remote config validation for interdependent fields: why to reject the entire
  config object rather than clamping individual fields. TRIGGER when
  designing validation for remotely-tunable configuration objects with fields
  that constrain each other, when choosing between whole-object rejection vs.
  per-field clamping in a validation schema, or when debugging a service that
  behaved unexpectedly after a remote config push. Load when writing validation
  for any config that is served remotely and has cross-field constraints."
---

# Remote Config: Atomic Validation for Interdependent Fields
*Why to reject the whole config object rather than clamping invalid individual fields.*

## Who this is for
Teams serving configuration remotely (feature flags, GrowthBook, LaunchDarkly, config service) where the config object has interdependent fields — fields that must satisfy constraints relative to each other, not just individually.

## Design insight 1: Reject the whole config object on any field violation; fall back to safe defaults

**Behavior:** When validating a remotely-served config object, if any field fails validation, reject the entire object and fall back to hardcoded safe defaults — rather than accepting the valid fields and clamping or defaulting the invalid one.

**Why it's non-obvious:** Per-field clamping feels safe: "if the poll interval is invalid, clamp it to 100ms." But interdependent fields can produce dangerous combinations even when each individual field appears valid after clamping. A config with one bad field (e.g., an operator enters `10` thinking it means 10 seconds, but the field is in milliseconds) is a signal that the operator made a mistake — partial trust of the rest of the config propagates that mistake to production.

**What callers must know:** (1) Use an all-or-nothing validator (e.g., Zod `safeParse` that returns `parsed.success ? parsed.data : DEFAULTS`). (2) Add cross-field refinements as schema-level constraints (e.g., "at least one of fieldA or fieldB must be nonzero"). (3) For fields where 0 means "disabled" (not "zero"), use a refinement that rejects values 1–99 — these are almost always unit confusion (operator entered 10 meaning seconds, not milliseconds).

**Evidence:** AI coding assistant CLI — March 2026

## The underlying pattern
Remote config serves as a trust delegation: the code trusts the config object to be internally consistent. Partial validation breaks this model: it accepts a config that the operator clearly intended to be different, and applies a part of it in combination with defaults that may not be compatible. The safest posture is: if the config is wrong, don't use any of it.

## Status
`seed-1-team` — promoted from static code analysis (design seed). Validate earlier than standard: needs a second team confirmation and ideally an incident cluster before treating as established.
