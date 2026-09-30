---
title: "Notification System — Evaluation"
status: complete
date: 2026-09-29
tags: [hld, mock, notification-system, evaluation]
---

# Notification System — Evaluation

## Scoring

| Phase | Score | Why |
| --- | --- | --- |
| Phase 1: Requirements | 9/10 | Split transactional from marketing, got per-category latency requirements that licensed three distinct trigger strategies, and pinned duplicate tolerance per channel rather than globally. |
| Phase 2: Estimate | 10/10 | Computed per-channel peak volumes and then proved the real ceiling is the SMS provider quota at roughly 200x over capacity, which reframed the entire design. |
| Phase 3: High-level design | 9/10 | Per-channel queues justified on four independent grounds, outbox used to make the dual write atomic, and producers fully insulated by the durable bus. |
| Phase 4: Deep dive | 9/10 | Content-derived dedupe keys with a time window, the side-effect-unknown retry class, asymmetric fail-open/fail-closed, and digest reframed as aggregation rather than batching. |
| Phase 5: Trade-offs and follow-ups | 8/10 | All "why not" questions answered with named costs; strongest on the observation that throughput instincts are wrong when the hard limit is a vendor contract. |

**Overall: 9.0/10** — Strong hire signal. The distinguishing behavior is that this candidate
sized the *external* constraint before sizing the compute, which is rare and is the correct order
for this problem.

## What Made This a Strong Answer

- **The capacity table was the design.** Showing SMS at ~200x over provider capacity in Phase 2
  is what forced admission control, channel fallback, and the honest statement that SMS throughput
  is a product-cost decision, not a scaling decision.
- **Fanout was correctly argued as structurally different from the chat case.** There is no pull in
  notifications, so read-time deferral does not apply, and the answer became a shared campaign row
  plus per-user dedupe as the work unit, paced by the provider quota instead of a 5 GB burst.
- **The side-effect-unknown error class was named and resolved correctly.** A timeout is an unknown,
  not a failure, and it must be retried with the *same* provider key. This is the actual source of
  duplicate SMS in production and most candidates never reach it.
- **Dedupe was defended as layered, not absolute.** Redis `SET NX` is called a strong optimization
  and not a guarantee, with the store's unique index on `idempotency_key` as the real boundary and
  the provider key as the layer nearest the side effect.
- **Failure modes were made asymmetric on purpose.** Fail open for push, fail closed for SMS,
  because a duplicate push is cheap and an unverified SMS is a charge plus a complaint.
- **Digest was reframed as product design.** Collapsing 100 likes into "12 people liked your photo"
  is better for the user, cheaper, and less annoying simultaneously, which is a stronger argument
  than any batching window.

## Memory Hooks

- 8,700 notifications/sec average, ~150,000/sec peak, 17x average because spikes are externally
  triggered and unsmoothable.
- 210,000 sends/sec across channels, split 55 push / 35 email / 10 SMS.
- SMS is ~200x over provider capacity. The hard limit is a vendor contract, never local compute.
- Templates 200 MB, preferences 15 GB, log 137 TB/year. Design effort follows bytes and risk.
- Dedupe key = H(user, category, channel, entity, kind, time_window). NX at enqueue, not at send.
- Timeout is an unknown, not a failure. Retry with the same key, never a fresh one.
- Fail open for push, fail closed for SMS.
- Outbox converts silent permanent loss into possible duplicate delivery. Prefer that trade.
- Quiet hours are local wall-clock plus IANA timezone, not a fixed UTC offset.
- Digest is aggregation, not a batching window. 1-minute cron cannot meet a 5-second SLO.

## Weak-Spot Pointers

- **The channel-fallback priority table was asserted without a worked example.** Pick one
  notification class, show the full decision trace through a provider outage, and price it. Drill
  [[circuit-breaker]] and [[load-shedding]] so the fallback ladder has a canonical form.
- **Weighted fair queueing was named in one sentence and never explained.** If you cannot state the
  algorithm and its two guarantees, do not invoke it. Drill [[tenancy-and-cells]] and
  [[distributed-rate-limiter]].
- **Per-provider throughput numbers were asserted as a table, not derived.** Interviewers push on
  invented numbers. Have a defensible way to say "these are order-of-magnitude and the design
  handles being wrong by two orders of magnitude". Drill [[bottleneck-identification]] and
  [[capacity-estimation]].
- **APNs and FCM specifics were hand-waved with a "Study separately".** The token-lifecycle
  mutation on `UNREGISTERED` and the APNs unsubscribe timestamp are exactly the details that
  distinguish a prepared candidate. There is currently no `push-notifications` note in the vault,
  so write one.
- **Nobody asked what happens when the notification bus itself is unavailable.** The transcript
  treats Kafka as a solved dependency. Have an answer: producers buffer locally, the bus is
  monitored by consumer lag, and the guarantee degrades from seconds to minutes. Drill
  [[consumer-lag]] and [[replayability]].

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the phase structure to reuse for the
  next problem
- [[01-rapid-revision|Rapid Revision]] — one-page refresh before a real interview
- [[delivery-and-retry]] — the delivery state machine and provider webhook feedback loop
- [[outbox-pattern]] — making the database-plus-broker dual write atomic
- [[distributed-rate-limiter]] — token buckets as the mechanism for external quota enforcement
