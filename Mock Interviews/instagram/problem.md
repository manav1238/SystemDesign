---
title: Instagram - Photo and Video Sharing Platform
status: active
tags: [hld, mock, instagram]
---

# Instagram - Photo and Video Sharing Platform

## Problem Statement

Design Instagram, a mobile-first social media platform where users upload photos and
videos, follow other users, and browse a personalized feed of content from the
accounts they follow.

You are a senior backend engineer on the platform team. The product manager has
handed you this brief, and you have one 45-minute system design interview to
present a scalable design.

> **The brief**
>
> Instagram lets users sign up, upload photos and videos, follow and unfollow
> other users, browse a personalized feed, like and comment on posts, view and
> create short-lived "Stories", browse content by hashtag, and receive push
> notifications when someone interacts with their content.
>
> Users expect the feed to feel instant: a like should show up in the counter
> within a second or two, and a new post from an account you follow should appear
> in your feed within a few seconds. They also expect images and videos to load
> instantly no matter where in the world they are.
>
> The content platform team is worried about two specific risks: (1) the
> upload pipeline breaks down and users cannot post from a particular region,
> and (2) a single viral creator with hundreds of millions of followers can
> overwhelm the feed for everyone else.
>
> There is no hard constraint on the exact product surface, but assume the
> feature set above is in scope for the initial design.

### Explicitly In Scope

- User profile, signup, login, follow and unfollow
- Media upload of photos and videos, including transcoding into multiple
  resolutions
- Feed generation and feed reads
- Hashtag browsing and trending hashtags
- Likes and comments, including displayed counts
- Stories with a 24-hour expiry
- Push notifications for interactions
- Content moderation of uploaded media
- Profile grid rendering

### Explicitly Out of Scope

- Recommendation and ranking of content for users who do not follow anyone
  (that is a separate ranking-system design)
- Live streaming
- Marketplace
- Payments or ads revenue optimization

### Scale Assumptions Given to You

- 1.5 billion registered accounts, 500 million daily active users
- 60 percent of daily active users post at least once per day
- 30 percent of daily active users post a video
- Average photo 3 MB, average 30-second video 20 MB
- 90 percent cache hit rate is the target for media delivery

## Clarifying Questions You Should Ask

A strong candidate does not start drawing boxes. Spend the first four or five
minutes on these questions, and record the answers you are assuming.

**About users and usage**

- "Is the feed strictly for accounts a user follows, or is there algorithmic
  discovery of accounts the user does not follow? This single answer changes the
  feed architecture from fanout-on-write to a ranking problem."
- "How many of the 500 million daily active users open the app mainly to post,
  mainly to scroll, or both? Posting users generate the write load and scrolling
  users generate the read load, and they peak at different times."
- "Do we need private accounts, where only approved followers can see a post?"
- "Is this a mobile-only design, or do we also need a web client with session
  based login?"

**About media**

- "What resolution and format do we serve? If we keep the original plus five
  renditions, storage per photo goes up by roughly 20 percent but the bandwidth
  saving on delivery is close to 5x. Which does the product care about?"
- "Are videos uploaded by the client already encoded, or do we re-encode
  everything server-side?"
- "What is the maximum acceptable time between an upload completing and the post
  appearing in the feed of followers? Seconds or minutes? This decides whether
  media processing is on the critical path."
- "Do we need to support resumable uploads for users on poor connections?"
- "How long do we retain original media, and do users have a right to deletion?"

**About engagement**

- "For likes, does the count have to be exactly correct, or is a count that is
  correct to within a minute acceptable?"
- "How deep do comment threads go, and do we need replies-to-replies?"
- "Do we want a global like count to keep increasing, or is it acceptable to
  roll up counts on older posts once traffic dies down?"

**About non-functional requirements**

- "What is the availability target? 99.9 percent is roughly 43 minutes of
  downtime per month, and 99.99 percent is 4 minutes. For a consumer social app
  that is not a bank, where do we draw the line between the feed and, say,
  payments?"
- "Is the feed eventually consistent acceptable? I am assuming a 5 to 30 second
  staleness is fine, but a like you just tapped must reflect back on your own
  device instantly because we optimistically render it client side."
- "What are the read and write latency targets for the feed? I would like to
  state them as percentiles, not averages."
- "Is this a single region design or multi-region? Global reach for media
  delivery is easy with a CDN, but global write is hard."
- "Do we have a hard budget? Storage and egress are the two dominant costs here
  and I want to reason about them explicitly."
- "What is the compliance bar? Deleting a user and everything they uploaded is a
  very different problem from anonymizing them."
- "What does abuse look like here? Spam, fake engagement, and non-consensual
  imagery each need a different control."

## What You Are Evaluated On

The interview is scored against these five phases. Each phase is a gate: a
candidate who nails estimation and then hand-waves the data layer does not pass,
and neither does a candidate with beautiful diagrams and no numbers.

### Phase 1: Requirements and Constraints

Do you distinguish functional requirements from non-functional requirements
without being prompted? Do you identify which features need strong consistency
and which tolerate staleness? Do you use the clarifying questions above to
narrow scope, or do you silently assume and never state the assumption?

Look for explicit mention of: read latency and throughput targets as
percentiles, availability target, media retention, the explicit staleness budget
for the feed, and the fact that a user's own action must read back instantly.

### Phase 2: Scale Estimation

Can you produce real numbers from memory in under five minutes, with the
arithmetic written out? The estimation must be internally consistent, and it
must be decomposed, not just totaled.

Look for: daily post volume, average and peak upload throughput in bytes per
second, daily media ingestion in TB, annual media storage, metadata row size
versus media size, the number of feed reads per day, and the read to write
ratio. The single biggest insight is that media bytes dominate storage by
roughly three orders of magnitude, and a candidate who notices that will frame
the entire storage design around [[blob-storage|Blob/Object Storage]] rather
than a database.

### Phase 3: API and Data Design

Do the APIs match the requirements, are they RESTful or gRPC with a stated
reason, and do they handle pagination, idempotency, and partial failure
explicitly? Are the schemas concrete enough to be reviewed, with real
partition keys and indexes?

Look for: presigned direct-to-object-storage uploads rather than proxying bytes
through application servers, cursor pagination for the feed, an idempotency
token on upload and like, a hashtag normalization step, and a schema where
counters are separated from the per-user interaction rows so that the hot rows
are small.

### Phase 4: Architecture and Data Flow

Is the high-level design decomposed into named, well-bounded services with
stated responsibilities? Can you trace a full request from the client to the
database and back, for both the write path and the read path? Is the diagram
legible and does every arrow have a reason?

Look for: the upload path going client to object storage directly, an
asynchronous media processing stage fed by a [[message-queue|Message Queue]],
fanout workers writing to per-user feed stores, a [[redis|Redis]] cache layer
for feed and counters, a [[cdn|CDN]] in front of every media byte, and a
[[publish-subscribe]] event stream for notifications and analytics.

### Phase 5: Deep Dive, Trade-offs, and Failure

This is where the interview is usually lost. Can you discuss
[[sharding|Sharding]] strategy and why the shard key differs per table, why
replication is used and what replica lag costs you, how a cache stampede is
prevented, and what happens when a single post receives more likes per second
than your write capacity.

Look for: the hot creator problem solved by hybrid fanout, at-least-once
delivery with idempotent consumers, sharded counters to absorb like spikes,
stale-while-revalidate plus jittered TTL on hot feed keys, circuit breakers
around every downstream call, and a specific answer to "what if the primary
database dies right now".
