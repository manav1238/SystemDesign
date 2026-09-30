---
title: "Design Netflix — Evaluation and Scoring"
status: active
date: 2026-09-29
tags: [hld, mock, netflix, evaluation]
---

# Design Netflix — Evaluation and Scoring

Session reviewed against the five phases in [[problem|Problem Statement]] and [[06-hld-interview-checklist|HLD Interview Checklist]].

## Scoring

| Phase | Score | Why |
|-------|-------|-----|
| 1. Requirements Clarification | 10/10 | Opened on tier and ad insertion, then went straight to territory licensing, time-windowed rights, geo-blocking, and data residency, which is where the real design work lives. Proposed measurable SLIs: 700ms p50 startup, under 0.5 percent rebuffer, 99.9 percent playback availability. |
| 2. Scale Estimation | 10/10 | Derived 35M peak concurrent, 90 Tbps peak, and 675 PB/day egress, then decomposed traffic into three request classes with 4,900 play starts, 10-18M segment requests, and 3.5M heartbeats per second, which is the estimate that actually drives the architecture. Landed the 6 PB versus 675 PB/day contrast cleanly. |
| 3. High-Level Architecture | 9/10 | Device BFFs separated from internal services, the edge router made explicit as a decision point rather than hidden behind the word CDN, and the control plane scoped as regional. Points lost for a slightly thin treatment of the ad-tier manifest assembly path. |
| 4. Deep Dive | 9/10 | Effective-dated rights rows with a `min(ttl, valid_to - now)` cache rule and a fail-closed deny-list; progress write coalescing cutting 3.5M/s to 1.2M/s; row-wise versus column-wise ranking plus per-user artwork personalization; telemetry into a lake feeding the ladder optimizer. DRM and the experiment statistics were left as stubs. |
| 5. Trade-offs and Failure Scenarios | 9/10 | The seasonal-drop warm-up failure was the standout scenario, and raising the owned-CDN counter-argument unprompted was the strongest moment. Hot title, cold cache after region loss, primary death per data type, and replica lag all covered. Points lost for not stating an RTO for the entitlement path. |
| **Overall** | **47/50** | Hire. Stronger than the YouTube session on licensing and cost reasoning, slightly weaker on sharding depth. |

## What Made This a Strong Answer

- Reframed the problem as a bandwidth business within the first ten minutes, which is the correct thesis and made every later decision fall out of it instead of being asserted.
- Identified licensing as a data-modeling problem, not a business rule, and modeled it as effective-dated temporal rows with a derived cache TTL rather than a country boolean matrix. That single idea removes an entire class of real incidents.
- Counted heartbeats, not views. 3.5M progress writes per second is the number that sizes the app tier, and finding it is the difference between an answer that works and one that does not.
- Showed the startup-latency budget as a table and then pointed out that most of it is device launch and decode, not our API. Identifying the part you do not control is a mature signal.
- Argued for per-title encoding optimization on cost grounds with real arithmetic, which is both the technically correct answer and the one that connects engineering to the business.
- Raised the anti-argument against Open Connect unprompted, including the cache-must-never-be-an-authority rule, which shows the difference between advocating a design and evaluating one.
- Failed entitlements closed and failed history open, and said why: a wrong licensing answer costs money and a slow history answer costs the user. That is a principled consistency split, not a memorized one.

## Memory Hooks

- This is 85 to 90 percent egress cost, so it is a bandwidth business: 35M peak concurrent, 90 Tbps, 675 PB/day. Design the delivery path first, catalog second.
- Three request classes: 5K play starts, 10-18M segment requests, 3.5M heartbeats per second. The app tier is sized by heartbeats, not views.
- Six petabytes of video against 675 petabytes a day of traffic. Storage is rounding error, so do not spend design budget there.
- Per-title encoding ladder is 25 to 40 percent bandwidth saved, roughly 7 million dollars a day, from a pipeline that costs almost nothing. Best ROI in the system.
- Own the edge where volume is high, rent it where coverage is needed. The answer is a hybrid, and the third-party CDN is a permanent fallback, not a temporary one.
- Rights are temporal rows, not booleans. Cache TTL = `min(config, valid_to - now)`. Never fail open.
- Catalog is 10 GB, so replicate it everywhere and keep it fully in memory. Never let a small dataset be a cache-coherence problem.
- Progress: coalesce in memory, flush every 30 seconds, last-writer-wins, so 1.2M writes per second and no exactly-once needed.
- Any table keyed by (user, entity) needs a denormalized per-user index, or the hot per-user read becomes a scatter-gather.
- Start playback on a conservative bitrate immediately, then step up. A blurred first frame beats a spinner.
- Row-wise ranking is real-time and must be cheap; column-wise ranking is a batch job and may be four hours stale.

## Weak-Spot Pointers

- Per-title encoding optimization was argued economically but never technically. You should be able to sketch a rate-quality curve and explain operating-point selection. Drill [[media-processing|Media Processing]] and [[compression|Compression]].
- Sharding depth was the weakest area, and it is exactly where the YouTube session was strongest. You gave 256 progress shards but never justified the number or discussed rebalancing. Drill [[shard-rebalancing|Shard Rebalancing]] and [[shard-key|Shard Key]].
- Multi-master for the progress store was asserted without conflict-detection mechanics. Know how a conflict is detected rather than assumed absent. Drill [[multi-region-consensus|Multi-Region Consensus]] and [[conflict-resolution|Conflict Resolution]].
- The experiment platform was hand-waved. The hard parts are sequential testing and guardrail metrics, not bucket assignment. Drill [[sli-slo-sla|SLI / SLO / SLA]] and [[observability|Observability]].
- DRM, device trust, and the ad-tier manifest assembly path were excluded. Expect a follow-up on each, and expect "why not exclude these" as the actual question. Drill [[encryption-and-keys|Encryption and Keys]] and [[authentication-vs-authorization|Authentication vs Authorization]].

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the five-phase skeleton this session was scored against
- [[01-rapid-revision|Rapid Revision]] — one-liners per concept for the day before the interview
- [[problem|Problem Statement]] — the prompt, clarifying questions, and evaluation criteria
- [[interview|Interview Study Transcript]] — the full dialogue
