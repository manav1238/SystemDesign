---
title: Transactions and ACID
category: Database
priority: must-know
status: learning
difficulty: medium
interview_ready: false
tags:
  - hld
  - database
  - transactions
  - acid
---

# Transactions and ACID

## 1. One-Line Definition
A transaction groups multiple database operations into one all-or-nothing unit; ACID (Atomicity, Consistency, Isolation, Durability) is the contract that makes those units correct even under concurrency and crashes.

## 2. Why Do We Need It?
Real operations touch multiple rows: "debit account A, credit account B". If the process crashes between them, or two threads race, money vanishes or duplicates. Transactions guarantee such steps behave as one atomic step with a consistent, concurrent view of the world — the foundation of correctness for any system with state.

## 3. Simple Intuition
A bank transfer at a counter: the teller (transaction) completes both debit and credit, or performs neither — no "half-transfer". While it's happening, other customers see the *before* or *after* balance, never the in-between. The ledger (WAL) captures it first so a power cut can replay or roll back.

## 4. What Happens Without It?
Half-applied updates snake into the system: money debited but credit lost, inventory decremented but order not created, profile saved but payment failed. Concurrency turns it worse: two orders read stock=1, both decrement to 0 → oversold. Duplicates and corruptions accumulate because every interleaving is possible.

## 5. Core Idea
**ACID breakdown:**
- **Atomicity** — all steps commit, or all roll back (undo). No partial effect.
- **Consistency** — committed transactions move the DB from one valid state to another (constraints, invariants hold — e.g., balance ≥ 0). (This is DB-consistency, not distributed/consistency in the CAP sense.)
- **Isolation** — concurrent transactions don't see each other's uncommitted/in-bedrock intermediate states. Implemented via **locking** (pessimistic) or **MVCC snapshots** (optimistic: readers see a stable version).
- **Durability** — once committed, survives crashes (WAL/fsync → replays on restart).

**Isolation levels** (trade staleness/porch for speed):
- Read Uncommitted → **dirty reads**
- Read Committed → no dirty reads, but **non-repeatable reads** (row changes mid-read)
- Repeatable Read → stable row reads, but **phantom reads** (rows appear/disappear in ranges)
- Serializable → fully isolated (slowest)

**How it's implemented:** pessimistic = locks held (row/range) until commit; optimistic/MVCC = versioned snapshots + abort on conflict at commit.

## 6. Important Terminology

| Term | Simple Meaning |
|------|----------------|
| Commit | Make the transaction durable and visible |
| Rollback | Undo the transaction's changes |
| WAL / redo log | Append-only durable log replayed on crash |
| Isolation level | How strictly concurrent txs are separated |
| Dirty read | Seeing another tx's uncommitted data |
| Non-repeatable read | Rows changing between two reads in one tx |
| Phantom read | A range's membership changing mid-query |
| Lock | Permission gate serializing access to a row/range |
| MVCC | Multi-version concurrency control: snapshots instead of block |
| Deadlock | Two txs wait on each other's locks forever → DB detects + aborts one |

## 7. Basic Architecture

```mermaid
sequenceDiagram
    participant App
    participant DB
    App->>DB: BEGIN
    App->>DB: UPDATE accounts SET bal=bal-100 WHERE id=A
    App->>DB: UPDATE accounts SET bal=bal+100 WHERE id=B
    App->>DB: COMMIT
    Note over DB: WAL written first; on crash, replay = durable
```

## 8. Request or Data Flow
1. `BEGIN` opens a transaction; DB assigns snapshot/locks.
2. Operations run under the isolation level's rules.
3. On success `COMMIT` → WAL/redo durable → release locks/snapshot; concurrent readers see the new state.
4. On error/crash `ROLLBACK` (or auto) → undo partial changes; other txs were never affected.

## 9. Practical Example
**Order + inventory (assumptions):** never oversell, never orphan an order.
- `BEGIN; check stock; UPDATE stock; INSERT order; COMMIT;` → atomic.
- Retry-aware app code: on commmit-time conflict in optimistic mode → re-read + retry with backoff.
- Long-tx antipattern: never hold a transaction open across HTTP/API calls — scope per DB.

## 10. Scaling
- **The scaling wall:** single-node ACID is easy; scaling transactions across nodes is the hard problem. Cross-shard = distributed transactions (2PC/Saga) — costly.
- **Isolation cost:** Serializable serializes writes per hot row → hot-row bottleneck; Repeatable Read is the practical max for most OLTP.
- **Write scaling:** partition to keep transactions inside one shard; design for single-shard transactions wherever possible.
- **Hot row:** a popular balance (celebrity account) serializes — shard/async or restructure (e.g., counters in cache reconciled to DB).

## 11. Reliability and Failure Scenarios
- **Crash mid-tx:** WAL replay → committed become durable, uncommitted roll back. No half state.
- **Deadlock:** DB detects cycle → aborts one tx → app retries (with jitter). Tune lock order/timeouts.
- **Lock explosion / starvation:** long-running transactions under concurrency → timeouts & retry storms → prefer short txs + MVCC reads.
- **Replica lag:** committed writes replicated async — a read from a stale replica is consistent with an *older snapshot* (fine for read-heavy).

