---
title: "Design YouTube — Evaluation and Scoring"
status: active
date: 2026-09-29
tags: [hld, mock, youtube, evaluation]
---

# Design YouTube — Evaluation and Scoring

Session reviewed against the five phases in [[problem|Problem Statement]] and [[06-hld-interview-checklist|HLD Interview Checklist]].

## Scoring

| Phase | Score | Why |
|-------|-------|-----|
| 1. Requirements Clarification | 9/10 | Asked about live streaming, search scope, and retention before estimating, and proposed explicit availability, latency, and consistency targets instead of waiting to be asked. Lost a point for not pinning down the monetization and ad surface early. |
| 2. Scale Estimation | 9/10 | Derived 70K peak view-starts/s, 33.3M uploads/day, 85 Tbps egress, and 75 PB over retention, and cross-checked bandwidth two independent ways. The video-bytes versus metadata-bytes split was the strongest single moment. |
| 3. High-Level Architecture | 10/10 | Drew the upload and playback paths as visibly separate systems, named the control-plane versus data-plane split, and made the 150:1 read-to-write ratio the organizing fact of the design. |
| 4. Deep Dive | 9/10 | Covered transcoding as async queue plus autoscaled workers on queue depth, the three-tier counter design with 64-way hot-key slicing, cache layering with jitter plus single-flight plus XFetch, per-table shard keys with the conflicting-comment-key tradeoff stated honestly, and precomputed recommendations. Points lost for a light treatment of the search ranking path and of licensing. |
| 5. Trade-offs and Failure Scenarios | 10/10 | Every major decision had a named cost, and the failure walkthroughs were concrete: primary DB death with cache-bounded blast radius, 5M concurrent viewers on one video, cold cache after regional failover, replica lag escalating to replica ejection, and a poisoned transcode queue. The "choose the design so the lossy thing is the least valuable thing" framing was the answer of the session. |

**Overall: 47/50.** Hire.

## What Made This a Strong Answer

- Stated every assumption out loud before using it, and validated the assumption set against public figures rather than pretending to a precision the numbers do not have.
- Made the read-to-write asymmetry (roughly 150:1) the single organizing fact, so that every later scaling decision had a reason.
- Separated video bytes from metadata early, which is what allowed the whole system to be tractable: 85 Tbps of video never touches an application server.
- Named consistency per operation (playback eventual, comment strong, view count approximate) rather than declaring one model for the entire system, which is the difference between understanding [[strong-vs-eventual-consistency|strong vs eventual consistency]] and memorizing it.
- Attacked the hot-key problem three separate times: counter slicing by viewer hash, cache single-flight with jitter and XFetch, and CDN immutability, each with the specific failure it prevents.
- Ended with explicit, itemized trade-offs, including the ones that make the system worse on purpose: 10 seconds of view-count loss, 6-hour-stale recommendations, single-writer metadata instead of active-active.
- Described the shard primary failover as a consensus and fencing problem, not a health-check problem, which is exactly the [[split-brain|Split Brain]] trap.

## Memory Hooks

- YouTube is 150-to-1 read-heavy: 70K peak view-starts per second versus 460 uploads per second. Everything scales off that ratio.
- Video bytes never touch your app servers. Upload goes client to object storage via pre-signed parts; playback goes CDN to client. The app handles metadata JSON only.
- Three-tier view counting: Redis INCR, flush every 10 seconds, land in per-minute SQL buckets, refresh the denormalized total every 60 seconds. Divide your write rate by the batch interval, not by wishful thinking.
- Approximate everything that counts: HyperLogLog for unique viewers, Count-Min Sketch for engagement, rounded billions on display.
- Transcoding is a queue problem, not an HTTP problem. Autoscale on queue depth, not CPU. Fast-path 360p and 720p so the video is playable in 60 seconds.
- Hot video: Redis 64 slices by viewer hash, plus cache jitter plus single-flight plus XFetch, plus the CDN absorbing the actual bytes.
- Search documents are denormalized and slightly stale, because 20 point lookups on the search path is a tail-latency disaster nobody can feel.
- Comments shard by video_id, accept scatter-gather for user history, and back it with an async secondary index.
- Shard metadata by video_id, 64 shards at 70K QPS with 2x headroom, and split to 128 when traffic doubles.
- Global reads, single writer region for metadata. Video bytes stored once. Counters regional and summed, so a region undercounts instead of corrupting.

## Weak-Spot Pointers

- Transcoder internals stayed at the architectural level: no codec ladder construction rules, no AV1 versus H.264 selection criteria, no segment duration reasoning. Drill [[media-processing|Video Transcoding Pipeline]] and [[media-processing|Media Processing]].
- Approximate counting structures were named but not explained. You should be able to state the memory and accuracy properties of HyperLogLog and Count-Min Sketch. Drill [[probabilistic-data-structures|Probabilistic Data Structures]].
- Consensus was hand-waved during primary failover. Know how a coordinator promotes exactly one writable primary and fences the old one. Drill [[consensus|Consensus]] and [[raft-and-paxos|Raft and Paxos]].
- Shard isolation and blast radius were never discussed: what happens when one of 64 shards is unhealthy, and how a cell boundary bounds it. Drill [[cell-based-architecture|Cell-Based Architecture]] and [[tenancy-and-cells|Tenancy and Cells]].
- Monetization, ads, and content licensing were out of scope by choice, but "why not" on regulatory and rights constraints is a follow-up you will get. Drill [[data-residency|Data Residency]] and [[waf-ddos|WAF and DDoS]].

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the five-phase skeleton this session was scored against
- [[01-rapid-revision|Rapid Revision]] — one-liners per concept for the day before the interview
- [[problem|Problem Statement]] — the prompt, clarifying questions, and evaluation criteria
- [[interview|Interview Study Transcript]] — the full dialogue
