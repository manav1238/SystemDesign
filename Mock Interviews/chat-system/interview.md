---
title: "Chat System (WhatsApp) — Interview Transcript"
status: active
tags: [hld, mock, chat-system, transcript]
---

# Chat System (WhatsApp) — Interview Transcript

Target duration: 45 minutes. Read this as a demonstration of *how* a strong candidate reasons, not
just what architecture they land on. Every number spoken aloud is shown with its arithmetic.

---

## Phase 1: Requirements Clarification

**Interviewer:** Thanks for joining. Today's problem is WhatsApp. Design a real-time messaging
system that supports 1:1 and group chat, online presence, message history, read receipts, and
typing indicators, at the scale of 500 million daily active users and roughly 60 billion messages
per day. Before you design anything, what do you need to clarify with me?

**Candidate:** Thanks. I have about eight questions, and they're ordered by how much they change
the architecture. Let me start with the strictest one.

**Interviewer:** Go ahead.

**Candidate:** First, is the one-second delivery a hard requirement, and is it a p99? Because
"real time" in messaging can mean 200 ms p99 for a user sitting on Wi-Fi in the same city, or
5 seconds for someone on a 2G connection in a rural area. Those two push the architecture in
different directions. If it is 200 ms p99, I have to keep a warm, always-on connection and I cannot
batch or coalesce sends. If it is 5 seconds, I can do request batching on the sender side and the
whole system gets about 30 percent cheaper.

**Interviewer:** Let's say 500 ms p99 for users who are online on a decent connection, and it is
allowed to degrade for users who are not.

**Candidate:** Good, that tells me the online path is a persistent connection, not polling. Second
question: is ordering required only within a conversation, or globally across all of a user's
conversations?

**Interviewer:** Only within a conversation. Cross-conversation ordering does not matter.

**Candidate:** That is a huge relief, because global ordering across all of a user's conversations
would force a single serialization point per user, which becomes a throughput bottleneck and a
tail-latency problem. Per-conversation ordering means I can shard by conversation id and get
ordering almost for free, because all messages for one conversation go to one shard.

**Interviewer:** Fine. Now, the group chat with 500,000 members. Does a new message in that
conversation get materialized as a row per member?

**Candidate:** This is the question I would push back on hardest. Let me ask it differently: how
many messages do you expect in a 500,000-member group per day, and does the product actually need
each member to have an independent unread count and an independent read receipt?

**Interviewer:** Assume a busy one gets a few thousand messages a day. Let's say every member needs
an accurate unread badge.

**Candidate:** Then I cannot do pure read-time fanout by default, because an accurate per-member
unread badge is inherently write-time state. But I can make it a two-tier design: write-time
fanout for normal conversations up to some membership threshold, and read-time fanout for the
pathological large groups, where the product compromise is a shared unread count per conversation
rather than per member. I will come back to that in the deep dive, because it is the single most
important design decision in the whole system.

**Interviewer:** Acceptable. Next question.

**Candidate:** Third: are read receipts per message, or do we only need "the highest message id this
user has read" per conversation, and the client renders ticks for everything below it?

**Interviewer:** Per-message granularity is nicer but I will take the cheaper one if you can show me
it looks the same to the user.

**Candidate:** It looks identical for the overwhelming majority of cases, and I can prove it is a
lossless compression. Read receipts are monotonic. If a user has read message 47, they have read 1
through 47, because you cannot read message 47 without having received 46. So the receipt stream is
a monotonically increasing sequence, and the only information content is the *latest* value. Storing
60 billion receipt rows a day to store a value that is a single integer is a 20x write amplification
for zero information. I will store one row per user per conversation, updated in place.

**Interviewer:** Good. What about the typing indicator?

**Candidate:** Typing must be ephemeral and lossy. I will never persist it, and I will never make
delivery of it reliable. The moment it stops being useful it should be dropped, and a typing
indicator that arrives four seconds late is worse than one that never arrives, because it actively
misleads the user. Same reasoning applies to presence, but presence is subtler, so I will come back
to it.

**Interviewer:** Now a hard one. End-to-end encryption. Does that change your design?

**Candidate:** Fundamentally yes, in two places. First, the server can no longer scan content, so
all abuse moderation, spam detection, and illegal-content handling must move either to the client or
to a separate client-side-report mechanism. That is a product and trust-and-safety problem, not a
storage problem. Second, it constrains server-side features: server-side search becomes impossible
unless you do search over the client's local index, and link preview generation has to happen on
the sender's device. Storage-wise the impact is small, because the server still needs to route,
order, and store ciphertext plus routing metadata.

**Interviewer:** And retention?

**Candidate:** Messages are near-immutable once written, so this is an append-only workload, which
is the friendliest possible storage pattern. I will keep years of history, and I will do soft
delete with a tombstone rather than a hard delete, because a hard delete across 3 replicas plus a
cache plus a search index is a multi-system consistency problem that is not worth solving for a
feature that runs once per user per message.

**Interviewer:** Good. I think I have enough. One more: what happens if a user is offline?

**Candidate:** Nothing breaks, and this is where the design earns its keep. Offline is the common
case, not the exception, at this scale. The message is durably stored, the recipient's per-device
message pointer is advanced, and delivery happens later at login or on reconnect by reading from
the message store using that pointer. I do not keep messages in a queue for offline users. Holding
a message in a queue for 30 days is just a worse database.

---

## Phase 2: Scale Estimation

**Interviewer:** Now do the math. Talk me through your numbers out loud.

**Candidate:** Starting from the top. 500 million daily active users. 60 billion messages per day.

**Interviewer:** Go.

**Candidate:** 60 billion divided by 86,400 seconds is about 694,000 messages per second on
average. So roughly 700,000 sends per second, and if I assume a peak-to-average ratio of 2, which
is normal for a consumer product synchronized to waking hours, peak is around 1.4 million sends per
second.

**Interviewer:** Is that the number I should worry about?

**Candidate:** No, and this is the key insight of the estimation phase. 700,000 is the number of
*sends*, but the real load on the system is *deliveries*, because a group message is delivered to
every member. If average group size including 1:1 is about 3, then delivery events per second is
700,000 times 3, which is about 2.1 million per second on average and roughly 4.2 million at peak.

**Interviewer:** That is a big difference.

