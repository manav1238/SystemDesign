---
title: Distributed File Storage - Interview Evaluation
status: active
date: 2026-09-29
tags: [hld, mock, distributed-file-storage, evaluation]
---

# Distributed File Storage - Interview Evaluation

**Problem:** Google Drive / Dropbox style file storage and sync backend
**Duration:** 45 minutes
**Overall:** Strong hire signal. Consistent phase structure, real arithmetic, honest about trade-offs, and did not flinch on the durability math.

---

## Scorecard

| Phase | Score | Why |
|---|---|---|
| Phase 1: Requirements | 9/10 | Asked about commit visibility, conflict semantics, and the file size distribution before drawing anything; caught that the read-to-write ratio in bytes and in request count imply completely different problems. |
| Phase 2: Estimation | 9/10 | Modelled the size distribution explicitly as four bands and reconciled row count against byte volume; got 50B files and 40B unique chunks with visible arithmetic rather than a single mean. |
| Phase 3: High-level design | 8/10 | Clean separation of client, metadata, index, and chunk storage with named shard keys; lost a point for introducing the Chunk Index as a separate service without fully justifying the split at first mention. |
| Phase 4: Deep dive | 9/10 | The two-phase upload with an explicit commit boundary, the single-shard transactional refcount, and the Bloom filter negative-cache argument were all the right answers for the right reasons. |
| Phase 5: Trade-offs | 8/10 | Defended against S3, against whole-file objects, and against HDFS-style blocks; was honest that 4 MB fixed chunking weakens dedup, which is a real concession most candidates hide. |

**Total: 43/50**

---

## What Made This a Strong Answer

- **The distribution question came first.** Asking for the file size distribution before any estimation is the single highest-leverage question in this problem. The mean is meaningless, and a candidate who computes a mean file size produces a design sized 100x wrong in one direction or the other.
- **Commit was treated as the visibility boundary.** Making the commit transaction the single moment a file becomes real is what makes partial uploads invisible, resume safe, and dedup refcounting transactional. Everything else in the design falls out of that one choice.
- **The read path was sized separately from the write path.** 6,000 QPS of chunk writes versus 175,000 QPS of download requests and 560 Gbps of egress means the write path was never the problem, and saying so explicitly prevents the wasted "add more metadata shards" answer.
- **Storage class was used as a lever.** Splitting hot 3x replication from cold [[erasure-coding|erasure coding]] at 8+3 turns the long tail from 3x cost into 1.375x cost, and that is the largest single lever on the P&L for a 260 TB system.
- **Consistency was answered as three separate questions.** Immutable chunks are trivially consistent, metadata commits are strongly consistent, replica reads are bounded-stale with a read-your-writes token. Collapsing these into "it is eventually consistent" would have been wrong and would have lost the durability argument.
- **Degradation was thought about as behavior, not as a percentage.** The trash sweep backpressure rule, the "store a duplicate rather than a dangling pointer" fallback, and the bounded repair bandwidth are all the mark of someone who has run a storage cluster.

---

## Memory Hooks

- Commit is the visibility boundary. Nothing before it is real, everything after it is a queue consumer.
- Shard metadata by user, not by file, because folder listing is a range scan and must not be a cross-shard join.
- Fixed-size chunks for everything, content-defined chunks for large media, algorithm versioned in the manifest.
- Dedup scope is per tenant. Global dedup is a storage win and an inference attack.
- The Bloom filter is for negative existence checks, which is the 99 percent path, not the 1 percent path.
- Refcount lives transactionally on the committing shard; the in-memory index holds only placement, which is rebuildable.
- 11 nines comes from scrubbing, not from a fourth replica.

---

## Weak-Spot Pointers

- **The erasure coding durability claim was asserted, not derived.** Eleven nines requires an actual argument about node failure probability, correlated failures in an availability zone, and repair bandwidth versus fault rate. Drill [[erasure-coding|Erasure Coding]] specifically for the 8+3 versus 3x storage math and the silent-corruption case that replication alone does not cover.
- **Garbage collection and the commit race were hand-waved.** The grace period plus transactional refcount argument is right in shape, but the actual race, where a concurrent commit reads refcount one and then the sweep fires, deserves a precise treatment. Drill [[soft-delete-audit-tables|Soft Delete and Audit Tables]] and write out the interleaving.
- **No explicit backup and disaster recovery story.** Zone survival was covered, region loss was waved at with "backup and restore" and no RPO or RTO. For an 11-nines system this is a real gap. Drill [[rpo-rto|RPO and RTO]] and be able to state a number.
- **Distributed locks were used without the safety caveats.** The request coalescer and the cache-fill lock are correct uses, but the answer did not discuss lock expiry failure modes, the thundering-herd-on-expiry problem, or why a mutex is a performance tool and never a source of truth. Drill [[distributed-locks|Distributed Locks]].
- **Search by file name was listed as a functional requirement and never designed.** That is a legitimate scoping decision, but it should be said out loud as a deferral, not quietly dropped. Review [[search-engine|Search Engine]] for the inverted index option.

---

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] - the phase structure this transcript follows
- [[01-rapid-revision|Rapid Revision]] - the one-page recall sheet for storage, sharding, and durability decisions
- [[chunking-and-uploads|Chunking and Resumable Uploads]] - the commit protocol in full
- [[storage-tiering|Storage Tiering]] - hot, warm, and cold transitions
- [[shard-rebalancing|Shard Rebalancing]] - what actually happens when a node joins
