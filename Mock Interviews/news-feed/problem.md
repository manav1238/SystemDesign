---
title: News Feed System Design - Problem Statement
status: active
tags: [hld, mock, news-feed]
---

# News Feed System Design

## Problem Statement

> Design the backend of a social network news feed, in the style of Facebook, Twitter, or Instagram.
>
> Users can post short text updates with an optional image. Every logged-in user opens the app and immediately sees an ordered, personalized list of posts from the people they follow. The feed must feel instant: the app renders something useful within a few hundred milliseconds, and a new post from someone the user follows shows up on the next pull-to-refresh.
>
> In scope:
> - Creating a post (text plus optional media reference)
> - Following and unfollowing users
> - Fetching a user's home feed, paginated, newest first
> - Ranking posts so that the most relevant content appears at the top, not merely the most recent
> - Likes and comments count displayed on a post
>
> Out of scope:
> - Media transcoding, storage of the actual images, and CDN delivery
> - Direct messaging, groups, and ads
> - The content moderation pipeline
>
> Design for 100 million daily active users, roughly 300 feed posts read per user per day, and a meaningful minority of "celebrity" users with tens of millions of followers.

---

## Clarifying Questions You Should Ask

Strong candidates ask these before drawing anything. Pick the ones you genuinely need answered.

**On product shape**
1. Is the feed infinite-scroll chronological, or is there an explicit ranking signal? Who decides what "relevant" means?
2. Can a user follow anyone, or are there categories (friends, celebrities, groups) with different semantics?
3. Is the feed the same on mobile and web, and do logged-out users see anything?
4. Is there an unread/new-post marker, and does the user need to know exactly how many new posts arrived since they last visited?
5. Can users delete or edit a post, and if so, does the deletion have to disappear from already-generated feeds?

**On scale and data**
6. What is the DAU/MAU ratio, and how many posts does an average user create per day versus consume?
7. What is the follower distribution? How many users have over a million followers, and what is the 99.9th percentile follower count?
8. What is the target read latency, and is the feed allowed to be slightly stale?
9. What is the retention target for a feed item in the home feed, and what happens when a user has not logged in for six months?
10. Are posts ordered strictly by time, or is there a quality score, recency decay, and social signal (comments, likes) in the mix?

**On infrastructure**
11. Is eventual consistency acceptable for the feed, or does a like count need to be exact the instant it is tapped?
12. Are there hard latency budgets per tier, and is a 200 ms p99 acceptable for the first screen of items?
13. Can I assume Redis-class in-memory serving, or do I have to design from primitives up?
14. Is multi-region read traffic a real requirement now, or a later phase?
15. What is the acceptable cost ceiling per user per day? Fanout to celebrities is the dominant cost, so this drives the whole design.

---

## What You Are Evaluated On

### Phase 1: Requirements
Separate functional from non-functional. Pin down the actual product semantics (ranked vs chronological, follow graph, unread markers, edit/delete propagation). State assumptions explicitly rather than silently absorbing them.

### Phase 2: Estimate
Do the back-of-envelope math out loud with real numbers: DAU to MAU, posts per user, read QPS, write QPS, fanout amplification, per-user storage, total storage, cache size. Show the arithmetic, not just the conclusion.

### Phase 3: High-level design
Draw the architecture. Identify the post write path, the fanout path, the feed read path, the ranking service, the cache tier, and the underlying stores. Name the shard key for each store and say why.

### Phase 4: Deep dive
Pick two or three areas and go genuinely deep. The expected core is the pull/push/hybrid fanout decision including the celebrity problem, the feed cache data model with cursor pagination, and the ranking timeline with precomputed scores. Be ready to talk through the media-reference-only post path, cache miss handling, and cold start for inactive users.

### Phase 5: Trade-offs and follow-ups
Defend the choices against alternatives. "Why not read-time fanout for everyone", "why not put the feed in Cassandra or DynamoDB instead of Redis", "what happens when the primary feed shard dies", "what do you do about a single hot key", "how do you survive replica lag and a cache stampede". Know exactly which trade-off you accepted and what it cost.