**Candidate:** It is a 6x difference, and it is the difference between "I need a big database" and
"I need a carefully designed fanout layer." If I materialize a delivery record for every member of
every conversation, then at 2.1 million deliveries per second I am writing 2.1 million rows per
second just for delivery bookkeeping, plus 700,000 for the messages themselves, plus receipts.

**Interviewer:** Where do receipts land?

**Candidate:** Here is where my monotonic-compression argument pays off. If receipts were per
message, that is another 2.1 million writes per second. Because I compress them to one integer per
user per conversation, receipt writes scale with *active conversations*, not with *members*, so it
is maybe 50,000 to 100,000 writes per second. That one decision removes about 2 million writes per
second from the system. That is the whole point of doing estimation before designing.

**Interviewer:** Storage next.

**Candidate:** Per message, let me budget a text payload of 100 bytes average, and then metadata:
sender id, conversation id, message id, timestamp, message type, and the per-conversation sequence
number. If I use 64-bit ids, 8 bytes each, that is about 50 bytes, plus indexes and the per-user
message pointer. Call it 250 bytes per message all-in.

**Interviewer:** So annual volume.

**Candidate:** 60 billion times 365 is 21.9 trillion messages per year, which I will round to 22
trillion. Times 250 bytes is 5.5 petabytes per year, call it 5 to 6 PB. Per day it is about
1.4 TB. That number matters for a specific reason: at 1.4 TB per day of pure append, I must be
careful about how many times I copy that data. Each replica copy is another 5 PB per year of disk
and another 5 PB of network traffic, so replication factor is a cost multiplier, not a rounding
error.

**Interviewer:** And the conversation metadata, the membership lists?

**Candidate:** Small, but not trivial. 500 million users, average maybe 30 conversations each, so
about 15 billion conversation-membership rows. At 50 bytes each that is 750 GB, and I index it by
both directions, so maybe 1.5 TB. That is comfortable. The asymmetry is important: the membership
graph is a normal-sized relational workload, while the message stream is an append-only
petabyte-scale workload. Treating them as one database is a mistake.

**Interviewer:** Read to write ratio?

**Candidate:** Very heavily read-skewed. Messaging is not just receiving, it is scrolling back
through history, searching, and switching devices and catching up. I would estimate reads to writes
at 20:1 or higher, and the hottest reads are the last 50 messages in the conversations you actively
open. That skew is exactly what caching is for.

**Interviewer:** Concurrent connections?

**Candidate:** At 500 million DAU, if average session length is 20 minutes, that is 500 million
times 3 hours a day is 1.5 billion user-hours per day, divided by 24 gives about 62.5 million
concurrent online users at any moment. 62.5 million open WebSocket connections. I need to be honest
with myself that this is the real sizing constraint on the gateway tier, not the database. One
process holding 100,000 connections is about 6.25 billion... let me redo that: 62.5 million divided
by 100,000 is 625 gateway processes. And with 3x for failure domains, roughly 2,000 machines
dedicated purely to holding sockets open.

---

## Phase 3: High-Level Architecture

**Interviewer:** Now design it. Walk me through the components and draw it.

**Candidate:** Let me describe it in tiers, then draw it.

**Interviewer:** Go ahead.

**Candidate:** Tier one is the connection tier. Clients on mobile and web hold a persistent
[[websockets|WebSocket]] connection to a gateway node. The gateway does four things and nothing
else: terminate TLS, authenticate the session, keep the socket in a map, and translate between
socket events and internal RPC. It holds no business logic and no durable state, which means it can
be killed at any moment. That is a deliberate design choice, because the cheapest way to survive a
gateway failure is for the gateway to be allowed to die.

**Interviewer:** So a gateway crash does what?

**Candidate:** Clients detect the closed socket and reconnect to a different gateway, which is a
TCP reconnect plus a re-auth, typically 200 to 400 ms. The session token lives in [[redis]] and
in the auth service, not in the gateway, so reconnect is stateless. Meanwhile, if the client was
mid-send, the idempotency key from the send API lets it retry without creating a duplicate.

**Interviewer:** Tier two?

**Candidate:** Tier two is the stateless service layer: a Chat Service that owns message writes, a
Presence Service, a Receipt Service, and a Sync Service for history. All of these are stateless and
horizontally scaled behind a [[load-balancing|load balancer]]. Tier three is the fanout and
delivery tier, which is the part that needs the most care because of the 6x fanout amplification I
calculated earlier. Tier four is storage: a sharded, replicated message store, [[redis]] for hot
state, and an object store for media.

**Candidate:** Here is the picture.

**Candidate:**
```text
                        +---------------------------+
                        |        Clients           |
                        |  mobile / web / desktop   |
                        +-------------+-------------+
                                      |
                            TLS + WebSocket (persistent)
                                      |
                        +-------------v-------------+
                        |  Gateway Tier  (stateless) |
                        |  auth, socket map, fan-in  |
                        |  ~62M concurrent conns     |
                        +------+-------------+-------+
                               |             |
              +----------------+             +----------------+
              |                                       |
   +----------v-----------+                   +-----------v----------+
   |  Chat Service        |                   |  Presence Service    |
   |  (stateless, write)  |                   |  (stateless, RPC)    |
   +----------+-----------+                   +-----------+----------+
              |                                       |
              |  MessageStore                        |  Redis presence
              |  shard = conversation_id              |  TTL keys
              |  append-only, 3 replicas              |
              |                                       |
              +----------------+----------------------+
                               |
                 +-------------v--------------+
                 |   Fanout / Delivery Svc    |
                 |   decides: online push?    |
                 |   or store-for-later?      |
                 +------+-------------+-------+
                        |             |
          +-------------v-+  +--------v-------------+
          |  Message Queue|  |  Gateway nodes that |
          |  (Kafka,      |  |  hold the recipient |
          |   per-conv    |  |  sockets            |
          |   ordering)   |  +---------------------+
          +---------------+
```

**Candidate:** The critical piece is that arrow from Chat Service to Fanout Service. Sending a
message and delivering a message are different concerns with completely different scaling
profiles, and coupling them is how you get an outage. The send path must return fast and
durably-acknowledge. The delivery path is allowed to be slow, to retry, to batch, and to drop
without user-visible harm.

**Interviewer:** Why a queue in the middle at all?