## 12. Consistency and Correctness
- ACID-consistency = database-level invariants, not cluster-uniform reads.
- Cross-service, cross-DB "single transaction" is fiction — model with Sagas/outbox + idempotency (graceful eventual). Distinguish "no atomic real-time across services" from "a single primary DB that is ACID for its own rows."
- Exactly the trap to name aloud: strong consistency ≠ atomic across the world; it scopes to whoever shares the transaction.

## 13. Performance
- Durability = fsync/WAL cost (cheap if `synchronous_commit` tuned + GROUP COMMIT).
- Index writes accelerate per write op; each index update costs the same "log plus tree write" per statement.
- Snapshot (MVCC) reads scale: no blocking readers/writers.
- p99 of a hot-row tx is dominated by lock wait + retry — monitor lock/conflict metrics.

## 14. Security
- Transactions scope security too: authorization checks *inside* the same tx so a "read-after-privilege-check" race can't leak (e.g., check tenant + read row atomically).
- Least-privilege DB roles: transactions should run as the narrowest role; no ad-hoc `DROP`/`DDL` in app services.

## 15. Trade-Offs

| Choice | Advantages | Disadvantages | When to Use |
|--------|------------|---------------|-------------|
| Read Committed | Good perf, no dirty reads | Non-repeatable reads | Default OLTP |
| Repeatable Read | Stable per-tx view | More conflict abort chance | Finance-style reads |
| Serializable | Full correctness | Slowest, hot-row serialization | Strictly money, low hot rows |
| MVCC reads | Non-blocking | Storage for versions | Whenever available |
| Locked writes | Simple rules | Writers block writers | High-conflict rows |
| Cross-node tx (2PC/Saga) | Global atomicity | Slowness/complexity | Rarely; prefer same-shard |

## 16. Common Mistakes
- Holding a DB transaction open across HTTP/service calls (lock held → deadlock/contention).
- Assuming every NoSQL store has ACID — MongoDB/Cassandra give different, weaker guarantees; verify.
- "We're ACID so we're consistent globally" — ACID is per-DB, per-transaction; cross-service writes are still your problem.
- Optimistic locks without conflict-retry loops → user-facing failures for a known-conflict limb.
- Overusing Serializable for hot counters → self-inflicted bottleneck.

## 17. HLD vs LLD Boundary
HLD: transaction scope per operation, isolation-level policy, write path shape (single-shard vs saga), hot-row strategy. LLD: transaction annotation/begin-commit-rollback in a service, retry wrapper, ORM flush ordering.

## 18. Interview Questions

### Beginner
- What does ACID stand for and why does it matter?
- What's the difference between a dirty read and a phantom read?

### Intermediate
- How does MVCC let readers and writers avoid blocking?
- A hot account row is serialized; what transactional alternatives exist?

### Advanced
- Design a multi-service money transfer without dropping ACID. Where do you give it up?
- When is Serializable isolation worth its cost, and how do you detect conflicts?

## 19. Interview Memory Summary

> [!abstract]- Interview Memory Summary

### Remember

- Tx = all-or-nothing unit (BEGIN → ops → COMMIT/ROLLBACK).
- ACID = atomicity, consistency, isolation, durability.
- Isolation levels trade correctness vs speed (dirty/non-repeatable/phantom).
- Implemented via locks (pessimistic) or MVCC (optimistic).
- ACID is per-DB, per-transaction — cross-service atomicity is a design problem.
- Keep transactions short; don't cross service boundaries with one "transaction".

### 30-Second Explanation

BEGIN→ops→COMMIT/ROLLBACK with WAL durability; pick the right isolation level; keep transactions short; don't cross service boundaries with one "transaction".

### Interview Traps

- "ACID across all our microservices" — ACID is a per-database, per-transaction property.
- Holding a DB transaction open across HTTP/service calls (lock held → deadlock).
- Assuming every NoSQL store has ACID — verify actual guarantees.
- Optimistic locks without conflict-retry loops → user-facing failures.
- Overusing Serializable for hot counters → self-inflicted bottleneck.

### Key Trade-Off

Strong isolation and cross-node atomicity buy correctness at the cost of latency, blocking, and availability; the balance is scoped single-shard, short transactions with the cheapest isolation level that preserves invariants.

## 20. Related Concepts

### Prerequisites

- [[database-fundamentals|Database Fundamentals]]
- [[database-connection-pooling|Database Connection Pooling]]

### Commonly Used Together

- [[sql-vs-nosql|SQL vs NoSQL]]
- [[database-replication|Database Replication]]

### Advanced Concepts

- [[cap-theorem|CAP Theorem]]
- [[strong-vs-eventual-consistency|Strong vs Eventual Consistency]]
- [[sharding|Sharding]]
- [[outbox-pattern|Outbox Pattern]]

Related planned topics (not authored yet): BASE, isolation levels, distributed transactions, saga.

## 21. References
PostgreSQL and MySQL docs on transactions/MVCC; Kleppmann ch. 7 in *Designing Data-Intensive Applications*. Verify isolation semantics per engine.

