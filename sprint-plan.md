---
title: 10-Day HLD + LLD Sprint (by Sept 30)
status: active
updated: 2026-09-21
---

# 10-Day HLD + LLD Sprint

Goal: interview-ready on the **maximum topics** in ~10 days. You are NOT going to read the knowledge base. The KB is a reference; you will only ever touch tiny slices of it.

## The 5-Minute Bite (reading protocol)

For ANY concept file, do ONLY this:

1. Guess: "what is this about?" in one sentence.
2. Open Section 19 (`> [!abstract]- Interview Memory Summary`) — the Remember list + 30-Second Explanation.
3. Do Section 22 (`> [!question]-`) **aloud** before peeking at answers.
4. If you get a question wrong, read Section 5 (Core Idea) and Section 15 (Trade-Offs). That's it.

Total: 5–10 minutes per concept. The other 20 sections exist for later deep-dives; ignore them this sprint.

## Daily Flow (about 3 hrs)

- 45 min — morning recall: yesterday's Section 22 questions without looking.
- 90 min — new concepts using the 5-Minute Bite.
- 45 min — practice problem (mock or written).

---

## Week 1 — HLD floor (Days 1–5)

### Day 1 (Sept 21) — Interview skeleton
- Concepts: System Design Fundamentals · Functional vs Non-Functional Requirements · Capacity Estimation
- Also read once: `../Revision/06-hld-interview-checklist.md` (whole thing).
- Practice: walk "Design a URL shortener" mentally using the 5 phases (no writing yet).

### Day 2 (Sept 22) — The core quartet
- Concepts: Scalability · Availability · Reliability · Latency vs Throughput · Horizontal vs Vertical Scaling · Load Balancing · Caching
- Practice: capacity estimate for URL shortener on paper (DAU → QPS → storage).

### Day 3 (Sept 23) — Data layer
- Concepts: Databases Fundamentals · SQL vs NoSQL · Database Indexing · Database Replication · Sharding · Consistent Hashing · CAP Theorem
- Practice: "SQL or NoSQL?" for 5 mini-scenarios, explained out loud.

### Day 4 (Sept 24) — Async + hardening
- Concepts: Message Queue · Kafka Architecture · Retry and Timeout · Circuit Breaker · Rate Limiter
- Clean Practice Day: re-do ALL Section 22 from Days 1–3 (you should have ~50 questions).

### Day 5 (Sept 25) — First full HLD mock
- Mock: **URL Shortener** (full 45 min, use the checklist phases).
- After: read its decision tree in `../Revision/02-database-decisions.md` + `03-caching-decisions.md`.
- Fix your top 2 weak spots by re-doing the 5-Minute Bite on those files only.

---

## Week 2 — LLD + HLD mock reps (Days 6–10)

### Day 6 (Sept 26) — LLD: OOP + classics
- Topics (short notes, not files): SOLID, composition over inheritance, interface design.
- Practices (written): **LRU Cache** · **Parking Lot**.

### Day 7 (Sept 27) — LLD: concurrency + state machines
- Topics: locking vs atomic, producer-consumer, thread pools, idempotency in requests.
- Practices (written): **Snake and Ladders** · **Vending Machine** · **Load Balancer OOD**.

### Day 8 (Sept 28) — Second full HLD mock
- Mock: **News Feed** (design Twitter-style).
- After: `../Revision/01-rapid-revision.md` aloud, top to bottom.

### Day 9 (Sept 29) — Third HLD mock + weak-spot surgery
- Mock: **Chat System** (or your pick).
- After: re-Bite ONLY the 2–3 concepts you flubbed.

### Day 10 (Sept 30) — Final drill
- `../Revision/01-rapid-revision.md` aloud (all 61+ lines).
- One 30-minute mock of **Rate Limiter** to finish sharp.
- Read `../Revision/05-reliability-decisions.md` (last gap close).

---

## What to skip (do not touch this sprint)

- All `Advanced` priority files (deep-dive sections 18, 21, Kafka internals, distributed consensus).
- Any concept's sections 1–18 and 21–24 unless you missed recall questions on it.
- None of the 247 files need a full read. Speed beats depth for these 10 days.

## How mocks work

- `HLD/Mock Interviews/` and `LLD/` are empty right now.
- On each mock day, ask me and I'll generate the problem session + a scored review from the sprint plan. One request per mock day keeps you honest.

## One rule

If a concept still escapes you after 2 Bites and its decision tree, write the filename on paper and move on. Revisit it on Sept 30. Momentum > perfection.