**Candidate:** Three reasons, and they are the standard reasons, but I want to apply them
specifically. First, decoupling: 700,000 sends per second should not have to wait for 4.2 million
deliveries. Second, burst absorption: when 500 million users wake up in the same time zone, I get
a traffic spike and the queue absorbs it while the delivery tier scales up. Third, and this is the
one people forget, failure isolation: if a gateway region goes down, in-flight messages are already
durably in the queue and the delivery service retries them when it comes back. The queue is
[[delivery-semantics|delivery semantics]] insurance.

---

## Phase 4: Deep Dive

### Message ordering

**Interviewer:** Ordering. How do you guarantee it?

**Candidate:** Two things. First, I do not use wall-clock time for ordering. Client clocks are
untrusted, and server clocks across machines drift. Instead, every conversation has a monotonic
sequence number assigned by the [[sharding|shard]] that owns the conversation, and ordering is by
that sequence number, not by timestamp. The timestamp is still stored, but only for display.

**Interviewer:** How do you assign the sequence number without a bottleneck?

**Candidate:** Because all messages in a conversation hash to the same shard, and that shard has a
single writer per partition, the sequence number is just a counter increment on the shard. There is
no cross-shard coordination, and no distributed lock. This is the payoff of per-conversation
ordering: the ordering guarantee falls out of the partitioning.

**Candidate:**
```text
conversation: hash("conv_9f2a") % 4096 = shard 1183

  seq 1041  | client A | 09:14:02.113
  seq 1042  | client B | 09:14:02.104   <- earlier wall clock, later seq
  seq 1043  | client A | 09:14:02.290
```

**Interviewer:** That example is interesting. The wall clocks say 1042 happened first, but 1042
was assigned a later sequence. What went wrong?

**Candidate:** Client B's message sat in a network buffer or in a retry, so it arrived later even
though the user pressed send earlier. If I ordered by timestamp, 1041 and 1042 would be displayed
in the wrong order relative to human intent, and worse, two different servers with skewed clocks
would produce genuinely non-reproducible orderings. Sequence numbers make the order a total,
consistent, server-decided fact.

**Interviewer:** But sequence numbers are assigned by the shard. What if the same user is on two
devices and sends from both?

**Candidate:** Then both sends are serialized by the same conversation shard and get consecutive
sequence numbers, but the order reflects arrival at the server, not the user's intent. This is
known as the interleaving problem and it is genuinely unsolvable without a central ordering
authority. I will document it as a semantic guarantee: per-conversation, total order by server
assignment. Most messaging products behave exactly this way.

**Interviewer:** What about the queue, does ordering survive there?

**Candidate:** This is where it gets subtle. Ordering is only preserved if a conversation's
messages all land in the same partition and are consumed by the same consumer instance, in order.
So I key the [[message-queue|queue]] by conversation id, and I note the cost honestly: with a fixed
partition count, a small number of very hot conversations can make some partitions much busier than
others. That is the classic skew problem, and I handle it with a larger partition count plus
consumer-side hot-partition splitting for the worst offenders.

Study separately: [[kafka-ordering|Kafka Ordering]], and the Kafka partition-to-consumer
assignment mechanism in [[kafka-producers-consumers|Kafka Producers and Consumers]].

**Interviewer:** Now the send API. Design it.

**Candidate:**
```text
POST /v1/messages
Headers:
  Idempotency-Key: <uuid>
  Authorization: Bearer <token>
Body:
  conversation_id: string
  type: "text" | "image" | "file" | "location"
  text: string
  media_ref: string
  reply_to_message_id: string
  client_sent_at_ms: long

Response 201:
  message_id: string
  conversation_id: string
  sequence: long
  server_timestamp: long

Response 409:
  error: "idempotency_key_reuse"
```

**Candidate:** Reading the contract: the idempotency key is generated on the device and is stable
across retries, which is what makes the retry safe. The `media_ref` is a pointer into object
storage rather than the bytes themselves, so a 3 MB photo never traverses the chat service. The
`client_sent_at_ms` is display-only and never used for ordering, for the clock-skew reason I gave
earlier. And the 201 returns the assigned `sequence`, so the sender's own UI can immediately sort
and place the message optimistically without waiting for a server read-back.

**Interviewer:** Why is 409 a thing?

**Candidate:** Because [[idempotency|Idempotency]] keys have a subtle failure mode. If a client
retries with the same key, I return the original result. But if a buggy client reuses one key for
two genuinely different messages, blindly returning the first result means the second message
silently disappears. So I store a hash of the request body with the idempotency record, and if the
hash differs I reject. This is a real bug class I have seen in practice, not a theoretical one.

**Interviewer:** Show me the message store schema.

**Candidate:**
```sql
CREATE TABLE messages (
  conversation_id   VARCHAR(64)  NOT NULL,
  sequence          BIGINT       NOT NULL,   -- monotonic per conversation
  message_id        VARCHAR(64)  NOT NULL,
  sender_id         VARCHAR(64)  NOT NULL,
  type              VARCHAR(16)  NOT NULL,
  text              TEXT,
  media_ref         VARCHAR(256),
  client_sent_at_ms BIGINT,
  server_ts         BIGINT       NOT NULL,
  edited_at         BIGINT,
  deleted_at        BIGINT,                  -- soft delete / tombstone
  PRIMARY KEY (conversation_id, sequence)
) PARTITION BY RANGE (server_ts);

CREATE INDEX idx_messages_sender ON messages (sender_id, server_ts);
```

**Candidate:** Two things to notice. The primary key is the conversation and sequence, not a
random message id, because that makes the write append-only at the right edge and makes history
pagination a pure range scan with no sorting. And I partition by time, because chat data has
enormous skew, almost all reads hit the last few days, and almost all deletes are old. Time
partitioning lets me drop whole partitions for retention instead of issuing mass deletes.

**Interviewer:** How do you page through history?

**Candidate:** Cursor-based, never offset-based. Offset means the database counts rows it skips,
which is O(offset) and gets slower the deeper you go. A conversation with 500,000 messages and a
user scrolling to message 400,000 would be brutal. Instead:

**Candidate:**
```text
GET /v1/conversations/{conversation_id}/messages?before_seq=1043&limit=50

Response:
  messages: [ {sequence, message_id, sender_id, text, server_ts, ...}, ... ]
  next_before_seq: 993
  has_more: true
```