## 22. Active Recall

Test yourself before revealing each answer.

> [!question]- What does each letter of ACID guarantee?
> **Atomicity** — all operations commit or all roll back, no partial effect. **Consistency** — committed transactions move the DB from one valid state to another (constraints hold). **Isolation** — concurrent transactions don't see each other's uncommitted/intermediate states. **Durability** — once committed, writes survive crashes (WAL/fsync).

> [!question]- What's the difference between a dirty read and a phantom read?
> A **dirty read** sees another transaction's uncommitted data (Read Uncommitted). A **phantom read** happens when a range's membership changes mid-query (rows appear/disappear) under Repeatable Read, because only rows (not ranges) are locked. Read Committed fixes dirty but not non-repeatable; Serializable fixes all.

> [!question]- How does MVCC let readers and writers avoid blocking?
> MVCC keeps versioned snapshots instead of blocking: readers see a stable snapshot at their read time, so a reader never waits on a writer and vice versa. A writer aborts on conflict at commit if it raced — optimistic concurrency instead of holding locks.

> [!question]- Trade-off: which isolation level should a money system use?
> Money wants strong isolation — Repeatable Read or Serializable for strict correctness, but those serialize hot rows and abort on conflicts. Practical finance: Repeatable Read with short transactions and retry loops; use Serializable only where hot-row contention is low and stricter guarantees genuinely buy correctness.

> [!question]- Failure scenario: two transactions deadlock. What happens?
> The database detects the lock cycle, aborts one transaction, and the app retries (with jitter). To prevent it: keep transactions short, acquire locks in a consistent order, and avoid holding transactions across network/HTTP calls — long transactions are the root cause of lock storms.

> [!question]- Interview scenario: a candidate says "we'd like ACID across all our microservices." How do you respond?
> ACID is a per-database, per-transaction property — a single cross-service "transaction" is fiction. For cross-service consistency, model with Sagas/outbox + idempotency (graceful eventual), not with one shared transaction. Name the scope: same-shard transaction vs cross-service saga.

> [!question]- Interview scenario: design a multi-service money transfer without dropping ACID.
> Keep each service's write transaction ACID within its own DB (single-shard, scoped). Across services, use an outbox pattern to reliably emit intents + idempotent processing so the transfer eventually completes; a saga manages compensating rollbacks. You give up global real-time atomicity and gain availability.

## 23. When Should I Use This?

### Use it when

- An operation touches multiple rows that must all commit or all fail (debit+credit, order+inventory).
- Concurrency could corrupt state (overselling, duplicates) without isolation.
- Durability is non-negotiable (money, auth state).
- You're scoping correctness per database on a single node or shard.

### Avoid it when

- The "transaction" spans services or databases (convert to Saga/outbox).
- Hot-row writes are extreme (Serializable serializes the hot row — bottleneck).
- You'd hold a transaction open across HTTP/service calls.
- The store doesn't actually provide the isolation you assume (verify NoSQL guarantees).

### What problem does it solve?

Problem: real operations touch multiple rows, and crashes or races between them corrupt state (half-applied transfers, oversold inventory). Bottleneck: no way to make multi-row work atomic, isolated, and durable under concurrency. Solution: transactions + ACID — atomic commit/rollback with isolation levels and WAL durability.

### What problem does it NOT solve?

It does not provide atomicity across services or databases (that's Sagas/outbox + idempotency), doesn't guarantee global/linearizable consistency across clusters (that's per-transaction scope), and doesn't make hot-row contention free — isolation still serializes and aborts under conflict.

## 24. Decision Connections

Decisions that go together with Transactions and ACID:

- [[database-fundamentals|Database Fundamentals]] — transactions are the correctness layer of the DB.
- [[database-connection-pooling|Database Connection Pooling]] — the pool must not hold long transactions.
- [[sql-vs-nosql|SQL vs NoSQL]] — ACID strength is a core reason to pick SQL.
- [[sharding|Sharding]] — single-shard design keeps transactions ACID; cross-shard changes the model.
- [[database-replication|Database Replication]] — durability + read scale while write transactions hit the leader.
- [[outbox-pattern|Outbox Pattern]] — the reliable way to extend atomicity across services.
- [[cap-theorem|CAP Theorem]] — the distributed constraints that limit global ACID.
- [[strong-vs-eventual-consistency|Strong vs Eventual Consistency]] — what you trade when leaving transactional scope.

Decision tree:

```
Operation must be all-or-nothing
    |
    +-- Within one DB / one shard?
    |      → [[transactions-and-acid|Transactions and ACID]] (BEGIN/COMMIT + isolation)
    |
    +-- Spans services or databases?
    |      → [[outbox-pattern|Outbox Pattern]] + Saga (eventual)
    |
    +-- Hot row writes dominant?
    |      → avoid Serializable; shorten transactions
    |
    +-- NoSQL chosen for scale?
    |      → verify actual guarantees ([[sql-vs-nosql|SQL vs NoSQL]])
    |
    +-- Need cross-node atomicity anyway?
    |      → distributed tx or redesign to [[sharding|Sharding]] single-shard scope
```