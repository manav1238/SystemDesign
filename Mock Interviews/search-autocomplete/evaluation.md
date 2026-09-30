---
title: Search Autocomplete System Design - Evaluation
status: active
date: 2026-09-29
tags: [hld, mock, search-autocomplete, evaluation]
---

# Search Autocomplete System Design - Evaluation

## Scoring

| Phase | Score | Why |
|---|---|---|
| Phase 1: Requirements | Excellent | Immediately separated autocomplete from search, then pinned down the debounce, the minimum prefix length, and the top-K contract, which are the three parameters that drive every other decision. |
| Phase 2: Estimate | Excellent | Did the keystroke-to-request multiplier explicitly instead of hand-waving "one request per session", then produced a 12 GB memory budget for the hot trie and an egress figure that justified edge placement. |
| Phase 3: High-level design | Excellent | Clean split of offline build versus online serve, with the filter placed upstream of counting so bad data never enters the serving structure, and three named serving tiers with a stated miss path. |
| Phase 4: Deep dive | Excellent | Gave the concrete node layout and arithmetic that makes 250M nodes affordable, defended the trie over hash and sorted array, and made personal-as-boost rather than replace a stated invariant. |
| Phase 5: Trade-offs and follow-ups | Good | Eight named trade-offs and every push-back answered defensively. The gap is that the personalization record size was asserted across two different numbers without reconciling them. |

**Overall: strong hire signal.** The tell was treating this as a read-only derived-index problem from the first minute, which made the 5-minute freshness budget, free regional failover, and safe snapshot rollback all fall out naturally.

---

## What Made This a Strong Answer

- The keystroke math was done bottom-up and produced a 1.2 million requests-per-second peak, which then justified every expensive decision downstream instead of being retrofitted to them.
- The memory estimate showed representation matters more than algorithm: 15x between an object-graph trie and a flat double-array, plus a pruned top-N, plus shared string tables. That is the kind of detail that separates a real system from a sketch.
- Safety was treated as a hard guarantee placed in two independent places, upstream in the build and again on the serving path, with the drift risk acknowledged rather than hidden.
- Personalization was defined as a boost on top of the global list, never a replacement, with a per-user size budget and a privacy boundary and a user-facing delete path.
- Sharding by query prefix rather than by query hash, with the Zipfian load skew and the "a is 100x hotter than z" consequence both named.
- The metrics were product-level, not infrastructure-level: does my top-10 contain the query the user actually executed. Plus the silent-failure detectors, a same-ten-rows check and a zero-result rate, that catch a broken pipeline that still returns 200s.

---

## Memory Hooks

- Autocomplete is prefix search, not search. O(prefix + K) precomputed versus O(matching rows) live. That asymmetry is the whole design.
- One keystroke can be a request. 15 characters times 3e9 sessions a day is 4.2e10 requests a day, roughly 1.2 million per second at peak.
- Memory is the budget: 250M trie nodes at 12 bytes flat is 3 GB; the same as heap objects is 50 GB. Representation, not algorithm, is the win.
- Precompute from an immutable log, publish an immutable snapshot, swap a pointer. Bad builds then cannot take down serving.
- Popularity is log-compressed and window-blended, or one dominant query takes every slot.
- Personalize by boosting the global list. Replacement leaks privacy and starves cold-start users.
- Blocklist in the build and on the serving path, or a newly banned term keeps being served from old snapshots.

---

## Weak-Spot Pointers

- **The personalization size math is inconsistent.** The deep dive says 6 KB per user, the summary says 6 KB, but the estimation phase earlier floated a 20-query "cheap version" of under a kilobyte without ever reconciling which one ships. Interviewers notice arithmetic that contradicts itself. Drill [[memory-estimation]].
- **Debounce and cancellation got good coverage on the client and thin coverage on the server.** The abort, the sequence number, and the 150 ms client timer were all raised unprompted, which is a real strength, but nothing was said about server-side protection from a client that ignores the debounce. Rate limiting, coalescing, and a maximum in-flight budget belong there. Drill [[rate-limiter]] and [[backpressure]].
- **No attempt at a cost estimate.** Two figures were produced, 12 GB of RAM and 3 TB of personalization, and neither was turned into a monthly number or a per-DAU cost. Autocomplete is a top-3 cost center in most consumer apps and interviewers like to see it named. Drill [[cost-estimation]].
- **The trie rebuild was explained but the stream-to-snapshot consistency model was not fully closed.** The answer correctly says "keep them as separate features rather than merging two count sources", but the freshness SLO per feature, and what a user sees when the 5-minute overlay disagrees with yesterday's snapshot, was left implicit. Drill [[consistency-models]] and [[event-driven-architecture]].
- **Correction and typos were waved at.** "iphnoe falls back to a spell-corrected prefix" is one clause, and typo tolerance is one of the highest-value features in autocomplete. Edit distance, ngram-based correction, and the interaction with the prefix tree deserved real treatment. Drill [[elasticsearch]] and [[search-ranking]].

---

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] - the phase-by-phase checklist to run through before you walk in
- [[01-rapid-revision|Rapid Revision]] - the full vault compressed into a revision pass
- [[autocomplete|Autocomplete]] - the dedicated concept file for this problem
- [[probabilistic-data-structures]] - Count-Min Sketch and heavy hitters, which make the tail affordable
- [[mapreduce-lambda-kappa|MapReduce, Lambda, and Kappa]] - the batch versus stream decision in depth
- [[edge-computing]] - why this endpoint belongs near the user