**Candidate:** The cursor is the sequence number, which is a natural stable key. If new messages
arrive while the user scrolls, the cursor pagination is unaffected, because I am slicing a
half-open range on an immutable ordered key. This is the same reason you never use
`OFFSET 10000` in a feed. See [[pagination]].

**Interviewer:** What about the newest messages, and pulling them to the top of the phone?

**Candidate:** Forward pagination with a `after_seq` cursor, but with a hard cap, usually 50. A
client should never ask for 10,000 messages in one request, because that is a denial-of-service
vector against my own database and it will blow the phone's memory. If a user is switching devices
and genuinely wants a full export, that is a separate asynchronous export job that writes a file to
object storage and emails a link. Never a synchronous API.

**Interviewer:** Now the interesting one. How do you fan out, and how do you do it at read time
versus write time?

**Candidate:** This is the core trade-off, so let me lay out both.

**Candidate:**
```text
WRITE-TIME FANOUT (used when member_count <= 256)
  on send:
    for each member:
       upsert per_user_conversation(
         user_id, conversation_id,
         last_delivered_seq, unread_count, muted, pinned, cleared_seq
       )

  read / sync:
    SELECT messages WHERE sequence > last_delivered_seq
    for THIS user only  -- 1 shard read, no join

READ-TIME FANOUT (used when member_count > 256)
  on send:
    store message once in the conversation shard
    store only conversation-level unread_count
    per-user bookkeeping happens lazily

  read:
    list conversations for user  ->  N shard reads, parallel
    for each: SELECT messages WHERE sequence > cleared_seq LIMIT 50
```

**Interviewer:** Why would you ever choose write-time fanout? It is more writes.

**Candidate:** Three reasons, and they are all about the read path. First, latency: the sync query
becomes a single-key lookup on one shard instead of a scatter-gather across shards. Second,
unread counts: an accurate per-member unread badge is mutable state that must be updated on read
anyway if you do read-time fanout, which means the read path takes locks and it becomes a
write-during-read. Third, the N+1 problem: at login I have to sync every conversation, and with
500 unread conversations that is 500 parallel shard reads, which overwhelms the connection pool and
spikes tail latency.

**Candidate:** So my answer is: write-time fanout by default because it moves work off the read
path, and read-time fanout as an escape valve for the pathological groups where the write cost is
astronomically worse than the read cost. A 500,000-member group with 3,000 messages a day is 1.5
billion per-user rows a day. That is 17,000 writes per second just for that one conversation, and
it would flatten the whole shard. Read-time fanout makes it 3,000 writes.

**Interviewer:** How do you decide the threshold?

**Candidate:** By measured write amplification on the shard, not by a hand-picked number. I would
start at 256, instrument fanout cost per conversation, and move the boundary based on which
conversations actually hurt. The principle is: choose write-time when fanout factor is small enough
that the read-path saving dominates; choose read-time when the fanout factor makes writes the
bottleneck.

**Interviewer:** Now presence. How do you track who is online at this scale?

**Candidate:** Presence is the most interesting problem in the system because of its write
frequency versus its value decay. A user toggles online and offline constantly, but the answer
goes stale in seconds. So it is almost pure ephemeral state, and it must never touch the durable
store.

**Candidate:**
```text
gateway node, on connect:    SET presence:{user_id} "online"  EX 40
gateway node, on disconnect: DEL presence:{user_id}
presence read:               EXISTS presence:{user_id}   (1 hop, sub-ms)

concurrent sessions per user: use a per-device set
  SADD presence:sessions:{user_id} {gateway_id, device_id}
  EXPIRE presence:sessions:{user_id} 40
  online := SCARD presence:sessions:{user_id} > 0
```

**Interviewer:** Why a 40 second TTL and not an explicit offline write?

**Candidate:** Because explicit offline writes are wrong under failure. If a phone loses signal,
the client never sends a disconnect, so without a TTL the user is stuck "online" forever, which
means their contacts keep trying to push to a dead connection and the last-seen timestamp is a lie.
The TTL is the source of truth and the heartbeat is just an optimization. This is the
[[heartbeat-health-checks|heartbeat]] pattern applied to user state, and it is the same reason
[[gossip-protocol|gossip]] exists in service discovery.

**Interviewer:** But presence reads now hit Redis on every conversation list load. At 62 million
online users with 30 conversations each, is that not a lot of Redis traffic?

**Candidate:** Yes, and the naive version is bad. Loading a 30-conversation list and doing 30
sequential Redis calls is unacceptable. Two fixes. First, a read repair: when the Presence Service
knows a user's status, it returns the presence of everyone in that batch as a side effect, so
subsequent conversation loads hit cache. Second, and more effective, presence is only
approximate and therefore allowed to be a few seconds stale, so I serve it from a local in-process
cache with a 2 to 5 second TTL, refreshed in the background.

**Candidate:**
```text
GET /v1/users/presence?ids=u1,u2,...,u100

  cache: local map, 3s TTL
  misses -> single MGET to redis (one round trip, not 100)
  on local TTL expiry -> async background refresh
```

**Candidate:** Note the batch cap of 100 ids. Presence has to be batched because the real call
site is a conversation list or a group member list, and those are naturally 10 to 100 ids. A single
per-user presence endpoint would be technically correct and operationally useless.

**Interviewer:** How do subscribers learn that presence changed?

**Candidate:** Publish presence-change events to the same message bus, scoped per user, and each
online gateway subscribes only for its own connected users. A gateway holding 100,000 sockets
subscribes to 100,000 user topics, which sounds expensive, but since it already holds 100,000
sockets, the marginal cost of 100,000 subscription entries is small and the alternative, polling,
is 100,000 times worse. This is fanout-on-write for presence, deliberately, because presence
changes are low-volume compared to messages, so paying at write time to keep the read path free is
correct. See [[fanout-and-aggregation]].

**Interviewer:** You said do not persist presence. But last seen?

**Candidate:** Last seen is different, and this distinction is worth stating precisely. Presence is
"are you connected right now", which decays in seconds and is worthless later. Last-seen is "when
did you last connect", which does not decay and is displayed for weeks. So last-seen is
durable-ish, but still not in the transactional message store, because writing a row to a petabyte
store on every connect is absurd. It goes in [[redis]] with a long TTL, say 30 days, and is
periodically flushed to a cheap key-value store. Best-effort durability is fine for last-seen, and
being honest that it is best-effort is better than pretending it is transactional.

