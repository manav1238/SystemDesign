---
title: WhatsApp - Interview Evaluation
status: active
date: 2026-09-29
tags: [hld, mock, whatsapp, evaluation]
---

# WhatsApp - Mock Interview Evaluation

## Scoring

| Phase | Score | Why |
| --- | --- | --- |
| Requirements and Constraints | 5 | Refused exactly-once before being asked, split the ordering question into within-conversation versus across-conversation, and listed the three things end-to-end encryption makes impossible as a design consequence rather than an implementation detail. |
| Scale Estimation | 5 | Reached the insight that decides the architecture: 45 messages per second per node against 20,000 connections, so the fleet is sized by memory and file descriptors, not throughput. Also estimated deliveries separately from writes, which is where group fanout actually shows up. |
| API and Data Design | 4 | The message row with server sequence plus client ULID plus a unique constraint is exactly right, and per-device delivery state instead of a conversation boolean is the detail most candidates miss. Lost a point for describing both per-row receipts and high-water marks without committing to the high-water mark design. |
| Architecture and Data Flow | 5 | Gateway pool with consistent-hash affinity, sequencer colocated with the message shard, media bypassing the gateway, and push-as-pointer instead of push-as-payload, each with a stated reason and a stated cost. |
| Deep Dive and Failure | 4 | Fencing on failover, RPO zero justified from the quorum choice, disconnect-and-resync as a deliberate backpressure strategy, and a worked tail-risk calculation on large groups. Lost a point for not quantifying the reconnect storm or the push provider ceiling. |

Total: 23 / 25

## What Made This a Strong Answer

- **Refused the impossible requirement up front.** Saying "exactly-once end to
  end is not achievable and here is the contract I can actually give you" is the
  single strongest opening move in a messaging design, and pairing it with
  exactly-once *effect* turned a rejection into a design.
- **Found the binding constraint that is invisible to a request-per-second
  estimate.** Forty-five messages per second per node with twenty thousand idle
  connections is the number that turns a gateway fleet from "thirty nodes" into
  "thirty-nine thousand nodes", and it also explains which limits actually bind:
  file descriptors, ephemeral ports, and socket memory, not application
  throughput.
- **Every large design had a cost stated next to it.** Pointer pushes, sticky
  gateway affinity, disconnect-and-resync backpressure, and quorum writes were
  each presented as a purchase with a named price, which is what makes an
  architecture defensible rather than aspirational.
- **Distinguished "late" from "lost" in every failure path.** A skipped push, a
  late history read, and a stale receipt were each given a specific fix and each
  was correctly identified as a non-data-loss event, which is the discipline that
  keeps an on-call engineer from being woken up unnecessarily.
- **Found the tail, not the average.** Average group size four is easy; the
  thousand-member group was turned into a concrete delivery-rate calculation
  that showed the storage tier would die before the gateway tier, and the 256
  threshold follows from that arithmetic rather than from intuition.
- **Named its own security weaknesses.** The deterministic pair conversation id
  leaking relationship structure to a malicious operator, and server-side
  metadata exposure from per-conversation delivery patterns, were volunteered
  rather than hidden.

## Memory Hooks

- Gateway fleet is sized by connections and memory, not by message rate.
  Forty-five messages a second per node, twenty thousand connections.
- Two-of-three quorum write is why RPO is zero, and it costs five milliseconds
  on every message in the system.
- Server sequence orders the conversation, client ULID deduplicates it. Never
  let the server generate the id.
- Range allocation from the sequencer turns one round trip per message into one
  per sixty-four. Unused numbers become gaps, and gaps need a hole registry.
- Push a pointer, never a payload. A stale body is a correctness bug; an extra
  round trip on an open connection is not.
- Disconnect and resync beats buffering. Seven hundred million connections times
  an unbounded queue is a fleet-wide OOM, and the store is authoritative anyway.
- Groups above 256 members stop materializing deliveries entirely and switch to
  a high-water mark plus pull.
- Offline is not a special case. It is a sync that starts later.

## Weak-Spot Pointers

- **The cryptography was named but not designed.** "X3DH plus the double
  ratchet" is correct as vocabulary and thin as a design. Prekey lifecycle,
  skipped-key storage, device unlink semantics, and the group sender-key
  rotation event were all asserted. Drill [[encryption-and-keys|Encryption and
  Keys]] and be able to describe a session break and its user-visible cost
  without hedging.
- **The two-tier liveness model had no numbers behind it.** A 25-second ping and
  a 15-minute backoff were plausible but unexplained. Drill
  [[heartbeat-health-checks|Heartbeat and Health Checks]] so you can justify
  ping frequency from battery cost and from the reconnect-storm risk you create
  when you ping too aggressively.
- **Range allocation was explained but its interaction with fairness was
  ignored.** A user who allocates sixty-four sequences and sends two messages
  holds the rest, and a hot conversation has an obvious starvation and
  head-of-line-blocking story. Drill [[kafka-ordering|Partition Ordering]] and
  [[consumer-lag|Consumer Lag]], and be able to say what happens when a single
  conversation is twenty times the median rate.
- **Availability discussion was asymmetric.** Failover and gateway loss were
  treated well, but there was no treatment of a metadata partition or of the
  session directory going down, which is a single Redis cluster sitting under
  every routing decision. Drill [[caching|Caching]] and
  [[single-point-of-failure|Single Point of Failure]] and work out what happens
  when routing degrades to "unknown" and whether the correct behaviour is
  broadcast or refuse.
- **No adversarial or abuse section.** A messaging platform's hardest
  availability threat is not hardware failure, it is a scripted abuse or spam
  flood against push and delivery. Drill
  [[adversarial-reliability|Adversarial Reliability]] and
  [[rate-limiter|Rate Limiter]] so that abuse resistance is part of the design
  rather than a follow-up conversation.

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] - check the answer
  against the five phases, and confirm the failure and trade-off sections were
  genuinely covered rather than summarised
- [[01-rapid-revision|Rapid Revision]] - the delivery semantics ladder and the
  "stateless gateway, durable store" split are the two things worth having in
  cold memory
