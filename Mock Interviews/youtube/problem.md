---
title: "Design YouTube (study transcript)"
status: active
tags: [hld, mock, youtube]
---

# Design YouTube — Problem Statement

## Problem Statement

Design a video-sharing and video-streaming platform similar to YouTube.

**Core product surface:**

- Users can register, log in, and manage a channel.
- Users can upload videos, supply a title, description, tags, and visibility setting.
- Uploaded videos are transcoded into multiple resolutions and formats (for example 144p through 4K, in both HLS and DASH container formats) so that playback works on any device and network.
- Users can watch videos. Playback is adaptive: the client picks the bitrate that matches its current bandwidth.
- Users can browse, search, and get recommendations for videos.
- Users can like, dislike, comment, subscribe, and leave view counts. View counts are displayed as approximate numbers.
- Video creators can see analytics about their own videos.

**The interviewer says:**

> "Assume roughly 2 billion monthly active users, of which about 50 percent use the service on any given day. Peak traffic is roughly 20 percent above the daily average. Assume the average user uploads one video every 30 days and watches 5 videos a day. Design the system so that it survives a single-region failure, supports horizontal scale-out, and stays affordable. Explain your storage math, your bandwidth math, and how the upload and playback paths differ."

## Clarifying Questions You Should Ask

Ask these before you draw a single box. Each one signals that you are scoping the problem deliberately rather than guessing.

### Product scope

1. Are we designing for live streaming as well, or only on-demand video? Live streaming introduces ingest, low-latency pipelines, and per-second manifests, which changes the whole design.
2. Do we need editing, playlists, and downloads, or is the core loop upload → transcode → watch → engage?
3. Is creator-side analytics in scope, or just the user-facing product? Analytics implies a separate analytics pipeline and data store.
4. Do we need a monetization or ads layer? Ads add an ad-serving service, impression counting, and a real-time bidding surface.
5. Is search a first-class feature here, or is it an assumed given? If it is in scope, I need an indexing and relevance system.
6. Do we support private and unlisted videos with restricted access, or are all videos public?
7. Do users need live chat or comments threaded in real time, and does that need to be strongly ordered?

### Scale and users

8. What are the monthly active users, daily active users, and peak concurrency? If not given, I will propose numbers and say so out loud.
9. What is the average and the 99th-percentile video duration? This dominates storage.
10. What is the average video bitrate at ingest, and what is the bitrate ladder we need to transcode into?
11. What proportion of watch time comes from mobile versus desktop and TV? This changes playback and CDN strategy.
12. What is the read-to-write ratio? I expect it to be enormously read-heavy, but I want it confirmed.

### Non-functional requirements

13. What is the availability target? I would propose 99.95 percent or better for playback, and note that upload can degrade more gracefully than playback.
14. How fast must a video go from upload to playable? Minutes is acceptable; seconds is a different product.
15. How stale can view counts, likes, and subscriber counts be? I intend to argue for eventual consistency and defend it.
16. Is search allowed to be a few hundred milliseconds, or does it need to feel instant? This decides whether search is a separate index or a direct database query.
17. What is the startup-latency and rebuffering budget for playback? Startup latency is the metric users actually feel.
18. Are there regional data-residency or licensing constraints? This decides whether metadata is global or regional.

### Constraints and preferences

19. Which cloud or platform are we building on, and is there a preference for managed services?
20. Do we prefer to build the CDN and transcoding stack ourselves or buy it?
21. Is there an existing video player client, and do we control the client? This decides how much playback logic is ours versus the CDN's.
22. What is the priority order when we have to trade off: freshness of counts, startup latency, personalization quality, or cost?

## What You Are Evaluated On

The interview is scored on five phases, matching [[06-hld-interview-checklist|HLD Interview Checklist]].

### Phase 1 — Requirements Clarification

Did you ask scoping questions before estimating, or did you start designing immediately? Did you separate functional from non-functional requirements explicitly? Did you surface the assumptions you are about to make instead of burying them? See [[functional-vs-non-functional-requirements|Functional vs Non-Functional Requirements]].

### Phase 2 — Scale Estimation

Did you derive daily active users, requests per second, peak QPS, storage, and bandwidth, showing arithmetic with real numbers? Did you round sanely and state the method rather than the precision? Did you estimate both the video-bytes side and the metadata side separately, because they differ by three orders of magnitude? See [[capacity-estimation|Capacity Estimation]] and [[server-capacity|Server Capacity]].

### Phase 3 — High-Level Architecture

Did you draw a clear box-and-arrow diagram with named services? Did you identify the upload path and the playback path as two genuinely different systems? Did you correctly place [[cdn|CDN]] in the read path and [[blob-storage|Blob/Object Storage]] in the write path, and justify that placement? See [[service-oriented-architecture|Service-Oriented Architecture]] and [[control-plane-vs-data-plane|Control Plane vs Data Plane]].

### Phase 4 — Deep Dive

Did you go deep on at least four of: upload and chunking, asynchronous transcoding via [[message-queue|Message Queue]] and worker pools, the storage layout and [[storage-tiering|Storage Tiering]], metadata [[database-replication|Database Replication]] and [[caching|Caching]], [[sharding|Sharding]] and shard key choice, search indexing, recommendation precomputation, and view-count aggregation? Did you reason about consistency per data type rather than picking one global model? See [[strong-vs-eventual-consistency|Strong vs Eventual Consistency]].

### Phase 5 — Trade-offs and Failure Scenarios

Did you name the cost of every major decision? Did you walk through concrete failures: primary database death, a hot video, a transcoding backlog, CDN origin saturation, replica lag, cache stampede? Did you end with a coherent summary of what breaks first and how the system degrades? See [[graceful-degradation|Graceful Degradation]], [[trade-off-analysis|Trade-Off Analysis]], and [[single-point-of-failure|Single Point of Failure]].

## Related Reading

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the skeleton this problem is scored against
- [[01-rapid-revision|Rapid Revision]] — one-liners per concept for the day before
- [[media-processing|Video Transcoding Pipeline]] — the core of the upload path
- [[media-processing|Media Processing]] — chunking, ladders, and packaging formats
