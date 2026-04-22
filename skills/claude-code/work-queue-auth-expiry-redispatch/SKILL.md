---
name: work-queue-auth-expiry-redispatch
description: "Work queues with short-lived auth credentials: why token expiry requires re-dispatching the claimed work item, not just refreshing the token. TRIGGER when building a work queue (Redis Streams, SQS, Kafka, RabbitMQ) where consumers authenticate with short-lived credentials (JWTs, OAuth tokens, signed claims), when debugging a consumer that appears healthy but receives no new work, or when designing token refresh for long-running queue consumers. Load when combining any ACK/claim-based queue with short-lived auth."
---

# Work Queue + Short-Lived Credentials: Re-dispatch on Auth Expiry
*What to expect when short-lived auth credentials expire while a consumer holds a claimed work item.*

## Who this is for
Teams building work queues where consumers authenticate with short-lived credentials (JWTs, OAuth tokens, signed claims) and the queue uses an ACK/claim model (Redis Streams PEL, SQS visibility timeout, Kafka consumer group offsets, RabbitMQ unacked messages). This failure mode is silent — the consumer appears healthy and processes heartbeats, but receives no new work.

## Design insight 1: Auth credential expiry requires re-dispatch of the claimed work item, not just credential refresh

**Behavior:** When a consumer's credentials expire mid-processing, the consumer must not only obtain new credentials but also trigger server-side re-dispatch of the work item. Without re-dispatch, the item remains in the queue's claimed state — acknowledged/claimed but not confirmed as complete — and the consumer's poll loop returns empty forever.

**Why it's non-obvious:** The natural mental model of credential refresh is: get new credentials, resume work. Queue ACK/claim semantics break this: an item that has been claimed (ACK'd, visibility-timeout extended, offset committed) is removed from the delivery stream and lives in a "claimed" state until the consumer explicitly completes or abandons it. Credential expiry causes an auth failure on the completion call, not on the claim — so the item appears to still be in the consumer's possession from the queue's perspective, even though the consumer can no longer authenticate to report completion.

**What callers must know:** When credentials expire during work item processing, trigger the queue system's re-dispatch mechanism (Redis `XCLAIM`, SQS `ChangeMessageVisibility` timeout reset, Kafka consumer group rebalance, RabbitMQ `basic.nack`) to move the item back to the unprocessed delivery stream. Only after re-dispatch will a subsequent poll return new work. Proactive credential refresh (refreshing before expiry, not after) avoids the problem entirely and is strongly preferred.

## The underlying pattern
Work queue ACK/claim state and auth credential state are independent systems. A work item can be in a "claimed" state from the queue's perspective while the claimer's auth credentials have expired. Any system that combines short-lived credentials with a queue's ACK/claim model must handle the intersection: credential expiry should trigger re-dispatch, not just credential refresh.

## Status
`seed-1-team` — promoted from static code analysis (design seed). Validate earlier than standard: needs a second team confirmation and ideally an incident cluster before treating as established.

*Evidence from: Redis Streams + JWT (AI coding assistant CLI, March 2026)*
