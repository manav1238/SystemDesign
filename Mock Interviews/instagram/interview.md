---
title: Instagram - Mock Interview Transcript
status: active
tags: [hld, mock, instagram, transcript]
---

# Instagram - Mock Interview Transcript

Forty-five minutes. The interviewer is a staff engineer who has run media
platform teams. She will interrupt, push back, and change the numbers on you.

## Act 1 - Opening, Scope, and Requirements

Interviewer: Thanks for joining. We are going to design a photo and video
sharing platform in the style of Instagram. I will be the product and
infrastructure side, you are the backend design side. We have forty-five
minutes. At the end I want a design I could hand to a team. Before you draw
anything, do you have questions for me?

Candidate: Yes, quite a few, and I want to group them so I do not waste our
time. First, scope. Is the feed strictly the accounts a user follows, or is
there discovery of accounts they do not follow?

Interviewer: Assume strictly follow-based for this exercise. Algorithmic
discovery of strangers is a separate ranking problem and I do not want to see
it muddled into the storage design.

Candidate: Understood. That fixes feed generation as a fanout problem rather
than a ranking problem, which I will come back to.

Interviewer: Good, that is the right first instinct.

Candidate: Second, the write path. Do we re-encode video server-side, or does
the client send us final renditions?

Interviewer: Assume we re-encode. Single-rendition video is a toy.

Candidate: Then media processing is a mandatory asynchronous stage and not an
optimization. Noted.

Interviewer: Third, consistency. How strict do you need to be on engagement
counts?

Candidate: I would like to split this into two different questions, because I
suspect the answer is different for each. First, is the like count itself
allowed to be approximate? Second, is the "did I like this" state allowed to
be approximate? I think the second one has to be strongly consistent, because
a double-tapped like that renders as still-liked is a visible bug. The count
can lag by a second and nobody notices.

Interviewer: That is exactly right. Count is eventual, your own action reads
back immediately.

Candidate: And I would solve "reads back immediately" by optimistic client
render plus a per-user liked set that is strongly consistent, rather than by
making the global count strongly consistent.

Interviewer: Yes. Write that down, because it is the key modelling decision in
this whole system.

Candidate: Fourth, availability. What is the target?

Interviewer: Ninety-nine point nine five for the feed, and I will be honest
with you, reads may be degraded but a like that a user taps must never
silently vanish.

Candidate: That rules out a synchronous multi-hop write with a required
quorum across regions for likes. I will use an in-memory counter with async
durable persistence, and I will be explicit that this trades a tiny loss window
for latency.

Interviewer: Reasonable. Now, the numbers I will give you. One and a half
billion accounts, five hundred million daily actives. Sixty percent of daily
actives post at least once a day. Thirty percent of daily actives post a video.
Average photo three megabytes, average thirty-second video twenty megabytes.
Ninety percent cache hit rate is the goal for media delivery. Go.

Candidate: Before I compute, one honest reaction. Thirty percent of daily
actives posting a video is a hundred and fifty million videos a day. At twenty
megabytes that is three terabytes a day of video alone, twelve petabytes a
year, and with three copies for durability that is thirty-six petabytes. I
want to sanity check that against reality before I design around it, because it
changes the storage tier and the cost model by an order of magnitude.

Interviewer: Good pushback. That is the right instinct and the right question.
Keep the sixty percent figure, and treat videos as twenty percent of the media
items posted, not twenty percent of users. And add Stories, which are photos
with a twenty-four hour life.

Candidate: So roughly six hundred million media items per day. Let me work the
numbers.

Candidate: Read and write frequency. I will assume a daily active user opens
the app around thirty times a day, and a feed load happens on most of those
opens. So fifteen billion feed reads a day. Likes, twenty-five per daily
active user per day, so twelve and a half billion likes a day. Comments, six
per user per day, so three billion comments a day. Follows, five per user per
day, so two and a half billion follow actions a day. Media items, six hundred
million a day, of which I will call four hundred and fifty million photos,
one hundred million videos, and fifty million Stories.

Candidate: Now traffic in requests per second. Feed reads, fifteen billion
divided by eighty-six thousand four hundred, is one hundred and seventy-three
thousand requests per second on average. Peak factor for a social app is
roughly three to three and a half times average, so I will design for about
six hundred thousand feed reads per second at peak.

Candidate: Writes. Six hundred million media items a day is six thousand nine
hundred and forty per second average, which with a three times burst is around
twenty-one thousand per second. Likes, twelve point five billion a day is one
hundred and forty-five thousand per second average and roughly five hundred
thousand per second at peak. Comments, three billion a day is thirty-five
thousand per second average, a hundred and twenty thousand at peak.

Candidate: Here is the number I want to lead with, because it reframes the
system. Read to write ratio at the API layer is fifteen billion reads against
six hundred million posts plus twelve point five billion likes plus three
billion comments, which is about sixteen billion writes. So the ratio is
roughly one to one. This is not a read-heavy system at the API layer, which
contradicts how people usually talk about Instagram. The reason is that likes
are writes. The asymmetry is not read versus write, it is that reads are
cheap and idempotent while writes are expensive and have fanout cost.

Interviewer: That is a genuinely good reframe. Keep going.

Candidate: Bandwidth is where the real scale shows up. Ingest: four hundred and
fifty million photos at three megabytes is one point three five terabytes a
day. One hundred million videos at twenty megabytes is two terabytes a day.
Fifty million Stories at three megabytes is a hundred and fifty gigabytes a
day. Total ingest is about three and a half terabytes per day.

Candidate: Three and a half terabytes over eighty-six thousand seconds is forty
megabytes per second average, which is about three hundred and twenty megabits
per second of sustained ingest. Peak at three times burst is roughly one
gigabit sustained, and I will add a factor of five for multipart overhead and
client retries and call it five gigabits of peak upload ingress into the region.

Candidate: Egress is the bigger bill. Fifteen billion feed loads, each showing
roughly one and a half media items, is about twenty-two billion media views a
day. That is two hundred and fifty-seven thousand media views per second. If
the average served rendition is two hundred and fifty kilobytes, total egress
is sixty-four gigabytes per second, which is about five hundred and fourteen
gigabits per second globally. With a ninety percent edge cache hit rate, origin
egress drops to about fifty gigabits per second. That is the number that makes
the CDN non-negotiable, and it is the number I will use to justify regional
read infrastructure.

Candidate: Storage. Three and a half terabytes a day is one point two eight
petabytes a year of originals. Derivative renditions, thumbnails and the video
transcode ladder, add about twenty percent, so one and a half petabytes. With
three copies for durability, about four point six petabytes per year. Over a
five year horizon, roughly twenty-three petabytes. That number alone justifies
tiering: originals and hot renditions on fast object storage, and archival
renditions moved to a colder tier after ninety days.

Candidate: Metadata is three orders of magnitude smaller and this is the
insight that decides the database design. Six hundred million posts a day is
two hundred and nineteen billion posts a year. At one and a half kilobytes per
post row, which includes the caption, media keys, dimensions and counters, that
is three hundred and twenty-eight terabytes a year of post metadata, about a
petabyte with replicas.

Candidate: Like edges are the sleeper. Twelve point five billion likes a day is
four point six trillion a year. At forty bytes per edge row that is one hundred
and eighty terabytes a year if I keep them forever. I will not. I will keep
raw like edges for ninety days, which is forty-four terabytes, and roll
everything older into the counters and into the analytics warehouse. The
"did I like this" answer for an old post is worth less than the cost of
retaining the edge.