**Interviewer:** Now typing indicators.

**Candidate:** Typing is the purest example of ephemeral broadcast in the system. It is
write-only, never read from storage, never acknowledged, and dropped without retry. The flow:
client sends a typing-started frame, the gateway forwards it to the conversation shard, the shard
publishes a typing event keyed by conversation id, and the delivery tier pushes to online members
of that conversation only.

**Candidate:**
```text
client -> gateway: {type: "typing", conversation_id, state: "start"}
                    (throttled client-side to 1 per 3s)

server: PUBLISH typing:{conversation_id} {user_id, state, ts}
        no persistence, no ack, no retry, TTL ~5s on receiver

receiver: show indicator, start local 6s timer,
          hide on timer expiry OR on "stop" OR on incoming message
```

**Interviewer:** Why no retry? If the packet is lost the indicator just never appears. Isn't that
a correctness problem?

**Candidate:** It is a deliberate trade, and the argument is that the indicator is not a fact about
the world, it is a hint about an instant that has already passed by the time it arrives. Retrying
an ephemeral signal is not just useless, it is harmful, because a successful retry delivers a
statement that is now false. Retries make sense when the signal is a durable fact whose value does
not decay, like a message. It does not make sense for a decaying signal. This is a general
principle I keep in mind: *the reliability guarantee you choose must match the half-life of the
signal's usefulness.*

**Interviewer:** Read receipts and delivered. Walk me through the distinction.

**Candidate:** They are two different facts with different latency requirements. Delivered means
the message reached the recipient's device. Read means a human looked at it.

**Candidate:**
```text
DELIVERED:
  on gateway receive -> set last_delivered_seq = {seq} for that device
  (1 MGET-pipelined redis write per conversation, batched every 2s)
  sender sees: delivered when their seq <= min(last_delivered_seq of all devices)

READ:
  client renders the conversation -> PUT /v1/conversations/{id}/read
      body: { up_to_seq: 1043, device_id: "..." }
  store: UPSERT conversation_state
      (user_id, conversation_id) -> (last_read_seq, last_delivered_seq, updated_at)
  sender sees: read when their seq <= last_read_seq
```

**Interviewer:** Why is delivered stored per device but read stored per user?

**Candidate:** Because delivered is genuinely per-device. It is a statement about one phone, and it
drives the multi-device UX, where the sender sees "delivered to this device at this time". Read is a
statement about the human, not the handset. Once you have read a message, you have read it, on
every device. Collapsing read to the user level also collapses the write rate, which as I
calculated is the whole point of the monotonic-compression argument from Phase 1.

**Interviewer:** And in a group chat, who gets a read receipt?

**Candidate:** Nobody, by default, because it becomes a social problem, not a technical one. In a
500,000-member group, who cares that 300,000 people read it? WhatsApp shows read receipts only for
messages you personally sent in a 1:1 chat, and group read counts are aggregate and optional. I
would default group receipts to off and make it a user setting, because the storage and social cost
is real and the product value is low.

**Interviewer:** Now backpressure. What if a client is on a slow connection or offline?

**Candidate:** Three layered defenses, because a slow consumer is a resource leak if you let it be.

**Candidate:**
```text
1. SOCKET BUFFER LIMIT
   gateway write buffer cap: 256KB per connection
   if full -> do NOT block the event loop

2. BACKPRESSURE SIGNAL
   overloaded client -> persist-and-stop
     - do not read more from the socket
     - pause the TCP window (kernel handles it)
     - mark connection "slow"
   after 30s paused -> mark connection "lagging"

3. ESCALATION
   lagging -> stop pushing ephemeral frames (typing, presence, read updates)
             keep pushing durable messages (they queue in the store anyway)
   beyond 60s -> close socket, client reconnects and does
                 a normal sync from last_delivered_seq
```

**Interviewer:** Why is the last step closing the socket? Isn't that punishing the user?

**Candidate:** It is protecting the server, and it is the kindest thing for the other 62 million
users. A socket that is 60 seconds behind is holding a file descriptor, a memory buffer, and a
gateway slot, and it will never catch up, so it is strictly consuming resources with no chance of
becoming healthy. The user reconnects, and because sync is a clean range query on an immutable key,
the recovery is a fast, well-defined operation rather than a resumption of a corrupt stream. This
is [[load-shedding]] and [[backpressure]], and the general rule is: degrade the slow client before
you degrade the fleet.

**Interviewer:** You mentioned idempotency earlier. How do you dedupe?

**Candidate:** Three layers, each catching a different failure.

**Candidate:**
```text
Layer 1 - CLIENT (fastest, no network)
  message_id generated on device, stable across retries
  same message_id within a conversation -> never appended twice

Layer 2 - GATEWAY / IDEMPOTENCY KEY
  Idempotency-Key header -> redis SETNX with 24h TTL
  returns cached response on retry

Layer 3 - STORE (the real guarantee)
  PRIMARY KEY (conversation_id, sequence)
  plus a secondary unique index on message_id
  duplicate insert -> constraint violation -> treat as success,
  return the existing row
```

**Interviewer:** Which layer actually matters?

**Candidate:** Layer 3, and I want to be honest about that. Layers 1 and 2 are optimizations that
avoid wasted work and give faster retries. They can both be defeated: a client bug, a redis flush,
a lost idempotency record. Only the store's uniqueness constraint is a real guarantee, because it
is the same transactional boundary as the write itself. This is why I always push back on
application-level dedupe as a correctness mechanism. See [[idempotent-consumer]] and
[[request-deduplication]].

**Interviewer:** Let me push you. Traffic just doubled overnight, from 700,000 sends per second
to 1.4 million. What breaks first, and what do you do?

**Candidate:** Let me trace the pressure rather than guess. The client entry point absorbs a lot,
because gateways are stateless, horizontally scalable, and the cost is per-connection not per-user,
so gateways scale out and a [[load-balancing|load balancer]] or L4 consistent hash spreads
connections. Stage two, the Chat Service, also scales horizontally, but it now competes for
message-store connections, and I start seeing connection pool contention. Stage three, the shard
set, is the hard limit, because shards are physical partitions of a keyspace and I cannot create
them without resharding.

