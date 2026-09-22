---
title: Database Keys
category: Database
priority: must-know
status: learning
difficulty: easy
interview_ready: false
tags:
  - hld
  - database
  - keys
---

# Database Keys

## 1. One-Line Definition
A key is a column or set of columns used to uniquely identify rows (primary key), connect tables (foreign key), or enforce uniqueness (unique constraint) — the skeleton on which relationships and lookups are built.

## 2. Why Do We Need It?
Without keys, you cannot refer to a specific row, join two tables, or prevent duplicates. Keys define identity (what makes a record *the* record), which is the basis for every index, join, constraint, and (at scale) every shard-key choice.

## 3. Simple Intuition
A passport number identifies one person (primary key). A booking reference on your ticket ties the booking to the flight (foreign key). The Aadhaar-style "each person has exactly one number" rule is a unique constraint. Remove the passport number and "book my seat" becomes "book *a* someone" — chaos.

## 4. What Happens Without It?
Duplicate rows pile up (can't tell two identical order entries apart), updates touch the wrong row, joins produce junk (usually Cartesian messes), and sharding/replication have no stable identity to route on. Every system operation becomes ambiguous.

## 5. Core Idea
- **Primary key:** one designated unique, non-null identifier per table. Often an auto-increment integer or a **UUID** (globally unique, no coordination — great for distributed inserts, cost: 128-bit storage and index bloat vs 64-bit).
- **Composite key:** PK made of multiple columns (order_id + line_no).
- **Foreign key:** column referencing another table's PK → enforces referential integrity (can't reference a nonexistent user). Parent/child relationship.
- **Unique constraint:** a column must be globally distinct (email, invoice number) — an implicit index.
- **Natural vs surrogate key:** natural (ISBN, email — meaningful, but fragile/unchangeable) vs surrogate (auto/seq id — meaningless, stable). Controversial but practical; most OLTP uses surrogate + unique constraints on natural fields.
- **Shard key (later link):** in sharded systems the identity of a row decides which node owns it — good shard keys spread writes evenly and keep related rows together (see sharding.md).

## 6. Important Terminology

| Term | Simple Meaning |
|------|----------------|
| Primary key (PK) | The designated unique row identifier |
| Surrogate key | Artificial, meaningless-but-stable id |
| Natural key | Real-world identifier (email, SSN-like) |
| Composite key | Multiple columns as one key |
| Foreign key (FK) | Reference to another table's PK |
| Unique constraint | Column(s) must be distinct — gets an index |
| UUID / ULID / Snowflake | Distributed-friendly unique id formats |
| Candidate key | Any column set *capable* of being PK |

## 7. Basic Architecture

```mermaid
flowchart LR
    O[orders] -->|user_id FK| U[users]
    U -->|PK: id| U
    O -->|PK: id| O
    OI[order_items] -->|order_id FK| O
    OI -->|PK: (order_id, sku)| OI
```

## 8. Request or Data Flow
1. Insert: PK enforces identity; FK ensures referenced parent exists; unique constraints reject duplicates.
2. Lookup: PK/unique-index gives O(log n) seek; FK enables the join query.
3. At scale: the PK/unique choice (auto-inc vs UUID) shapes write locality (heaps vs index-tree page contention, shard spread).

## 9. Practical Example
**E-commerce (assumptions):**
- `users(id BIGSERIAL PK, email UNIQUE)` — surrogate id + natural unique email.
- `orders(id, user_id FK, status, total)`.
- `order_items(order_id FK, sku, qty, price)` with composite PK.
- At scale: keep auto-increment for locality on write-heavy tables, switch to UUID for multi-datacenter inserts to avoid single-sequence contention and cross-region conflicts.