Candidate: Comments, three billion a day is one point one trillion a year, at
five hundred bytes per row is five hundred and fifty terabytes a year. I will
keep ninety days hot, about a hundred and thirty-seven terabytes, and archive
the rest. The follow graph is small by comparison: five hundred million users
at an average of three hundred follows each is a hundred and fifty billion
edges, at sixteen bytes is two and a half terabytes, and that one does need to
be permanent because it is a correctness dependency.

Candidate: Total live data-tier volume: post metadata three hundred and
twenty-eight terabytes, comments a hundred and thirty-seven, like edges
forty-four, follows two and a half, plus indexes and overhead. Call it five
hundred and twelve terabytes of live rows, which is roughly one and a half
petabytes with three replicas. That needs sharding, and it is a very different
shape from the twenty-three petabytes of media, which is why the media never
touches the relational tier.

Interviewer: Good. Now tell me your functional requirements in one pass, and
separate them from the non-functional ones so I know you know the difference.

Candidate: Functional: accounts and profiles, authentication, follow and
unfollow, media upload of photos and videos, the feed, hashtag browsing and
trending hashtags, likes, comments, Stories with a twenty-four hour expiry,
push notifications for interactions, content moderation, and the profile grid.

Candidate: Non-functional: feed read at p99 under two hundred milliseconds
globally, feed staleness between five and thirty seconds, like acknowledgement
under one hundred fifty milliseconds, upload accepted in under two seconds
regardless of file size because the bytes bypass our servers, media time to
first byte under three hundred milliseconds, and a content propagation delay for
uploaded media of under thirty seconds. Availability: ninety-nine point nine
five on feed reads. Durability: the system acknowledges an upload only after the
bytes are durably in object storage with at least two copies, and
acknowledges a like only after it is in a durable, replayable stream.
Consistency: strong for follow edges, for the "did I like this" set, and for
comment and post existence; eventual for the feed contents, for like counts, and
for hashtag timelines.

Interviewer: Anything here you want to flag to a junior on your team as a
concept they should go study on its own?

Candidate: Three. Study separately: Count-Min sketch with heavy-hitters detection
for trending hashtags, and study it next to
[[probabilistic-data-structures|Probabilistic Data Structures]], because a plain
hash map of counts will not hold a million distinct hashtags in memory at
five-minute windows. Study separately: LSM-tree write path, because the like
and comment ingest is an append-mostly workload and
[[btree-lsm-hash-index|B-tree versus LSM Tree]] is exactly the argument you
want a new engineer to be able to make. And study separately: the short-video
recommendation pipeline. Reels ranking is a completely separate system with
its own feature store, and pretending it is a feed query is the most common
mistake I see.

## Act 2 - API Design

Interviewer: APIs. Go.

Candidate: Four surfaces, and I will argue for the protocol on each. Client to
service is gRPC or HTTP with JSON, and I would actually use HTTP with JSON for
everything except the internal service mesh where gRPC wins on latency. Let me
show the contracts.

Candidate: One, media upload. The critical decision is that the bytes never
touch an application server.

Candidate: POST /v1/media/uploads
Candidate:   request:  { media_type: "PHOTO"|"VIDEO"|"CAROUSEL", byte_size, content_hash, client_token, caption?, hashtags[] }
Candidate:   response: { upload_id, object_key, upload_url, expires_at, part_size_bytes, required_parts[] }
Candidate:   semantics: client_token is the idempotency key. Repeating the call with the same token returns the same upload_id and does not create a second upload.

Candidate: The client then does multipart upload directly to object storage
using the presigned URLs, chunked so a dropped connection resumes at the last
confirmed part rather than restarting. For a video we do not have the final
codec yet, so the client uploads the source and the server transcodes.
Media processing is a well-known problem area: study separately:
[[media-processing|Media Transcoding and Derivative Generation]], including
transcode ladders, thumbnail generation and how you make the pipeline
idempotent so a retried job does not produce duplicate renditions.

Candidate: Two, finalize.

Candidate: POST /v1/media/uploads/{upload_id}/complete
Candidate:   request:  { parts: [{part_number, etag}], client_token }
Candidate:   response: { post_id, status: "PROCESSING" }
Candidate:   semantics: verifies all parts present and checksums match, then emits a MediaUploaded event. Returns PROCESSING, not PUBLISHED, because the post is not visible until transcode and moderation finish. The client shows a progress state and is told when it is live.

Candidate: Three, feed.

Candidate: GET /v1/feed?cursor={opaque}&limit=20
Candidate:   response: { items: [{post_id, user, media_urls, like_count, comment_count, liked_by_me, created_at}], next_cursor, feed_stale_ms }
Candidate:   semantics: opaque cursor, never offset. Offset pagination is wrong here because a user receiving new posts mid-scroll would see duplicates or gaps as rows shift underneath the cursor. I want to see this come up in the follow-up questions because it is one of the most common real bugs.

Candidate: Four, like and unlike.

Candidate: PUT /v1/posts/{post_id}/likes
Candidate:   response: { liked: true, like_count: 1574820, count_as_of: "..." }
Candidate:   semantics: PUT because liking is idempotent by nature. The response carries the count so the client does not have to guess, and carries count_as_of so the UI can label it if it is stale. DELETE to unlike. Re-liking within a cooldown is a separate product decision, not a technical one.

Candidate: Five, comment, follow, story, hashtag, notification.

Candidate: POST /v1/posts/{post_id}/comments  { body, parent_comment_id? } -> { comment_id, created_at }
Candidate: PUT  /v1/users/{target_id}/followers/me  -> { following: true }   idempotent, same reasoning
Candidate: GET   /v1/users/{target_id}/stories -> ephemeral list, server-filtered by viewer
Candidate: GET   /v1/tags/{tag}?cursor= -> reverse-chronological hashtag timeline
Candidate: GET   /v1/tags/explore      -> trending, from a precomputed list not a live query
Candidate: GET   /v1/notifications?cursor= -> grouped by actor, so 1M likes on one post become one row saying "A and 1,000,000 others liked your post"
Candidate: GET   /v1/users/{id}/media?cursor= -> profile grid

Candidate: On error handling I want a consistent envelope: a machine-readable
code, a human message, a retryable boolean, and a request id that the client
surfaces so support can trace it. And on rate limiting, likes and comment
creation get a per-user token bucket, because the abuse vector here is one
account farming engagement, not volume.

Interviewer: Good contracts. Before you architect, I want to know what
happens with idempotency for likes, because your like write is asynchronous
and I can see a hole.

Candidate: The hole is a double count, not a lost like. The client retries
because the response timed out, the retry is processed, and the user sees their
like count jump by two. My defence is three layers. The per-user liked set is
keyed on (user_id, post_id) with a uniqueness constraint, so a duplicate write
fails at the database and is treated as success. The counter increment is
applied to a per-user-per-post delta inside Redis using a script that only
adds when the set insert succeeded. And the client carries a client_token per
like action, deduplicated at the edge. Idempotency is not one mechanism, it is
a unique constraint plus a conditional increment plus a client token.

## Act 3 - High-Level Architecture

Interviewer: Now the architecture. Draw it.

Candidate: Here is the whole system. I will read it left to right in three
bands: the write path, the read path, and the async backbone.