**Candidate:** So the ordered response is: first, verify the read path is not the problem, because
doubling sends does not double reads, and the read path is 20x the write path, so I check whether
my cache hit rate degraded. Second, vertical scale the hottest shards, which is often the cheapest
real fix and I do it before buying machines. Third, if that is not enough, add shards via
[[shard-rebalancing|resharding]], accepting the operational cost. Fourth, degrade gracefully: cap
the number of large-group fanout operations, and shed background work like presence broadcasts
and last-seen flushes, because nobody notices when last-seen is 10 minutes stale but everybody
notices when messages are late.

**Interviewer:** The primary database for the message store dies. Walk me through the recovery.

**Candidate:** First, what kind of store is this? A relational store, or a wide-column or
document store? I will assume a wide-column store sharded by conversation id, because the access
pattern is key-value and append-only, which is not what a relational database is optimized for.
See [[sql-vs-nosql]]. With that assumption, each shard is a leader with 2 followers using
[[database-replication|synchronous replication to one follower and asynchronous to the other]].
When the leader dies, the synchronous follower has all acknowledged writes and is promoted, and the
asynchronous follower catches up. If the failure is a node crash, this is a 5 to 10 second
promotion plus client reconnect, and no data is lost beyond the async replica's window.

**Interviewer:** And if it is not a crash, but corruption or a bad deploy?

**Candidate:** Then the asynchronous follower is the one we promote, and we have lost its window of
writes, which is where [[idempotency]] becomes load-bearing. Because the client retries with a
stable message id, a retried send that was actually committed re-inserts and hits the uniqueness
constraint, so the message is recovered rather than duplicated. That is the property that makes
asynchronous replication tolerable for messaging: at-least-once transport plus idempotent
application equals an effectively-once outcome. The write I care about most, the
last-delivered-seq pointer, is also idempotent because it is a monotonic max, so a replayed
delivery receipt is harmless.

**Interviewer:** What about replica lag? You mentioned it, let's make it concrete.

**Candidate:** Concrete scenario: device A reads up to sequence 500, then goes offline for 40
seconds. Meanwhile 200 more messages arrive. Device A reconnects and says `after_seq=500`, gets
200 messages, all good. Now the bad case: device A says `after_seq=500` but its last-delivered-seq
pointer was updated optimistically to 900 while it was actually offline, so the query returns
nothing and the user believes their messages are gone.

**Candidate:** The fix is that the sync query must never trust the server's pointer. The server
pointer exists for cross-device coordination, and it is allowed to run ahead optimistically. For a
device's own catch-up, the device asks from the sequence it itself last rendered, because only the
client knows what actually made it onto the screen.

**Candidate:**
```text
sync(client_reported_seq):
  msgs = shard.range(conversation_id,
                     from = client_reported_seq,
                     limit = 200)
  return msgs
```

**Candidate:** The rule is simply: the client is always the source of truth for what it has
received. The server pointer is used for cross-device sync, where device B needs to know how far
device A got, but for a device's own catch-up, it asks from its own last sequence. That removes the
lag dependency entirely from the single-device path. The cross-device path is the one that tolerates
lag, and it tolerates it because it is showing a progress bar, not delivering a message.

**Interviewer:** What is replica lag actually made of, in your system?

**Candidate:** Almost entirely the read-receipt and delivered writes, because those are high-rate
small updates and they are the writes most likely to be batched. The message inserts are
sequential appends and replicate predictably. So if I see a shard's replication lag alarm, the first
thing I check is whether the receipt write path started falling behind, because that is the
highest-rate, lowest-value write in the system. That is a case where the right response is to drop
or coarsen receipts rather than to scale the cluster.

**Interviewer:** Single hot key. A single conversation with 500,000 members, or a celebrity
broadcasting. Where does it break?

**Candidate:** It breaks in three distinct places, and I want to name all three because people
usually only see the first. One, the conversation shard: 3,000 messages per second to one partition
while everything else averages 20. Two, the fanout: if I chose write-time fanout, 1.5 billion
per-user writes per day from that one conversation. Three, and this is the sneaky one, the
[[redis]] per-user state: every one of 500,000 members is reading and writing their
unread-count entry for that conversation, so the key space for one conversation is 500,000 hot
entries, and even though they are different keys they may land on the same shard or the same
physical host.

**Candidate:** The mitigations. For the message shard, read-time fanout as I described. For the
fanout, an explicit large-group path: store once, and materialize per-user state lazily with a
batch job, or accept aggregate unread. For the redis skew, I shard the per-user state by user id
rather than by conversation id, which I should have done from the start, and I add a per-conversation
read-through cache so that a shared conversation is served from a shared cached projection.

**Candidate:** I also want to mention the classic mitigation I would reach for if asked:
[[hotspot-handling|hot key handling]] by adding a random jitter to shard selection, or by
replicating the hot key. Shard jitter is wrong here, because it destroys the ordering guarantee
I built everything on. That is an important lesson: you cannot apply the generic hot-key
distribution trick when your key also carries an ordering constraint. The correct answer is to
change the *workload shape*, not to scatter the key.

**Interviewer:** Cache stampede. You have a hot conversation, everyone opens it at once, and the
cache is cold. What happens?

**Candidate:** Classic stampede, and in a messaging system it is aggravated by the fact that the
miss path is a database read on a petabyte-scale store, which is slow, so requests pile up while
waiting. I use four defenses together. One, request coalescing: a single flight per key, so 10,000
concurrent misses for the same key produce one database read and 9,999 waiters.

**Candidate:**
```text
key: chat:msgs:{conversation_id}:{bucket}
value: JSON of 50 messages
TTL: 5 minutes
negative caching: unknown conversation -> cache "empty" for 30s
```

**Candidate:** Two, a short jittered TTL so that a mass expiry does not become a synchronized
thundering herd. If I give every key exactly 300 seconds and 500,000 conversations were populated
at the same instant, they all expire at the same instant. So TTL is 300 seconds plus random 0 to
60 seconds. Three, serve stale on database failure: I keep a stale copy and serve it with a
degraded flag rather than returning an error, because a slightly stale message list is a much
better user experience than a spinner. Four, circuit-break the miss path: if the store's p99
degrades, I stop hitting the store and serve stale, because the miss path is precisely what
turns a slowdown into an outage.

**Interviewer:** Why is a stale message list acceptable here but a stale unread count also
acceptable? Why is anything stale acceptable?

