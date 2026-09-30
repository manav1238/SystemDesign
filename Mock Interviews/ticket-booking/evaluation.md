---
title: Ticket Booking System Design - Evaluation
status: active
date: 2026-09-29
tags: [hld, mock, ticket-booking, evaluation]
---

# Ticket Booking System Design — Evaluation

Session: 45 minutes of design conversation. Interviewer: Principal Engineer, Marketplace (live events).
Scale assumption used: 1M concurrent users at on-sale, 100,000 units, 600,000 booking attempts in 120s, 20,000 events/year, 10-minute hold.

## Scoring

| Phase | Score | Why |
|---|---|---|
| Phase 1 — Requirements | 4.5 / 5 | Opened by forcing the demand-to-supply ratio (6 attempts per unit) and explicitly demanded a zero oversell tolerance from the business before designing anything. |
| Phase 2 — Estimate | 5 / 5 | Separated average day from on-sale, showed the arithmetic both ways, and proved the database is not the throughput bottleneck (40 connections at 8ms) while contention is. |
| Phase 3 — High-level design | 4 / 5 | Waiting room, reaper, and inventory cluster are all correctly identified as the load-bearing pieces, though the search/catalogue split was introduced late. |
| Phase 4 — Deep dive | 5 / 5 | The reaper-versus-confirm race was resolved correctly with mutually exclusive compare-and-set predicates, and the "hold is a state, not a lock" framing was the strongest single idea in the session. |
| Phase 5 — Trade-offs | 4.5 / 5 | Pessimistic versus optimistic was answered with a contention argument rather than a preference, and six accepted trade-offs were each stated with their cost. |
| **Overall** | **4.5 / 5** | Strong hire signal for Senior, with the reaper race analysis alone sufficient to clear a Staff bar. |

## What Made This a Strong Answer

- **Reframing throughput as contention.** Calculating 5,000 attempts per second against 8 ms of transaction time to get "40 connections, this is not a throughput problem" was the turning point. The candidate then recognised that 500,000 failures per sale is more work than 100,000 successes, and made shed-response cost a first-class design requirement.
- **The storage calculation that decided the caching layer.** 100,000 seats as a packed bitmap is 12.5 KB, so the entire on-sale catalogue is 62.5 MB. That arithmetic justified caching per-seat availability everywhere, rather than hand-waving "we will cache availability".
- **The reaper race, resolved with mutually exclusive predicates.** Confirm requires `hold_expires_at > now()`, the reaper requires `hold_expires_at <= now()`. At most one can match, the row lock serialises them, and the loser updates zero rows and refunds. This is the correct answer and it is rare.
- **"The hold is a state, not a lock."** Converting a ten-minute user-facing timer into an 8-millisecond row-locked transaction plus a durable row state is the most transferable idea in the session, and it also cleanly answered the "long hold pins the row" objection.
- **Refusing to cache the deciding path.** Redis bitmap renders, database row lock decides. The candidate named this as the most common way a "cached" ticket system oversells, and kept Redis entirely off the booking critical path so a Redis failure degrades display without touching correctness.
- **The database unique constraint as the floor.** `UNIQUE (event_id, seat_id)` on tickets was presented as the guarantee that holds even if every lock and every piece of application logic is wrong, with restoration and manual-insert paths covered separately by reconciliation.

## Memory Hooks

- Staleness is cheap, oversell is not. Consistency on the deciding path, freshness everywhere else.
- The hold is a state, not a lock. Never hold a database lock for the length of a user-facing timer.
- Confirm says `expires_at > now()`, the reaper says `expires_at <= now()`. Mutually exclusive, so exactly one wins, always.
- Validate all seats before mutating any, and lock in sorted order or you will deadlock.
- Redis renders, the database decides. If Redis dies, availability degrades and nothing else moves.
- A 100,000-seat bitmap is 12.5 KB. The whole on-sale catalogue is under 100 MB. Do the math, it picks your cache.
- The 429 must not touch Redis or the database, or shedding adds the load it removes.

## Weak-Spot Pointers

- **The waiting room was described but not designed.** No admission algorithm, no fairness or starvation argument, no discussion of what happens when a session is admitted and the sale has already sold out, and no answer for bot-driven queue-jumping where a bot holds 400 sessions. Drill: [[rate-limiter|Rate Limiter]] and [[backpressure|Backpressure]].
- **The payment saga was thin at the boundaries.** Void-on-lost-race was named but the case where a capture succeeds and the ticket insert fails was not worked through, and there is no story for partial authorisation or a PSP that authorises less than requested. Drill: [[saga-and-strangler|Saga Pattern]] and [[exactly-once-effect|Exactly-Once Semantics]].
- **Optimistic concurrency was rejected by assertion plus one argument.** You need the actual trade-off curve — when optimistic wins, what the retry amplification looks like, and why a bounded retry budget changes the answer. Drill: [[database-locking|Database Locking]] and [[isolation-levels|Isolation Levels]].
- **Multi-shard hot events were hand-waved.** "Sub-shard by section" was asserted without addressing that a cart spanning two sections then needs a cross-shard transaction, and without a threshold for when to trigger the split. Drill: [[shard-key|Shard Key]] and [[cross-shard-queries|Cross-Shard Queries]].
- **No alerting and SLO section.** Oversell counter, reclaim rate, hold-leak drift, and shed ratio were all mentioned in passing but never turned into an SLO with a threshold and an owner. Drill: [[sli-slo-sla|SLI, SLO, and SLA]] and [[golden-signals|Golden Signals]].

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the estimate and trade-off sections are where this session was won and lost.
- [[01-rapid-revision|Rapid Revision]] — re-read the consistency row; this problem is entirely a consistency trade-off.
- [[database-locking|Database Locking]] — the deepest dive in this session, and worth reading properly rather than from memory.
- [[isolation-levels|Isolation Levels]] — the section that decides whether `SELECT ... FOR UPDATE` is even correct.
- [[load-shedding|Load Shedding]] and [[circuit-breaker|Circuit Breaker]] — the flash-crowd control plane.
- [[hotspot-handling|Hotspot Handling]] — the framework for the hot-event argument the candidate made.