Candidate: +---------------------------------------------------------------------------------------------+
Candidate: |                                    CLIENTS (mobile, web)                                    |
Candidate: |                          HTTPS / REST + JSON, CDN for all media                             |
Candidate: +--------------------------------+--------------------------------------------------------+
Candidate:                                  |  |
Candidate:                     media bytes   |  metadata RPC
Candidate:                                  v  v
Candidate:                    +---------------------------------------------+
Candidate:                    |            GLOBAL EDGE  (anycast + CDN)      |
Candidate:                    |  TLS termination, WAF, rate limit, cache       |
Candidate:                    |  /v1/media/**  -> cached renditions            |
Candidate:                    |  /v1/feed/**   -> feed edge cache              |
Candidate:                    +--+------------------------------------------+
Candidate:                       |  |
Candidate:        presigned URL  |  authenticated RPC
Candidate:                       v  v
Candidate: +----------------+   +------------------------------------------------+
Candidate: | OBJECT STORAGE |   |  API GATEWAY  (authn, quota, idempotency)      |
Candidate: | photos, videos |   +--+----------+----------+----------+----------+
Candidate: | renditions     |      |          |          |          |          |
Candidate: | 3x replicated  |      v          v          v          v          v
Candidate: +-------+--------+  +------+  +--------+ +--------+ +--------+ +--------+
Candidate:         |          | feed |  | post  | | social | | media  | | notif  |
Candidate:         |          | svc  |  | svc   | | svc    | | svc    | | svc    |
Candidate:         |          +------+  +---+----+ +---+----+ +---+----+ +---+----+
Candidate:         |                 |        |         |         |         |
Candidate:         |            +----+----+ +--+---+ +---+---+ +--------+ +-------+
Candidate:         |            |  CACHE | | shard| | shard | | object  | | shard |
Candidate:         |            |  TIER  | | db   | | db   | | storage | | store |
Candidate:         |            |  Redis | +------+ +-------+ +--------+ +-------+
Candidate:         |            | cluster|
Candidate:         |            +--------+
Candidate:         |
Candidate:         v
Candidate: +---------------------------------------------------------------------+
Candidate: |                    ASYNC BACKBONE  (Kafka topics)                   |
Candidate: | MediaUploaded | MediaProcessed | PostPublished | LikeAdded        |
Candidate: | CommentAdded  | FollowChanged  | ModerationVerdict               |
Candidate: +----+----------+---------+---------+---------+----------+-----------+
Candidate:      |          |         |         |          |          |
Candidate:      v          v         v         v          v          v
Candidate:  media      fanout    hashtag    notification moderation analytics
Candidate:  workers    workers   indexer   service     service    pipeline
Candidate:  (thumbs,   (feed     (tag      (push,     (human     (warehouse,
Candidate:   transcode,  store)   timelines) badge)     review)    BI)

Candidate: The one thing I want you to notice is that there is no synchronous
path from a client to a transcoder and no synchronous path from a like to a
notification. Both of those are the obvious places to accidentally build a
distributed monolith, and both of them are why [[asynchronous-processing|Async
Processing]] is a load-bearing part of the design and not a stylistic choice.

Interviewer: Walk me through the upload path end to end, with hops.

Candidate: Step one, the client calls the media service for an upload ticket.
The media service authenticates, checks quota, generates an upload id, computes
the object key, and returns presigned multipart URLs.

Candidate: Object key design matters more than people expect. I would use
`raw/{user_id}/{yyyy}/{mm}/{dd}/{upload_id}/{content_hash}` rather than
`{upload_id}` alone. Hashing the content gives me deduplication for free, the
prefix groups a user's uploads for lifecycle rules, and a date prefix lets me
expire old raw files without scanning the whole bucket.

Candidate: Step two, the client uploads parts directly to object storage. Our
servers see bytes zero times. That is the single most important scaling
decision in this system, because at five gigabits of peak ingest a proxy tier
would need to be enormous and would still add latency to every byte.

Candidate: Step three, the client calls complete. We verify part count,
ETags and checksums, copy or move the object into its final prefix, and write
an upload status row.

Candidate: Step four, and this is where I use [[outbox-pattern|the outbox
pattern]] rather than just publishing to Kafka, because publishing is the step
that fails silently. I write the post row and an outbox row in one local
transaction, and a relay publishes to Kafka and marks the outbox row sent. If
the relay dies, the row is still there. Publishing directly to Kafka inside a
transaction is the classic dual-write bug.

Candidate: Step five, the media pipeline consumes MediaUploaded. Per job: strip
EXIF because location metadata is a privacy leak, run virus and format
validation, generate a blurred placeholder so the client can render instantly
before bytes arrive, generate thumbnails at 150x150 for the grid and 640x for
the feed, and for video build a transcode ladder at 240p, 480p, 720p and 1080p
plus a DASH or HLS packaging so we can adapt to bandwidth.

Candidate: Content moderation runs in two places and both are needed. A
synchronous fast path during processing, using perceptual hash matching
against known-bad media and a classifier, which must finish inside the thirty
second propagation budget. And a slower human review queue for anything the
classifier is unsure about. I would never make the human review step
synchronous, because a reviewer takes minutes and the upload would appear
broken.

Candidate: Step six, when all renditions exist and moderation is clear, the
pipeline emits PostPublished. The post service flips the post to visible,
invalidates the author's own feed, and the fanout workers begin.

Candidate: Step seven, the CDN. When a rendition is written, we do not need to
invalidate anything because the URLs are content addressed, so a new rendition
is a new URL and is simply not cached yet. The first request to each new URL
is a miss that I absorb at origin, which is why I pre-warm the author grid's
newest three items on publish rather than waiting for the author's own reload.

Candidate: Total time to live is upload duration plus about eight to twenty
seconds of processing, dominated by video transcode. For a photo I would target
under two seconds, which means synchronous processing is tempting, but I would
still keep it asynchronous so that one slow transcode job cannot delay the post
metadata write.

## Act 4 - Request and Data Flow

Interviewer: Feed read. Every hop.

Candidate: One, the client's request hits the global edge. The edge checks auth
token validity, applies per-user rate limits, and checks the regional feed edge
cache.

Candidate: Two, on a miss, the request goes to the feed service. The feed
service checks the user's own hot feed in Redis. That is a single key lookup,
`feed:{user_id}`, a sorted set of post ids scored by rank.

Candidate: Three, on a miss of that too, the feed service reconstructs. It
reads the user's follow graph, which I keep in a dedicated follow-graph store
sharded by follower id, then pulls the most recent post ids for each followee.
Here is the design decision I want to be explicit about: I never fan out a
follow list of eight hundred followees into eight hundred database queries. I
keep a per-user inverted list, `recent_posts:{user_id}` in Redis, a sorted set
of that user's latest fifty post ids. Merging eight hundred small sorted sets
is cheap. That is the pull model.

Candidate: Four, that merged candidate set, say six hundred ids, goes to a
ranker that scores by recency, affinity and engagement rate, and returns the top
two hundred. I do not persist this; I persist the top one hundred and a half in
Redis with a short TTL.

Candidate: Five, I hydrate the top twenty from a batched post-metadata read.
I never do a per-item query, I use a single multi-get or a datastore batch so
that twenty items is one round trip. That is a [[bottleneck-identification|
N+1 problem]] and it is the most common cause of slow feeds in practice.

Candidate: Six, I return the cursor. The cursor is the rank score of the last
item plus a tiebreaker, not an offset, so inserts at the head do not shift
anything.

Interviewer: Now the like path, and I want to see where you give up
synchronous durability.

Candidate: One, the like service checks a Redis set `liked:{user_id}:{post_id}`
for the dedup, and if absent inserts with a conditional add. If the insert
returns false, the user already liked it and we return the current count
unchanged.

Candidate: Two, we increment a sharded counter. I am going to explain why it is
sharded now, because it is the answer to your hot creator question later. The
naive design is a single counter key per post. The production design is
`cnt:{post_id}:{user_id mod 32}`, thirty-two independent sub-counters, so a
post taking five hundred thousand likes per second spreads across thirty-two
keys and, with pipelining, across many Redis shards. Reads sum thirty-two keys,
which is a cheap fan-in, or we maintain a periodically recomputed total.

Candidate: Three, we return success immediately with the current approximate
count. We have not written to the database. We emit LikeAdded to Kafka, which
is our durability boundary.

Candidate: Four, a consumer writes the edge to the sharded likes table and
applies the delta to the authoritative post counter, idempotently keyed on
event id. If this consumer is down for thirty seconds we lose nothing, because
the Kafka partition retains it and the client already got its acknowledgement.

Candidate: Five, the notification service consumes a separate topic. I keep
notifications on their own topic, not the LikeAdded topic, because notification
fanout to a five hundred million follower creator is orders of magnitude more
expensive than the like itself and must not hold up the counter.

Interviewer: Hold on. You just told the user the like succeeded and then
asynchronously wrote it. Walk me through what happens when the Kafka broker
disk fills at 2am.

Candidate: Then the produce call fails. Three responses. First, the like
service fails the write loudly and returns a 503 with retryable set, because
the alternative is a silent lie. Second, I keep a small synchronous write to a
local append-only WAL in the like service as a backstop, replayed when Kafka
recovers, so the window of acknowledged-but-lost likes is a few hundred
milliseconds rather than thirty minutes. Third, and this is the operational
answer, Kafka is provisioned with headroom and the produce path has a
[[circuit-breaker|circuit breaker]] plus a local disk queue, so a broker outage
degrades to a slower, local-queue-backed like path rather than to data loss.

Interviewer: Fair. I have pushed on the thing I would push on. Next question:
the notification path for a creator with a huge audience.

Candidate: That is where the follow graph's out-degree destroys a naive design.
Five hundred million followers times one notification is five hundred million
rows and five hundred million push calls for a single post.

Candidate: Three mitigations. First, the notification service does not consume
the raw interaction topic. It consumes a pre-aggregated topic produced by a
rollup worker, so it receives one event per (post, actor-group, minute) rather
than one per interaction. Second, at read time notifications are grouped by
actor, so the user's notification list is one row saying a hundred thousand
people liked your post, not a hundred thousand rows. Third, push delivery goes
through a per-user coalescing queue with a per-minute window, so a user
receiving two hundred notifications in a minute gets the three most important.

Candidate: And the fourth is a product one: above a follower threshold,
individual notifications are not generated at all, only aggregate ones. That is
a batching decision made in the design rather than an engineering patch, which
is why requirements clarification matters.

## Act 5 - Database Design

Interviewer: Schemas. Make them concrete enough that I can review them.

Candidate: Four core tables plus two aggregate tables. Everything below is
logically defined; the physical sharding column is called out per table because
it is different for each one.

Candidate: CREATE TABLE posts (
Candidate:   post_id       BIGINT      NOT NULL,   -- snowflake: time-sortable, collision-free
Candidate:   user_id       BIGINT      NOT NULL,
Candidate:   media_type    TINYINT     NOT NULL,   -- 1 photo, 2 video, 3 carousel
Candidate:   object_prefix VARCHAR(255) NOT NULL,   -- base key; renditions derived by suffix
Candidate:   thumb_key     VARCHAR(255) NOT NULL,
Candidate:   placeholder_key VARCHAR(255) NOT NULL,
Candidate:   caption       VARCHAR(2200),
Candidate:   like_count    BIGINT      NOT NULL DEFAULT 0,
Candidate:   comment_count BIGINT      NOT NULL DEFAULT 0,
Candidate:   status        TINYINT     NOT NULL,   -- 0 processing, 1 published, 2 blocked, 3 failed
Candidate:   created_at    TIMESTAMP(6) NOT NULL,
Candidate:   PRIMARY KEY (post_id),
Candidate:   KEY idx_user_created (user_id, created_at DESC)
Candidate: );
Candidate: Shard: HASH(post_id) into 512 shards, 2 read replicas each.
Candidate: Why post_id and not user_id: post_id is globally unique and random, so every write and every point read hits exactly one shard, and there is no hotspot on any creator. I pay for it with the idx_user_created secondary index, which I implement as its own table sharded by user_id, so listing a user's posts is a single-shard range scan instead of a scatter-gather.

Candidate: CREATE TABLE post_user_index (
Candidate:   user_id    BIGINT NOT NULL,
Candidate:   created_at TIMESTAMP(6) NOT NULL,
Candidate:   post_id    BIGINT NOT NULL,
Candidate:   PRIMARY KEY (user_id, created_at DESC, post_id)
Candidate: );
Candidate: Shard: HASH(user_id). This is the denormalization that lets me keep
Candidate: post_id sharding and still do a fast profile grid.

Candidate: CREATE TABLE likes (
Candidate:   post_id    BIGINT NOT NULL,
Candidate:   user_id    BIGINT NOT NULL,
Candidate:   created_at TIMESTAMP(6) NOT NULL,
Candidate:   shard      TINYINT NOT NULL,        -- which of the 32 counter sub-shards applied it
Candidate:   PRIMARY KEY (post_id, user_id)
Candidate: );
Candidate: Shard: HASH(post_id) into 4096 shards. Post_id because the primary
Candidate: access pattern is "all likes on this post", and because the hot-post
Candidate: problem is handled by sub-sharding the post_id itself with a suffix,
Candidate: not by splitting the table differently.

Candidate: I need to be precise about the hot-post case because you will ask.
Hashing post_id into 4096 shards does not help when one post is hot, because
every row has the same post_id and therefore lands on the same shard. The fix
is to make the shard key (post_id, user_id mod 32) and store thirty-two
partial counts per post. Query "all likes on this post" then becomes thirty-two
ranged scans, and "count" is a sum of thirty-two counters. Trade-off: I have
made the write path better and the read path slightly worse, and for a like-heavy
workload that is the right direction.

Candidate: CREATE TABLE comments (
Candidate:   comment_id        BIGINT NOT NULL,
Candidate:   post_id           BIGINT NOT NULL,
Candidate:   user_id           BIGINT NOT NULL,
Candidate:   parent_comment_id BIGINT NULL,
Candidate:   body              VARCHAR(2200) NOT NULL,
Candidate:   like_count        BIGINT NOT NULL DEFAULT 0,
Candidate:   status            TINYINT NOT NULL,
Candidate:   created_at        TIMESTAMP(6) NOT NULL,
Candidate:   PRIMARY KEY (comment_id),
Candidate:   KEY idx_post_created (post_id, created_at DESC)
Candidate: );
Candidate: Shard: HASH(post_id). Same reasoning as likes.

Candidate: CREATE TABLE follows (
Candidate:   follower_id BIGINT NOT NULL,
Candidate:   followee_id BIGINT NOT NULL,
Candidate:   created_at TIMESTAMP(6) NOT NULL,
Candidate:   PRIMARY KEY (follower_id, followee_id),
Candidate:   KEY idx_followee (followee_id)
Candidate: );
Candidate: Shard: HASH(follower_id) for the "who do I follow" read, and
Candidate: HASH(followee_id) in a mirrored table for "who follows me" and, more
Candidate: importantly, for fanout. I keep two tables, not one, because the two
Candidate: queries have opposite shard keys and a single table would force a
Candidate: scatter-gather across every shard for the followers count.

Candidate: CREATE TABLE post_counters (
Candidate:   post_id     BIGINT NOT NULL,
Candidate:   sub_shard   TINYINT NOT NULL,
Candidate:   like_delta  BIGINT NOT NULL DEFAULT 0,
Candidate:   updated_at  TIMESTAMP(6) NOT NULL,
Candidate:   PRIMARY KEY (post_id, sub_shard)
Candidate: );
Candidate: This is the compaction target. Raw like edges are retained ninety days
Candidate: for audit and abuse detection; after that the edges are dropped and
Candidate: this table plus the analytics warehouse is the permanent record.

Candidate: CREATE TABLE user_liked (
Candidate:   user_id BIGINT NOT NULL,
Candidate:   post_id BIGINT NOT NULL,
Candidate:   created_at TIMESTAMP(6) NOT NULL,
Candidate:   PRIMARY KEY (user_id, post_id)
Candidate: );
Candidate: Shard: HASH(user_id). This table exists purely to answer "did I like
Candidate: this" in one lookup, and its existence is why the global count is
Candidate: allowed to be eventual. I pay for one extra write per like, and I
Candidate: would make that write synchronous because it is a small point write on
Candidate: a key that is already hot in the user's own page.

Candidate: On Stories, I use a different store entirely. Story rows carry a
Candidate: hard expires_at at creation plus twenty-four hours, the object has a
Candidate: lifecycle rule that deletes it at twenty-five hours as a backstop, and
Candidate: the read path filters on expires_at in the query rather than in
Candidate: application code. Stories also get their own feed pipeline and their
Candidate: own cache namespace, because the cardinality and the access pattern
Candidate: are different and mixing them would let a Story request evict hot feed
Candidate: keys.

Candidate: On hashtags, a hashtag is normalized to lowercase with the hash
Candidate: stripped and Unicode-normalized, then a hashtag_post table keyed
Candidate: (tag_id, created_at DESC) gives reverse-chronological, and a
Candidate: separate Elasticsearch index serves tag and caption search. I am
Candidate: aware this is denormalization and I am doing it deliberately:
Candidate: study separately: [[normalization-vs-denormalization|Normalization
Candidate: versus Denormalization]] and be able to justify when the tag timeline
Candidate: is the source of truth and when the search index is.

## Act 6 - Caching

Interviewer: Caching. And I want to hear about stampedes, because that is the
failure that actually pages you.

Candidate: Three cache layers, with different jobs and different failure
behaviours.

Candidate: Layer one is the CDN, for media. Ninety percent hit rate target.
Content-addressed URLs mean a rendition can never be stale, so the only invalidation
problem is eviction. For media I use a long TTL, one year, and no invalidation
at all.

Candidate: Layer two is the regional feed edge cache, sitting in front of the
feed service, keyed by user id with a short TTL of thirty to sixty seconds. This
absorbs the bursty re-open pattern where a user pulls to refresh six times a
minute and every refresh would otherwise be a full feed build.

Candidate: Layer three is the Redis cluster, and I split it into separate
clusters per access pattern because they have opposite eviction needs. The feed
cluster is volatile, TTL-driven, sized to hold roughly the hot working set. The
counter cluster is volatile and must never evict, so it is sized generously and
configured with noeviction. The session and liked-set cluster is durable-feeling
and small. One shared cluster with one eviction policy is a latent outage, and
this is exactly what [[caching|Caching]] and [[redis|Redis]] tutorials do not
tell you.

Candidate: Now stampedes. The failure is: five hundred thousand followers of a
creator all have `recent_posts:{creator_id}` cached, and it expires at the same
instant, and every one of them misses and rebuilds it. Mitigations, in the order
I would actually implement them.

Candidate: One, jittered TTL. Never a constant. Base TTL times a random factor
between 0.9 and 1.1, so keys do not align.

Candidate: Two, stale-while-revalidate. Store the value with a hard TTL and a
soft TTL. Serve the stale value while a single background job refreshes. For a
feed, a value thirty seconds old is indistinguishable from fresh.

Candidate: Three, single-flight. A per-key mutex so that on a miss, one caller
does the work and the rest wait on a promise, using a short wait with a fallback
to serving stale. This is the direct defence against a thundering herd.

Candidate: Four, logical expiry. Store an explicit expired_at field inside the
value and let the reader decide. Popular keys get proactively refreshed by a
background warmer driven by access frequency, so they never logically expire at
all. Popularity-driven warming is the right tool here, not blind timed warming,
and it is what study separately: [[cache-warming|Cache Warming]] should be
framed around.

Candidate: Five, negative caching with a short TTL, so a lookup for a deleted or
not-yet-processed post does not hammer the database during the window before
moderation finishes.

Candidate: Six, the deepest defence: do not let the key expire at all for the
hottest keys. A creator with five hundred million followers has a
`recent_posts` key that is refreshed by a cron job every thirty seconds and has
no TTL. The cost of a hot key is bounded and known; the cost of a stampede is
unbounded and unknown.

Interviewer: Cache stampede, replica lag, and traffic doubling in one
interview. What happens to your feed cache hit rate when a post goes viral?

Candidate: Two competing effects and the answer is not obvious. Viral means
read traffic spikes, which should improve hit rate because the same key is hit
more. But viral also means many new distinct readers, users who are not in the
warm set, and their keys are cold. Net effect in my experience is hit rate
drops at first, from say ninety-five percent to eighty-five percent, and
recovers within a few minutes as the warmer catches up. So I pre-warm
aggressively on virality signal, which I get from the interaction rate on a
single post crossing a threshold.

## Act 7 - Messaging and Asynchrony

Interviewer: What is on the bus, and what guarantees do you need?

Candidate: Kafka, in three logical groups. Media events, social events, and
derived or analytics events. Everything is keyed and everything is idempotent.

Candidate: Guarantees I actually rely on: at-least-once delivery, per-partition
ordering by key, and consumer idempotency. I do not rely on exactly-once
delivery from the broker because even a transactional producer does not make the
side effects outside Kafka transactional, and the moment you write to a
database or a push provider the illusion breaks. The rule is: exactly-once
processing is achieved with idempotent consumers, not with broker features.

Candidate: For a consumer, idempotency means a processed-event marker, or for
database writes, an upsert keyed on the event id. The consumer-lag metric is
part of my alerting, because lag is the leading indicator of a user-visible
failure and it degrades long before an error rate does.

Candidate: Ordering I need: events for one post must be ordered, so I partition
by post_id. Events for one user must be ordered for the feed, so I partition by
user_id. Where a single event needs both, the producer writes it once to the
post_id partition and the downstream fanout worker re-partitions by user_id.
That re-partition is a real cost and I only do it for the feed path.

Candidate: Poison messages go to a dead-letter topic with the full payload and
the failure reason, and they page nobody until the dead-letter volume crosses a
threshold, because a single malformed event should not wake anyone.

Interviewer: You said media processing must finish inside thirty seconds. What
if the queue backs up to ten minutes of work?

Candidate: Three responses, in order. First, priority lanes: I classify jobs as
interactive, user-visible, or background, and interactive jobs get a dedicated
consumer group with reserved capacity, so a flood of background transcodes
cannot starve a user watching their own post upload. That is
[[bulkhead|Bulkhead]] applied to a queue rather than to a thread pool.

Candidate: Second, autoscaling on queue depth rather than on CPU, because media
workers are I-O and CPU-bound in bursts and CPU-based autoscaling lags the
backlog by minutes. Scale on oldest-message-age, which is the number that maps
to user pain.

Candidate: Third, load shedding at admission. If oldest-message-age exceeds one
hundred twenty seconds, I degrade gracefully: drop from the 1080p rendition to
720p only, keep the placeholder and thumbnail, and mark the post published
with reduced quality rather than blocking it. A slightly blurry post beats a
stuck spinner. I keep a local queue inside the service as a shock absorber so
a broker blip does not immediately become a backlog.

Candidate: And a fourth that is often missed: idempotent jobs. A retried
transcode must overwrite, not append, and the job id is derived from
`post_id:rendition` so a duplicate is a no-op.

## Act 8 - Sharding, Replication, and Consistency

Interviewer: Partitioning. I have heard your shard keys. Now tell me what
happens when you outgrow them.

Candidate: Right now: 512 shards for posts, 4096 for likes and comments, 256
for follows, both directions. Each shard is a primary with two read replicas in
the same region. That is roughly one and a half petabytes across about eight
thousand primaries, which is a hundred and eighty gigabytes per primary. That
is a healthy size. A shard should be a few hundred gigabytes, not a few
terabytes, because resharding a primary requires copying its data.

Candidate: Growth plan: the shard count is virtual, from day one, even though
the physical count is 512. Routing is a directory that maps key to shard, and
the directory is what I split. When shard s is too big, I split it into s0 and
s1, dual-write both during the copy, backfill, then flip. This is
[[shard-rebalancing|Shard Rebalancing]] and it is the difference between a
system that scales for years and one that needs a migration weekend every
eighteen months.

Candidate: Read/write split within a shard: the primary serves writes and read-
your-own-writes, both replicas serve global reads with a configurable lag
threshold, and a rare replica that is too far behind is removed from the read
set by the health check rather than being trusted.

Interviewer: Replication and replica lag. A user taps like, gets a count of
1574820, then refreshes the post detail from a replica and sees 1574815. Walk me
through why and what you do.

Candidate: Because the like went to Redis sub-counter 7, the authoritative
delta is still in Kafka, and the replica of the post row has not replayed it
yet. Replica lag here is seconds, not milliseconds, because the write path is
asynchronous by design.

Candidate: Three fixes, and I would do all three. One, the post detail read for
a user who just interacted routes to the primary for that post, using a
short-lived read-your-writes token returned by the like API. Two, the counter
read is served from Redis, not from the post row, so the post row is only the
durable eventually-correct version. Three, the like response returns the count,
so the client never re-reads to discover it.

Candidate: What I will not do is make the global count strongly consistent. If I
made the counter synchronous, a post with five hundred thousand likes per second
becomes a five hundred thousand per second serialized write to one row on one
primary, and that primary is the answer to your next question.

Interviewer: Go ahead then. Your primary database dies. Right now. Post detail
and likes. Walk me through the first ninety seconds.

Candidate: Assume the posts primary and its two replicas are in one shard, and
the primary process is gone.

Candidate: Zero to five seconds: replica health checks detect the failure. There
is a short window where a minority of requests fail. If the shard is
replicated across availability zones, which it is, the replicas are up.

Candidate: Five to fifteen seconds: the shard's role is promoted, a new primary
is elected. I want to use explicit failover with a [[quorum|Quorum]] or a
consensus-based role manager rather than manual promotion, because at this scale
nobody is watching a dashboard. The key safety property is fencing: the old
primary must be stopped from accepting writes before the new one is promoted,
or I have a split-brain with two writers on one shard. Fencing is a lease with
an epoch number, and the epoch is checked on every write. If you remember one
thing about failover it is this: a replica that was down for ten seconds and
rejoins with data the new primary does not have will silently lose writes, so
fencing plus a re-sync-from-primary, not replication rejoin, is the recovery
path.

Candidate: Fifteen to forty-five seconds: reads and writes resume. New writes
go to the promoted primary. Feeds keep working because they are served from the
Redis cluster, which is a separate system that did not fail, and that is an
accidental but important benefit of cache-aside.

Candidate: What about the likes that were acknowledged in the last thirty
seconds? They are in Kafka, durable, and the counter rebuild reads from Kafka
replay, so the counter self-heals. That is the payoff of making the durable
write the event and the database a projection of it.

Candidate: And the honest caveat: with asynchronous replica promotion, writes
accepted by the old primary in its final second can be lost. My RPO is
therefore "up to a few seconds of likes", and my RTO is about thirty seconds. I
am comfortable with that for likes and not comfortable with it for follows, which
is why the follow graph is synchronous and replicated separately.

Interviewer: Consistency. Make me a table of what is strong and what is
eventual, and defend the boundaries.

Candidate: Strong: follow and unfollow edges, because the user must see their
own action and must not see a follow they just revoked. Strong: post and
comment existence, because a 404 on a post you just made is unacceptable.
Strong: the "did I like this" set, because double-tapped-liked-still-liked is a
visible bug. Strong: the media object, whose acknowledgement means the bytes
are in storage with two copies.

Candidate: Eventual, with a stated bound: feed contents at five to thirty
seconds, like counts at about one second, hashtag timelines at about one second,
trending hashtags at five minutes, notifications at a few seconds, media
processing completion at about twenty seconds.

Candidate: The rule behind the table: strong on the data the acting user
perceives as their own action, eventual on everything that is aggregate or
derived. Aggregates are the cheapest thing in the system to make eventual
because the next write overwrites the error.

Candidate: And the deeper principle, which is worth more than the table: choose
per-relation, not per-service. A service is not consistent or eventual; a
relation is. My like service holds three relations with three different
guarantees. If you say "the like service is eventual" you have said nothing
useful.

Interviewer: Push back on one thing. Your follow edge is strong. But the feed
is built from the follow graph. If follow is strong and feed is eventual, when
does a new followee's post show up in my feed?

Candidate: After fanout completes, which is one to five seconds in the push
model, and after the follower's own cache invalidation. So there is a window
where I have followed someone, I look at my feed, and their post from the last
hour is not there. That is a real, visible product bug, and there are three
options.

Candidate: One, accept it and put a "refresh" affordance. Two, on follow, do an
immediate synchronous on-read pull for the followee's recent posts, bounded to
the last ten, and inject them into the materialized feed. Three, mark new
follows as "hot followees" with a shorter fanout SLA, ten seconds, at the cost
of prioritizing them in the fanout queue. I would ship two and three, and I
would make the follow API return a boolean so the client can show the state
honestly.

## Act 9 - Availability, Fault Tolerance, and Bottlenecks

Interviewer: Bottlenecks. Where does this break first?

Candidate: Ranked.

Candidate: One, the like counter write path at peak. Five hundred thousand likes
per second across thirty-two sub-counters, with a fanout of follow-graph reads
and notification rollups. This is the tightest resource and it is the one that
degrades before anything else.

Candidate: Two, feed read QPS. Six hundred thousand per second at peak. This
is a Redis problem, not a database problem, and it is solved by the regional
edge cache absorbing re-opens. If the edge hit rate falls from ninety-five to
eighty percent, that is an extra ninety thousand origin requests per second and
Redis becomes the bottleneck.

Candidate: Three, media processing queue depth, which is elastic but has a lag
that is user-visible.

Candidate: Four, hashtag timelines for a global event, where a single tag's
partition takes a disproportionate share of the index write load. I mitigate
with a dedicated high-throughput partition set for suspected hot tags, plus
sampling above a rate threshold, plus a strong read cache on the tag timeline.

Candidate: Five, the follow-graph fanout for a huge creator, which I will take
as its own section.

Candidate: Six, notification volume on virality.

Interviewer: Take the huge creator now. Five hundred million followers, one
post. Go.

Candidate: This is the question the whole system is designed around, so let me
lay out the three feed models first and then the fix.

Candidate: Model one, fanout on write. On publish, write a feed entry for every
follower. For a normal creator with a thousand followers that is a thousand
writes, trivial. For our creator it is five hundred million writes for a single
post. At two hundred bytes per entry that is a hundred gigabytes of feed
storage for one post, and at a fanout throughput of, say, two million entries
per second it takes two hundred and fifty seconds. The post is live long before
the fanout finishes, so followers see a partial feed. And if the user has five
hundred million followers, their out-degree is the entire problem. So pure
fanout-on-write is out for high-out-degree users.

Candidate: Model two, fanout on read. Do not materialize. At read time, fetch
recent posts from each followee and merge. This is exactly what I described in
the feed flow. It is O(follows) per read, and with eight hundred followees
that is eight hundred sorted-set reads. That is survivable with a good local
cache but it is eight hundred operations, and if a user follows five thousand
people it is not survivable at six hundred thousand reads per second.

Candidate: Model three, hybrid. And the threshold is the whole design. If a
creator has fewer than ten thousand followers, fan out on write into each
follower's materialized feed. If more than ten thousand, do not materialize;
mark the edge as pull-only, and at read time merge that creator's recent posts
from a single cached sorted set.

Candidate: The pull-only merge is the key. A five hundred million follower
creator becomes one extra sorted-set read per feed request, and the same sorted
set is served from a dedicated hot Redis cluster. It is one key, not five
hundred million writes.

Candidate: Three additions that make it better. One, for the top ten thousand
creators, precompute. A scheduled job regenerates that creator's merged recent
set every thirty seconds into a chunk keyed by creator_id, and all five
hundred million followers read the same precomputed chunk. Read cost per
follower drops to a pointer chase, and freshness is thirty seconds, which for a
creator with high posting frequency is unnoticeable. Two, cap materialized fanout
with a hard limit, so a user with a million followers is pull-only by rule, not
by heuristic. Three, use virtual nodes in the consistent hash for the feed
cluster so that a single enormous creator's key does not own an entire node,
which is what [[virtual-nodes|Virtual Nodes]] are for.

Candidate: And the fourth is the one people forget: the pull-only creator's
posts must not vanish from a follower's ranked feed just because they are not
in the materialized set. So the merge at read time is a real ranked merge, not a
concatenation, and I cap pull-only contributions to the top fifty slots so they
cannot crowd out the rest.

Interviewer: Good. Now the viral-post notification path and the viral-post
comment path. Same math, different object.

Candidate: Comments are different from likes because they must be read in order
and are user-generated text, so they go to the same-shard comments table, and a
single hot post puts a hundred and twenty thousand comment writes per second on
one shard. I mitigate with the same sub-shard suffix trick on the shard key, so
a post's comments live on thirty-two shards and the "first page of comments"
read is a thirty-two-way fan-in of small ranges, which is a few milliseconds.
The alternative is rejecting writes, which I would only do above a hard abuse
threshold.

Candidate: Notifications I already covered: pre-aggregated events, grouped at
read, coalesced push.

Interviewer: Fault tolerance. Walk me through four failure scenarios, and be
specific about what the user sees.

Candidate: Scenario one, region loss. A whole region goes down. I have
multi-region reads: content is served from the nearest healthy region with
replicated metadata, media comes from the CDN which is already global, so the
user sees a normal feed. Writes: I use a single home region per shard, so
uploads and likes in the failed region fail or are queued. My choice for a
consumer social product is to accept the write unavailability rather than pay
for multi-writer, and to fail loudly with a retryable error, plus a local
durable queue on the client that retries with backoff. The alternative,
[[multi-region-models|multi-region active-active]] with conflict resolution, I
am rejecting because like counts and follow edges are last-writer-wins by
nature and that would make the counts lie in a way users can see.

Candidate: Scenario two, the CDN origin is overloaded because of a miss storm.
I lower TTLs on media to force more edge caching, I serve lower renditions
only, I add a dedicated read-only origin replica pool, and I put a rate limit
on uncached media fetches. User sees slightly lower resolution photos, not
errors.

Candidate: Scenario three, the object store write succeeds but the event is
lost, or the media pipeline writes a corrupt rendition. For the first, I
reconcile: a periodic inventory job compares the object store prefix manifest
against the post table and re-emits missing MediaUploaded events. For the
second, each rendition carries a checksum verified at read time by the CDN edge
origin, and a failure quarantines the rendition and falls back to the original,
so the user sees a photo instead of a broken image.

Candidate: Scenario four, a dependency is slow rather than down. The post
service calls the follow service, and the follow service is at p99 four seconds.
Synchronous, the feed dies even though the follow data is fine. Every
cross-service call needs a timeout, a circuit breaker, and a fallback. For the
feed, the fallback is to serve a slightly stale materialized feed from Redis
with no ranking refresh, which is a good outcome. This is the single most
important reliability pattern in this design and I would not ship without it
[[resilience-patterns|Resilience Patterns]].

Interviewer: What is your monitoring story? Give me the signals, not the tools.

Candidate: Golden signals, and they are different per subsystem. For the edge
and services: latency percentiles per route, not averages, error rate split by
class, saturation on connection pools and Redis memory, and traffic. For the
async backbone: consumer lag per consumer group, and oldest-message-age, which
is the one that maps to user pain. For the feed: cache hit rate as a first-class
metric, and the p99 feed build time decomposed into its stages, follow-graph
read, merge, rank, hydrate.

Candidate: For correctness, and this is the part teams skip: a reconciliation
job that compares the sum of counter sub-shards against the durable edge count
for a sample of hot posts, alerting on divergence above a threshold. That
catches double counts from a retry bug, which no error rate will ever show you,
because a double count is not an error, it is a wrong number that looks fine.

Candidate: Distributed tracing on the feed path end to end, including the Redis
lookups, because a feed at p99 six hundred milliseconds is almost always one
slow sub-call and not slow compute.

Interviewer: Scaling. You have ten times the users and the same team.

Candidate: The scaling is mostly already in place, which is the point of
choosing it early. Media scales by CDN and object storage, which is elastic and
managed. Metadata scales by adding shards to the directory, which is a routing
change plus a background copy. Feed reads scale by adding Redis nodes and
regional edge cache points of presence.

Candidate: What does not scale linearly and needs an explicit answer is fanout.
At ten times users, the number of feed entries materialized per post scales with
total follow edges, which scales superlinearly if growth is user-led rather than
creator-led. So the hybrid threshold has to be re-tuned as the platform grows,
and I would expect to move from a count threshold to a cost-based threshold:
fan out if the estimated fanout cost is below a per-post budget, where cost is
in entries written and the budget is derived from the total feed write capacity
divided by the post rate.

Candidate: The other thing that changes at ten times is the notification
rollup, because the rollup worker itself needs sharding and its own lag
monitoring, and moderation, because a linear increase in uploads needs a linear
increase in classifier throughput, and classifiers are GPUs which are the most
expensive thing in the company.

Interviewer: Trade-offs. Give me the four you agonized over most and what you
gave up.

Candidate: One, feed model. I gave up exactness and immediacy for cost. Fanout
on write gives the fastest read and the most expensive write; pull gives the
cheapest write and the slowest read. Hybrid gives me both, at the cost of two
code paths, two sets of bugs, and a threshold that I will have to re-tune. I
chose it because the cost difference is roughly a hundred times.

Candidate: Two, like durability. I chose asynchronous durability with a Kafka
boundary over synchronous quorum writes, and I gave up a strict guarantee in
exchange for sub-150-millisecond acknowledgement at five hundred thousand per
second. The thing I protected with that trade is the user's own like state,
which stayed strong, so the user-visible contract held even though the
infrastructure one did not.

Candidate: Three, counters versus edges. Keeping every like edge forever is
truthful and unbounded, at one hundred and eighty terabytes a year. Rolling up
after ninety days is lossy for analytics but bounded at forty-four terabytes.
I chose the rollup, and the cost is that I cannot reconstruct a like graph older
than ninety days, which matters for fraud investigations, so I push the edges to
the analytics warehouse rather than deleting them outright.

Candidate: Four, store choice. A relational store for posts, follows and
comments because of joins, transactions and the follow-graph correctness
dependency, and a wide-column or document store for feeds because of the
write-heavy sorted-set access pattern and because I do not want feed generation
competing with post writes for the same primary. Cost is operational: two
stores, two on-call skills, two backup paths. I rejected a single store because
the access patterns have opposite shapes, and I rejected full polyglot because
every additional store is permanent operational debt.

Interviewer: Last technical question, and I picked it because most candidates
miss it. A user requests deletion under a data erasure request. What happens to
their eleven years of posts, their media, and their comments on other people's
posts?

Candidate: That is the hardest correctness problem here and it is not an
availability problem, so I will treat it as a data-lifecycle problem. Four
pieces. First, the user row and credentials go, and I use a soft delete plus a
tombstone so that a post written after the deletion request by an in-flight
request does not resurrect the account. Study separately:
[[soft-delete-audit-tables|Soft Delete and Audit Tables]] for why a hard delete
is the wrong tool here.

Candidate: Second, the follow graph. I delete the follower side immediately and
the followee side asynchronously through the event stream, because the followee
side touches many shards and must be a fanout job, not a request-path write.

Candidate: Third, the media. This is the long tail. Objects are content
addressed, so I cannot delete by key without knowing every derived rendition key,
so I store a manifest per post listing every derivative, and the erasure job
deletes the manifest, not the guessed prefix. I also need the CDN purge for
every rendition URL, and if the purge fails the content is still reachable by
URL, which is a real exposure, so I do not rely on the purge alone, I overwrite
or crypto-erase. Study separately: [[encryption-and-keys|Encryption and Keys]],
specifically crypto-shredding, which is the only way to make deletion of a
multi-terabyte media estate fast.

Candidate: Fourth, their comments on other people's posts. This one is a genuine
product and legal decision, not an engineering one. Either comments survive with
the author anonymized to a tombstone identity, which is what I would do because
removing them damages other users' threads, or comments are removed, which
damages those users. I would build the anonymous tombstone because it is
reversible if the legal answer changes, whereas deletion is not.

Interviewer: Anything thin in your own design that you would send a junior to
study separately before they touch it?

Candidate: Four things. Study separately: [[hotspot-handling|Hot Key
Handling]], because the creator problem is the single hardest part of this
design and it is a small set of ideas applied well. Study separately:
[[fanout-and-aggregation|Fanout on Read versus Fanout on Write]], including the
inverted-index formulation, because my hybrid is a special case of it and
knowing the general shape will let me reason about the threshold rather than
just copying my number. Study separately: [[idempotent-consumer|Idempotent
Consumers]] and [[delivery-semantics|Delivery Semantics]], because every
correctness bug in an asynchronous design is an idempotency bug. And study
separately: [[tail-latency|Tail Latency]], because my availability target is
essentially a tail-latency target and average latency is a lie I would be
reporting to you if I only gave you averages.

## Act 10 - Final Summary

Interviewer: Give me the design in ninety seconds, as if I am the engineer who
has to build it on Monday.

Candidate: Clients talk to a global edge, which terminates TLS, authenticates,
rate limits, and serves media from the CDN.

Candidate: Media bytes go client to object storage directly through presigned
multipart URLs. Application servers never see them. On complete, we write the
post row and an outbox row in one transaction, and a relay publishes
MediaUploaded to Kafka.

Candidate: Media workers consume that topic and do EXIF strip, validation,
thumbnails, blurred placeholders, the transcode ladder for video, and
moderation. On success they emit PostPublished, which flips visibility, warms
the author's grid cache, and starts feed fanout.

Candidate: Metadata lives in sharded relational stores: posts by post_id with
a separate user-time index, likes and comments by post_id with a thirty-two-way
sub-shard for hot posts, follows duplicated into follower-keyed and
followee-keyed tables. Each shard is a primary with two replicas, virtual
shards from day one, and quorum-based failover with fencing.

Candidate: Feed is hybrid. Under ten thousand followers, fanout on write into a
per-user materialized feed in a dedicated Redis cluster. Over ten thousand, no
materialization; one extra cached sorted-set read per feed request, and for the
top ten thousand creators, a precomputed chunk refreshed every thirty seconds.
Cursor pagination, single hydrated batch read, never an N plus one.

Candidate: Likes are a synchronous Redis sub-counter plus a synchronous write
to a per-user liked set, with durable persistence delegated to Kafka and the
relational store as a projection. The count is eventually correct within about a
second, and the user's own like reads back instantly.

Candidate: Everything derived, meaning feed entries, counters, hashtag
timelines, trending, and notifications, is built by idempotent Kafka consumers,
with pre-aggregation and grouping at the read layer for anything with a huge
fanout.

Candidate: Availability comes from multi-region reads, a single writer home
region, circuit breakers with fallbacks on every cross-service call, jittered
TTLs with stale-while-revalidate and single-flight on the cache, and graceful
degradation that would rather serve a 720p photo than an error.

Candidate: The three risks the product team named both have named answers. The
upload pipeline is regional and asynchronous with retry queues and a
reconciliation job, so a broken region costs availability, not posts. The
hot creator is the hybrid threshold plus precomputed chunks, so five hundred
million followers is one cache read rather than five hundred million writes.

Interviewer: That is a good ninety seconds. Thank you.