**Candidate:** The right way to think about it is that every field in this system has a staleness
budget determined by what the user can detect and how much it costs them if they notice. Messages
you can scroll and see are strongly consistent, budget near zero, never serve stale. The last 50
messages are weakly consistent, budget 5 seconds, cache freely. Unread count, budget 30 seconds,
because a badge that is 10 seconds wrong is invisible. Typing indicator, budget 3 seconds, and
beyond that it is actively harmful. Presence, budget 5 seconds. Last seen, budget of minutes, since
it is a historical fact that nobody checks to the second. Once you assign staleness budgets, the
consistency model for each field stops being a judgement call and becomes a lookup. See
[[consistency-models]] and [[strong-vs-eventual-consistency]].

**Interviewer:** Why not use a relational database with partitioning? Why not SQL all the way?

**Candidate:** I would actually push back on that framing, because I would use relational
databases for a meaningful part of this system. The conversation and membership graph is
inherently relational, it has multi-hop queries like "all conversations shared with this person",
and it is small at 1.5 TB. That is a great fit for SQL, and I would use it. The message store is
the part that does not fit, because 22 trillion appends a year with no joins and no multi-row
transactions, is exactly the shape that wide-column stores and LSM trees handle well. So my answer
is not "no SQL", it is "use the right store per access pattern", and I will partition the
conversation graph by user id to keep membership lookups single-shard while accepting that
cross-user queries become scatter-gather. See [[data-model-types]] and
[[normalization-vs-denormalization]].

**Interviewer:** Why not gRPC between services, or HTTP everywhere?

**Candidate:** Internal, gRPC with protobuf, for three concrete reasons: it is binary so
serialization cost is lower on a hot path carrying millions of messages per second, it gives me
generated clients so the schema is enforced at compile time rather than at runtime, and it supports
streaming and deadlines natively, which I need for the presence subscription path. The client
facing edge is HTTP and WebSocket, because mobile clients and browsers are the constraint there and
they cannot do gRPC. The rule is: optimize the wire format on the internal hot path, use the
boring protocol at the edge. See [[rpc-grpc-graphql]] and [[api-gateway]].

**Interviewer:** How do you operate this? Containers, orchestration?

**Candidate:** The gateway and stateless services are perfect for [[containers-and-vms|containers]]
on [[kubernetes|Kubernetes]]. They are stateless, horizontally scaled, and their scale is driven by
connection count or CPU, which is what an HPA is good at. The message store is different, because a
sharded database shard is stateful and its capacity is disk and IOPS-bound, so it runs on dedicated
machines or on a managed offering, with the orchestrator managing only its deployment, not its
scaling. Putting a stateful shard under an HPA that reacts to CPU would be actively wrong, because
replication lag and disk pressure do not show up in CPU. See [[stateless-vs-stateful-services]]
and [[autoscaling]].

**Interviewer:** How do you know it is working? What are your dashboards?

**Interviewer:** Sorry, that was two questions. What are the four things you alert on?

**Candidate:** The four that actually matter, and I would resist adding more. One, send-to-deliver
latency, at p50, p99, and p99.9, split by online versus offline recipients, because that split tells
you whether the delay is in the queue or in the network. Two, per-shard replication lag, which
detects a sick follower before it becomes data loss. Three, gateway connection count and per-node
CPU, since that is the constraint I sized the tier on. Four, queue depth and consumer lag, which
tells me whether delivery is keeping up with send. Everything else is a dashboard, not a page. See
[[golden-signals]] and [[sli-slo-sla]].

**Interviewer:** Bunching up. What is the first thing that breaks in a regional outage?

**Candidate:** Depends on whether the region was serving reads or the write path. If a whole
region dies, the immediate issue is that in-flight deliveries in that region's queue are stuck, and
they are only recovered if the queue is replicated across regions. So the queue runs
multi-region with [[database-replication|replication factor 3 across regions]] or a
cross-region cluster, and the delivery consumers in the surviving region pick up the failed
region's partitions. The second issue is client reconnect, which is a thundering herd of 15
million connections, so I need reconnect jitter and a token bucket on the gateway, which is
[[load-shedding]] on the client edge. The third is a split-brain risk if clients can write to two
regions, and I prevent that with a single-writer-per-conversation lease, so a conversation in the
dead region fails to accept writes rather than accepting writes in two places. Failing to accept
writes is recoverable. Accepting writes in two places creates [[split-brain]], which is not.
See [[failover]] and [[regional-failover]].

**Interviewer:** Last deep-dive question. Cost. This design is enormous. Where would you cut if
you had to halve the bill?

**Candidate:** In this order. First, read-time fanout becomes the default and write-time becomes
opt-in, because I showed that fanout amplification is the dominant write cost. Second, drop
delivered receipts and keep only read receipts, since delivered is per-device and multiplies my
redis writes, and I can approximate it client-side. Third, shorten retention on media, since text
is 100 bytes and photos are 3 MB, so media is where 95 percent of the bytes are; I move media to
cheaper cold storage after 90 days. Fourth, tier presence: regional presence rather than global
presence, because a user in Frankfurt does not need to appear online to a contact in Sao Paulo in
real time. Fifth, and this is the one people avoid saying, reduce the last-seen and typing
features entirely, because they are pure cost and low engagement. Cost reduction should come from
removing features with low value per byte, not from making the hot path 10 percent faster. See
[[cost-estimation]] and [[storage-tiering]].

---

## Phase 5: Trade-offs and Follow-ups

**Interviewer:** Let me fire off a rapid set of "why not" questions. Start simple.

**Candidate:** Ask them in order of difficulty, that is kinder to both of us.

**Interviewer:** Why not long polling instead of WebSockets?

**Candidate:** Because I need bidirectional, low-latency, persistent connections with the server
pushing to the client. Long polling gives me server-to-client push but at the cost of a request
per message, with connection churn, and it burns a server thread or coroutine per idle
connection. [[websockets|WebSocket]] gives me a persistent duplex connection with a small frame
header and per-connection state, which at 62.5 million connections is the difference between a
system that fits on a few thousand machines and one that does not. The tradeoff I accept with
WebSocket is that I now own reconnection, heartbeats, and backpressure myself, and I would have
gotten some of that from an infrastructure layer. That is a real cost, not a free win.

**Interviewer:** Why not server-sent events?

