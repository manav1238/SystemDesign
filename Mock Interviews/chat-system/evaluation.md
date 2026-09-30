---
title: "Chat System (WhatsApp) — Evaluation"
status: complete
date: 2026-09-29
tags: [hld, mock, chat-system, evaluation]
---

# Chat System (WhatsApp) — Evaluation

## Scoring

| Phase | Score | Why |
| --- | --- | --- |
| Phase 1: Requirements | 9/10 | Resolved ordering scope, fanout requirement, and retention before designing, and correctly refused per-message read receipts by proving the monotonicity argument. |
| Phase 2: Estimate | 10/10 | Distinguished sends (700K/s) from deliveries (2.1M/s), identified the 6x fanout amplification as the real driver, and sized 62.5M concurrent connections for the gateway tier. |
| Phase 3: High-level design | 9/10 | Clean tier separation with a stateless gateway, and correctly placed the queue downstream of durability so send acknowledgement never depends on the broker. |
| Phase 4: Deep dive | 9/10 | Sequence-number ordering that derives from partitioning, a three-layer backpressure ladder, and a per-field staleness budget that makes consistency choices a lookup rather than a debate. |
| Phase 5: Trade-offs and follow-ups | 9/10 | Answered every "why not" with the accepted cost stated explicitly, and led cost reduction with feature removal by value-per-byte rather than micro-optimization. |

**Overall: 9.2/10** — Strong hire signal. The distinguishing behavior is that this candidate
derives design decisions from the arithmetic rather than recalling patterns.

## What Made This a Strong Answer

- **The fanout insight came before the architecture.** Separating 700K sends/sec from 2.1M
  deliveries/sec is what makes the rest of the design fall out, and it was done in Phase 2 rather
  than discovered as a surprise in Phase 4.
- **Read receipts were compressed by proof, not by assertion.** The monotonicity argument (you
  cannot read message 47 without having read 1-46) removed roughly 2M writes/sec and is the single
  highest-leverage decision in the system.
- **Ephemeral signals were given the right reliability level.** Typing indicators and presence are
  never persisted, never retried, and served from a 3s local cache, with the reasoning that
  reliability must match the half-life of usefulness. Retrying a decaying signal delivers a
  statement that is already false.
- **Failover was answered with the data structure, not a slogan.** The claim that the store's
  uniqueness constraint is the only real dedupe guarantee, and that layers 1 and 2 are
  optimizations, is the correct and under-taught position.
- **Every trade-off named its cost.** WebSocket means owning reconnection and backpressure; Kafka
  means owning rebalancing; read-time fanout means accepting a scatter-gather read. An answer
  without a stated cost is not a trade-off, and this transcript had a cost on every "why not".
- **Hot keys were analyzed in all three places they break**, including the Redis key-space skew
  that usually gets missed, and it correctly rejected shard jitter because it would destroy the
  ordering guarantee the design depends on.

## Memory Hooks

- 700K sends/sec, 2.1M deliveries/sec — the 6x fanout multiplier is the whole design driver.
- Per-conversation ordering falls out of conversation-id sharding; no lock, no cross-shard coord.
- Sequence number orders, timestamp displays. Client clocks are untrusted.
- Read receipts are monotonic, so one integer per user per conversation replaces 2.1M writes/sec.
- Presence: 40s TTL heartbeat, 3s local cache, never persisted, last-seen is a different animal.
- Typing and presence are lossy on purpose. Half-life of usefulness sets the reliability level.
- Three-layer backpressure: buffer cap, pause TCP, then shed ephemeral then close the socket.
- Backpressure on a chat system is a resource leak if a slow consumer is allowed to block a loop.
- 250 bytes/msg x 22 trillion/yr = ~5.5 PB/yr, and every replica is a 5 PB copy.
- Store once, fan out wide. Reject the generic hot-key jitter fix when the key carries ordering.

## Weak-Spot Pointers

- **Cross-shard and cross-device sync deserves more depth.** The interview answered the single-device
  catch-up path well, but the hard case, device A read to 500 while device B advanced to 900, was
  handled as a pointer-authority argument rather than a merge rule. Drill
  [[replication-lag]] together with [[event-sourcing-cqrs]] and think about what "read watermark"
  should be keyed on.
- **The cost of read-time fanout on login was asserted, not calculated.** The N+1 across hundreds of
  unread conversations deserves a real number: shards touched, connection pool pressure, and p99
  tail. Drill [[cross-shard-queries]] and [[connection-pooling]].
- **Per-user Redis state was sharded by user id in the mitigation, but the initial design never
  justified choosing that key.** If that is not the natural default, say why out loud. Drill
  [[shard-key]] and [[hotspot-handling]].
- **E2E encryption was waved off in one paragraph.** It removes server-side search, server-side
  moderation, and link previews, which are three real architectural consequences. Study separately,
  and drill [[encryption-and-keys]].
- **No numbers were given for the 500,000-member group fanout in the read path.** You quoted 1.5
  billion write-time rows per day but not the read-time cost you accepted in exchange. Always price
  the option you reject. Drill [[trade-off-analysis]].

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the phase structure to reuse verbatim
  for the next problem
- [[01-rapid-revision|Rapid Revision]] — one-page refresh before a real interview
- [[delivery-semantics|Delivery Semantics]] — the backbone of the queue and idempotency argument
- [[websockets]] — connection lifecycle, heartbeats, and reconnection at 62M connections
- [[backpressure]] — the slow-consumer escalation ladder in general form