## 10. Scaling
- **Write locality:** auto-increment PKs cluster new rows at the tail → fast inserts, hot last-page contention at very high write QPS. UUID/random keys spread writes but fragment indexes (page splits, bloat) — the classic trade-off.
- **Sharding:** the PK/shard key must spread writes evenly and co-locate related rows (same user's rows on one shard) — a bad key = hot shards (see sharding.md).
- **Cross-region:** UUID/Snowflake avoid sequence coordination — plan for it in multi-region from day one.

## 11. Reliability and Failure Scenarios
- **Sequence exhaustion:** `int` PK runs out at 2^31 rows → migration pain. Use bigint.
- **Sequence contention at scale:** one sequence = one generator = a write hotspot / SPOF for inserts — Snowflake/UUID-esque generators replace it.
- **Duplicate on retry:** unique constraints double as idempotency enforcers (unique `payment_key` rejects the second identical charge attempt).

## 12. Consistency and Correctness
Constraints (PK/FK/UNIQUE) are the DB's enforcement of your data invariants. They *protect* consistency: a FK stops orphaned orders; a unique stops duplicated charges. Never weaken them "for performance" without carefully engineering (soft-delete tables, denorm copies) the invariant elsewhere.

## 13. Performance
- Key columns ARE indexes: each FK/unique auto-creates an index (write cost).
- UUID PKs bloat secondary indexes (every index embeds the PK) + random insert page splits — ~2-3x bigger than bigint in extreme cases. ULID/ordered UUIDs keep some locality.
- Covered/composite keys can carry hot read templates (see database-indexing.md).

## 14. Security
- Never expose internal surrogate IDs directly (prediction/enumeration — e.g., `GET /order/12345`); map to opaque token/UUIDs, or authorize by ownership on every access.
- Natural keys used as identifiers (email, SSN) leak in URLs/logs — treat them as data, not identity handles.

## 15. Trade-Offs

| Choice | Advantages | Disadvantages | When to Use |
|--------|------------|---------------|-------------|
| Auto-increment int/bigint | Small indexes, write locality | Sequence SPOF, no cross-region | OLTP, single-writer regions |
| UUID v4 | Global, coordination-free | Big indexes, insert fragmentation | Distributed inserts, sync |
| ULID / ordered UUID | UUID-like + time-ordered | Slight complexity | Distributed + ranged scans |
| Natural key as PK | No extra column | Fragile, changes, leaks | Read-only/gov data, immutable |
| Composite PK | Enforces joint uniqueness | Awkward FKs to it | Join-table rows |

## 16. Common Mistakes
- Using `int` PK then hitting 2B rows (exhaustion) — always bigint when in doubt scale.
- UUID as PK on tables that never leave one region (paying index bloat for nothing).
- Exposing sequential IDs in public URLs for private resources (enumeration attack).
- Foreign keys without a story for hard-deletes or tenant scoping (FKs cross-tenant leak if IDs enumerate).
- Wrapping everything in one "global unique id service" — see distributed-id-generation.

## 17. HLD vs LLD Boundary
HLD: id strategy per table/domain (auto-inc vs UUID vs snowflake), shard-key shape, unique/idempotency keys, exposure policy. LLD: the specific DDL, the repository mapping, ordinal vs composite mapping code.

## 18. Interview Questions

### Beginner
- What's the difference between a primary key and a foreign key?
- When would you use a natural key vs a surrogate key?

### Intermediate
- Why avoid auto-increment IDs for public resources?
- Design a queue that must never enqueue the same payment key twice.

### Advanced
- Auto-increment vs UUID for a multi-region write-heavy system — justify fully.
- How does your choice of PK change sharding and hotspot behavior?

## 19. Interview Memory Summary

> [!abstract]- Interview Memory Summary

### Remember

- PK = row identity; FK = relationships; UNIQUE = invariants.
- Surrogate vs natural keys: stable-but-meaningless vs meaningful-but-fragile.
- Keys ARE indexes → every key adds write cost.
- Auto-inc = write locality but sequence SPOF; UUID = global but bloated — pick by region/load.
- Keys power idempotency (unique constraint) at scale.
- The PK choice shapes sharding and hotspot behavior.

### 30-Second Explanation

Give every table a stable PK, unique-constrain real identity, FK the relationships, and choose auto-inc vs UUID by write locality + multi-region needs.

### Interview Traps

- "UUID solves all ID problems" — index bloat and fragmentation are a real tax; order your answer by requirement.
- Using `int` PK and hitting 2B rows — always bigint in doubt.
- UUID on tables that never leave one region (paying bloat for nothing).
- Exposing sequential IDs in public URLs (enumeration attacks).
- Foreign keys without a tenant-scoping story.

### Key Trade-Off

Auto-increment gives small indexes, write locality, and read speed but a sequence SPOF across regions; UUID gives coordination-free global uniqueness at the cost of index bloat, fragmentation, and weaker write locality — the choice hinges on whether you need multi-region/distributed inserts.

## 20. Related Concepts

### Prerequisites

- [[database-fundamentals|Database Fundamentals]]
- [[database-indexing|Database Indexing]]

### Commonly Used Together

- [[shard-key|Shard Key]]
- [[sharding|Sharding]]
- [[transactions-and-acid|Transactions and ACID]]

### Advanced Concepts

- [[consistent-hashing|Consistent Hashing]]
- [[sharding-strategies|Sharding Strategies]]
- [[partitioning-vs-sharding|Partitioning vs Sharding]]

Related planned topics (not authored yet): distributed ID generation.

## 21. References
PostgreSQL/MySQL key/constraint docs; standard database design texts. Verify current UUID/bigserial guidance with docs.

## 22. Active Recall

Test yourself before revealing each answer.

> [!question]- What is the difference between a primary key, foreign key, and unique constraint?
> A primary key uniquely identifies a row (one per table, non-null). A foreign key references another table's PK to enforce referential integrity (no orphaned rows). A unique constraint forces a column to be globally distinct (email, invoice number) and creates an index — it also powers idempotency.

> [!question]- Why are keys also indexes, and why does that cost writes?
> Every PK/unique/FK constraint creates an index automatically. Each index must be updated on every INSERT/UPDATE/DELETE — so more keys = more write work and storage on the write path.

> [!question]- Design decision: auto-increment vs UUID for a multi-region write-heavy system?
> Auto-increment gives compact indexes and write locality but a sequence is a single generator (SPOF/hotspot) and can't coordinate cross-region. UUID/Snowflake are coordination-free and global, but bloat indexes (~2-3x) and fragment inserts — pick distributed-friendly IDs for multi-region inserts from day one.

> [!question]- Trade-off: when should you keep a natural key vs a surrogate key?
> Natural keys (email, ISBN) are meaningful and queryable but fragile and leak PII. Surrogate keys (autoincrement/UUID) are stable but meaningless. In OLTP the practical pattern is surrogate PK + unique constraint on the natural fields.

> [!question]- Failure scenario: `int` PK runs out of values. What happens and how do you avoid it?
> At 2^31 rows the sequence exhausts and inserts fail — a painful migration. Use `bigint` when scale is a possibility, and replace single-sequence generators with Snowflake/UUID-esque generators in distributed write paths.

> [!question]- Interview scenario: a candidate says "UUID solves all ID problems." How do you respond?
> UUIDs solve global uniqueness and cross-region coordination, but they cost index bloat, fragmentation, and weaker write locality — pure tax on single-region tables that only need `bigint`. Order the answer by requirement: locality vs global distribution.

## 23. When Should I Use This?

### Use it when

- You need stable identity for rows (PK), relationships (FK), or invariants (UNIQUE).
- Idempotent retries matter (a unique `payment_key` rejects duplicate charges).
- Writes must spread evenly or co-locate related rows in a sharded system.
- Multi-region inserts need coordination-free IDs.

### Avoid it when

- You misuse keys as public identifiers (exposing sequential IDs → enumeration attacks).
- Requirements don't need distributed IDs but you pay UUID bloat anyway.
- Foreign keys cross tenant boundaries without scoping (leaks if IDs enumerate).

### What problem does it solve?

Problem: without keys you can't refer to a specific row, join tables, or prevent duplicates. Bottleneck: identity ambiguity breaks every index, constraint, and lookup. Solution: PKs define identity, FKs enforce relationships, unique constraints enforce invariants and idempotency — the skeleton every schema and shard key builds on.

### What problem does it NOT solve?

It doesn't tell you which ID format fits a region/load profile (that's the auto-inc-vs-UUID trade-off), and it doesn't prevent security problems like enumeration if you expose raw IDs publicly. It also doesn't pick the shard key — it just provides the building block.

## 24. Decision Connections

Decisions that go together with Database Keys:

- [[database-fundamentals|Database Fundamentals]] — keys are the identity layer of any schema.
- [[database-indexing|Database Indexing]] — keys create indexes; index strategy follows key choice.
- [[transactions-and-acid|Transactions and ACID]] — constraints are part of the correctness contract.
- [[shard-key|Shard Key]] — the PK/ID choice feeds directly into sharding placement.
- [[sharding|Sharding]] — stable identity enables horizontal partitioning.
- [[consistent-hashing|Consistent Hashing]] — distributed ID and placement interaction at scale.
- [[sharding-strategies|Sharding Strategies]] — how keys decide which shard owns a row.

Decision tree:

```
Pick an ID strategy for a table
    |
    +-- Single-region OLTP, insert-heavy?
    |      → auto-increment `bigint` (locality)
    |
    +-- Multi-region / distributed inserts?
    |      → [[database-keys|Database Keys]] (UUID/Snowflake)
    |
    +-- Real-world identity must be enforced?
    |      → surrogate PK + UNIQUE on natural field
    |
    +-- Retries must be idempotent?
    |      → unique constraint as the enforcement key
    |
    +-- Now sharding?
           → [[shard-key|Shard Key]] (spread + co-location)
```