**Candidate:** SSE is unidirectional, so it is perfect for presence, read receipts, and typing,
which are all server-to-client, and it is cheaper than a WebSocket because I do not need the
upgrade dance. But it is useless for sending, which is client-to-server. I could pair SSE with
plain HTTP POSTs for sending, and for a read-mostly feed that is a very clean design. For chat I
need duplex, so WebSocket. I will note that for a hypothetical "read-only notification channel",
SSE is the better answer.

**Interviewer:** Why not gRPC streaming for everything?

**Candidate:** Because mobile clients, especially iOS, cannot hold 60 million gRPC streams
efficiently, and browser WebSocket support is a translation layer I would have to pay for anyway.
gRPC is right for service-to-service and wrong for the client edge. Also, a long-lived gRPC stream
through many mobile proxies is more fragile than a WebSocket through the same path.

**Interviewer:** Why not Kafka for everything, including the send path?

**Candidate:** I do use [[kafka-architecture|Kafka]] for delivery, but the send path must not
depend on it. If message acceptance requires a Kafka write, then a Kafka partition rebalance
becomes a send-path outage, and a Kafka availability blip becomes total message loss. The send
path's contract to the user is "your message is safe", and the only place I can honestly make that
guarantee is the durable store that the user will later read from. Kafka is a delivery mechanism,
so it must sit downstream of durability, never upstream of it. Its role is absorbing fanout
amplification, which is a throughput problem, not a durability problem. See
[[loose-coupling]] and [[kafka-replication]].

**Interviewer:** Why not a message queue per conversation? Would that guarantee ordering for free?

**Candidate:** At 500,000 conversations in my example, that is 500,000 queues, which means
per-queue overhead, poor batching, and terrible broker utilization, because most conversations are
cold most of the time. This is exactly the case where [[sharding|Sharding]] of one logical queue
into a fixed number of partitions is the right answer instead. Fixed partitions give me
utilization, and conversation id as the key gives me ordering for free. Over-partitioning is the
failure mode to avoid, and it is a real cost in broker memory and file handles.

**Interviewer:** Why not use the same database for presence, receipts, and messages?

**Candidate:** Three reasons with three different failure domains. They have completely different
access patterns: the message store is append and range-scan by conversation, presence is point
reads with a 40 second lifetime, and receipts are point overwrites by user and conversation. They
have completely different lifetimes: messages live for years, presence for 40 seconds, receipts
longer but not forever. And they have completely different scaling shapes: the message store is
petabyte-scale and shard-bound, presence is 62 million ephemeral keys, receipts are high-rate
overwrites. One storage engine per access pattern, which is the same instinct as
[[sql-vs-nosql]] applied at the level of a single system rather than a single query.

**Interviewer:** Why not caching everything in Redis and treating the database as a backup?

**Candidate:** Because Redis is not durable in the sense I need, it is a shared-nothing in-memory
store, and the message store is my only copy of the truth. If Redis loses data, I recover because
the store is authoritative. If the store loses data, nothing recovers it. I will never invert that
dependency. What I will do is cache aggressively *into* Redis, with explicit TTLs and explicit
stale-serving rules, treating it purely as a read accelerator. See [[redis]] and
[[cache-size-estimation]].

**Interviewer:** Consistency or availability, in the CAP sense, where do you land?

**Candidate:** For message writes, consistency and durability win, because a message that
disappears is an unacceptable product failure, so I choose consistency. For presence, availability
wins absolutely, because a stale presence answer is harmless while an unavailable presence answer
breaks the whole experience, so I choose availability. For history reads, availability with
bounded staleness, because a 5 second old message list is fine. The interesting part is that I
refuse to apply one answer uniformly, and the [[cap-theorem]] is not a reason to give up on
reasoning per operation. See [[consistency]] and [[availability]].

**Interviewer:** Any final surprises a candidate usually misses?

**Candidate:** Four, and they are all things I would want to raise unprompted if I could. One,
the send path needs a synchronous, linearizable "did you actually accept this" answer, and
everything else can be asynchronous, so that distinction should be explicit in the API contract.
Two, a per-conversation sequence number is a schema decision with migration consequences, because
if I ever add a global ordering requirement I cannot retrofit it cheaply. Three, read receipts
create a privacy surface, since "when was I online and who saw my messages" is sensitive data, and
that is a compliance conversation, not just a storage conversation. Four, the cost of the naive
fanout is the dominant cost of the entire system, and it is invisible until the first 500,000
member group is created in production.

**Interviewer:** That's a lot of pressure. Go ahead, walk me through your final architecture in
two or three sentences, as if I am the one who has to remember it.

**Candidate:** Clients hold WebSocket connections to a stateless gateway tier that authenticates
and holds sockets but stores nothing durable; a stateless chat service accepts a send, assigns a
per-conversation sequence number on the conversation's shard, and durably appends to a sharded,
replicated, append-only message store keyed by conversation id, with a client-generated message id
giving idempotency, and then hands off delivery to a queue-based fanout layer that pushes to
online gateways or simply advances the recipient's sync pointer if they are offline. Presence and
typing are deliberately ephemeral and never touch the durable store, while read and delivered
receipts are compressed into a single monotonic last-read sequence per user per conversation, which
removes roughly two million writes per second. And the whole read path is fronted by a
sharded-ownership cache with single-flight coalescing, jittered TTLs, and stale-serving, because
the system is read-dominated twenty to one and the hot conversation is the only real scaling risk.

**Interviewer:** Good. That is the system. Thank you.

**Candidate:** Thank you.

---

## Concepts to Review Alongside This Transcript

- [[websockets]] for the connection layer and the reconnection/heartbeat model
- [[message-queue]] and [[kafka-architecture]] for the delivery decoupling and ordering key choice
- [[delivery-semantics]] for why at-least-once plus idempotency is the right contract
- [[idempotency]] and [[idempotent-consumer]] for the three-layer dedupe argument
- [[sharding]] and [[shard-key]] for conversation-id partitioning and the fanout consequence
- [[hotspot-handling]] for the 500,000-member group, and why jitter is the wrong fix there
- [[backpressure]] and [[load-shedding]] for the slow-consumer escalation ladder
- [[database-replication]] and [[replication-lag]] for failover and the receipt-write lag story
- [[strong-vs-eventual-consistency]] for the per-field staleness budget framing
- Study separately: Push Notifications (APNs/FCM), E2E encryption, and mobile offline sync
  conflict resolution
