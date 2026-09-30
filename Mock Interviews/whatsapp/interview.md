---
title: WhatsApp - Mock Interview Transcript
status: active
tags: [hld, mock, whatsapp, transcript]
---

# WhatsApp - Mock Interview Transcript

Forty-five minutes. The interviewer runs the messaging backbone for a
consumer chat product. He is going to attack the delivery semantics, the
gateway fleet, and anything that claims to be exactly-once.

## Act 1 - Opening, Scope, and Requirements

Interviewer: Design a real-time messaging platform in the style of WhatsApp.
Two billion registered users, nine hundred million daily actives, sixty billion
messages a day. Eight percent of those are in groups, average group size four,
long tail up to a thousand. Average message one kilobyte, ten percent carry
media averaging two hundred kilobytes. Peak to average is two and a half, with a
two times burst on top for a global event. End-to-end encrypted, multi-device.
Forty-five minutes. Questions first.

Candidate: I have about eight, and I want to flag at the start that I think
exactly-once delivery is the requirement everyone states and nobody can have.

Interviewer: Go ahead.

Candidate: First, what is the observable delivery contract you want? I am
proposing at-least-once delivery with client-side deduplication by a
client-generated message id. Exactly-once end to end is not achievable across a
mobile radio, the internet, a broker, a database and a handset. What I can give
you is exactly-once effect: the user never sees a duplicate and never sees a
lost message.

Interviewer: Accepted. Duplicates and reordering are both visible bugs, though,
so do not just tell me the client will sort it out.

Candidate: Understood. I will be precise about where the ordering comes from
rather than relying on the client.

Interviewer: Fine. Next.

Candidate: Second, does order matter within a conversation only, or across
conversations? I am assuming a total order within a conversation and no order
across conversations, which is a big simplification and I want it confirmed
because a global order across two billion users is a different product.

Interviewer: Within a conversation only. Confirmed.

Candidate: Third, the threat model. Is a malicious server operator in scope, or
only a server compromise? Because if a malicious operator is in scope then
metadata is also sensitive, and I have to think about traffic analysis, not
just message bodies.

Interviewer: Assume a determined operator who controls the servers. Message
bodies are out of reach. Metadata is a known cost that the product has accepted.

Candidate: Then I will say the following out loud as a design consequence:
server-side search, server-side content moderation, and server-side backup
that the user can restore from without re-verifying keys are all impossible.
I will propose substitutes for each later, and I want you to hold me to it
because a candidate who designs a search index over encrypted messages has not
understood the constraint.

Interviewer: Good, that is the right thing to say unprompted.

Candidate: Fourth, multi-device. If a user links a laptop today, can the laptop
read the last year of history, and can I revoke the laptop?

Interviewer: Linking a device does grant it access to existing history after a
security code verification step. Revocation removes its future access. Already
delivered messages we treat as unrecoverable and the UI warns the user.

Candidate: Understood, and that last sentence is the honest answer. I cannot
delete what a device has already decrypted. The only technical control is
crypto-shredding, which prevents future reads, and it does nothing for
retrospective compromise. I will design forward-only revocation and be explicit
that it is not a guarantee.

Interviewer: Fifth, presence. Exact or approximate?

Candidate: Approximate, strongly. I would like a 30 to 60 second TTL and
best-effort delivery of the transition. Exact online status requires a write per
state transition on a key that a user's entire contact list reads, and it buys
almost nothing. If the product insists on last-seen accuracy to the minute, I
can do that with a separate, lower-frequency, durable path that does not run on
the connection heartbeat.

Interviewer: Sixty seconds TTL is fine.

Candidate: Sixth, the battery complaint. Is it heartbeat frequency or connection
lifetime, because I think you have two different complaints and I would fix
them differently.

Interviewer: Both. Battery on Android, and dropped connections on mobile
networks when the app sits in the background.

Candidate: Then the fix is a two-tier liveness model. Foreground app gets a
WebSocket with an application-level ping every 25 seconds. Background gets an
exponentially backing-off ping starting at 60 seconds up to 15 minutes, and
push notification is the actual delivery mechanism while backgrounded, not the
socket. I will design the gateway to treat a connection as soft-dead on missed
heartbeats rather than holding it.

Interviewer: Seventh and last. Retention?

Candidate: Product decision, but I will propose it and you can correct me. Hot
window of seven days on fast storage, ninety days warm, and beyond that an
erasure-coded cold archive, with per-conversation trimming so that any single
conversation never exceeds five hundred messages regardless of age. Most
conversations are dead within days and keeping them forever is the single
largest avoidable cost in this system.

Interviewer: That is roughly what we do. Good.

## Act 2 - Estimation

Interviewer: Numbers. Go.

Candidate: Writes. Sixty billion messages a day divided by eighty-six thousand
four hundred seconds is six hundred and ninety-four thousand messages per second
on average. Peak at two and a half times average is one point seven four million
per second. And then the global event burst doubles that to about three and a
half million per second. I am going to design for three and a half million
message writes per second sustained for a few minutes.

Candidate: But writes are not the real number. Deliveries are. A one-to-one
message has to reach every device of the recipient, average one point three
devices, so forty-eight billion one-to-one messages a day times one point three
is sixty-two billion deliveries. A group message, twelve billion a day, goes to
four members, so forty-eight billion deliveries. Total about a hundred and ten
billion deliveries a day, which is one point two seven million per second
average and roughly three point two million per second at peak, before the
event burst.

Candidate: Connections. Nine hundred million daily actives, and I will assume
sixty percent are concurrently online at peak, so five hundred and forty million
concurrently active users, times one point three devices, is about seven hundred
million concurrent WebSocket connections. At the burst, nine hundred million.

Candidate: Here is the number that determines the shape of this system. Each
gateway node holds twenty thousand connections at roughly fifteen kilobytes of
resident memory per connection, which covers socket buffers, the session record,
the encryption handshake state and a small outbound queue. That is three hundred
megabytes of connections per node, so seven hundred million connections is
eleven terabytes of memory, which is thirty-nine thousand nodes.

Candidate: And now the insight I want you to take away. Peak message writes are
one point seven four million per second spread over thirty-nine thousand nodes
is forty-five messages per second per node. Peak deliveries are about eighty per
second per node. So each node does roughly a hundred and thirty events per
second while holding twenty thousand idle connections. The CPU is idle. This
fleet is sized entirely by connection count and memory, not by message
throughput, and the practical constraints are file descriptor limits, ephemeral
port exhaustion on the acceptor, and kernel socket memory, not application
throughput. If you only estimate requests per second you will size this system
ten times too small.

Interviewer: Storage.

Candidate: Text. Sixty billion messages at one kilobyte is sixty terabytes a
day, twenty-two petabytes a year. Media, ten percent of messages, six billion
times two hundred kilobytes is one point two terabytes a day, four hundred and
thirty-eight terabytes a year. Total ingestion about twenty-two and a half
petabytes a year of new data, call it sixty-two terabytes a day.

Candidate: With three copies for durability that is sixty-seven petabytes a year
if I keep everything. I am not going to keep everything on fast storage, so
here is the working set. Hot window of seven days is four hundred and twenty
terabytes of text and media, times three for replicas, which is about one point
three petabytes. Warm to ninety days is about seven and a half terabytes a day
times eighty-three, call it six hundred and twenty terabytes, replicated to one
point nine petabytes. Beyond ninety days, cold erasure-coded at one and a half
times, which is roughly thirty-four petabytes a year.

Candidate: The important structural point is that the hot set is bounded by the
retention window, not by total history, and the total history grows without
limit. That is the whole reason the two tiers can be different storage classes
rather than one cluster that keeps growing. And it is why the per-conversation
trim matters, because a trim policy is the only thing that keeps a single very
active group from dominating the hot set.

Candidate: Conversation and membership scale. Assume one point five billion
active conversations. If I shard the message store into eight thousand shards
keyed by conversation id, that is about one hundred and ninety thousand
conversations per shard, and the seven-day hot set of one point three petabytes
over eight thousand shards is about a hundred and sixty gigabytes per primary.
That is a healthy primary size, big enough to be efficient, small enough to
reshard in hours rather than days.

Candidate: And the sequencer: one point seven four million sequence
allocations per second over eight thousand shards is two hundred and eighteen
per second per shard. Trivial. So the ordering machinery is not a scaling
problem at all, which is a relief, and I would rather spend the complexity
budget on fanout than on ordering.

## Act 3 - API and Data Design

Interviewer: APIs and schemas. And I want to see the message row, because that
is where most designs are wrong.

Candidate: Schemas first, then APIs.

Candidate: CREATE TABLE messages (
Candidate:   conversation_id BIGINT      NOT NULL,
Candidate:   seq             BIGINT      NOT NULL,   -- server-assigned, strictly increasing per conversation
Candidate:   sender_id       BIGINT      NOT NULL,
Candidate:   client_msg_id   CHAR(26)    NOT NULL,   -- ULID generated by sender before encryption
Candidate:   ciphertext      BLOB        NOT NULL,   -- includes media key, sender identity key id, timestamp
Candidate:   media_ref       VARCHAR(512) NULL,     -- object key + sha256, never a public URL
Candidate:   type            TINYINT     NOT NULL,   -- 1 text, 2 image, 3 voice, 4 doc, 5 system
Candidate:   reply_to_seq    BIGINT      NULL,
Candidate:   edited_at       TIMESTAMP(6) NULL,
Candidate:   created_at      TIMESTAMP(6) NOT NULL,   -- server receive time, for retention only
Candidate:   PRIMARY KEY (conversation_id, seq),
Candidate:   UNIQUE KEY uq_sender_client (sender_id, client_msg_id)
Candidate: );
Candidate: Shard: HASH(conversation_id). Two indexes and both matter. The
Candidate: primary key gives me ordered history for a conversation, which is the
Candidate: dominant read, and the unique key on (sender_id, client_msg_id) is
Candidate: what makes duplicate submission a no-op instead of a double message.

Candidate: That unique key is the whole at-least-once story on the write side. A
retried send from a flaky mobile connection hits the constraint, I return the
existing row, and the sender sees success. I do not need the sender to be smart
about retries and I do not need a distributed lock.

Candidate: CREATE TABLE conversation_members (
Candidate:   conversation_id BIGINT NOT NULL,
Candidate:   user_id         BIGINT NOT NULL,
Candidate:   joined_seq       BIGINT NOT NULL,   -- first seq visible to this member
Candidate:   left_seq         BIGINT NULL,      -- non-null means a tombstone, not a hard delete
Candidate:   role             TINYINT NOT NULL,  -- member, admin, superadmin
Candidate:   muted_until      TIMESTAMP(6) NULL,
Candidate:   PRIMARY KEY (conversation_id, user_id),
Candidate:   KEY idx_user (user_id)
Candidate: );
Candidate: Shard: HASH(conversation_id) with a mirrored index sharded by
Candidate: user_id, because "list my conversations" is the app's first screen and
Candidate: must not be a scatter-gather across eight thousand shards.

Candidate: CREATE TABLE devices (
Candidate:   user_id         BIGINT NOT NULL,
Candidate:   device_id       BIGINT NOT NULL,
Candidate:   identity_key    VARBINARY(64) NOT NULL,
Candidate:   signed_prekey   VARBINARY(64) NOT NULL,
Candidate:   prekey_signature VARBINARY(64) NOT NULL,
Candidate:   one_time_prekeys BLOB     NOT NULL,   -- batch, replenished on fetch
Candidate:   status          TINYINT NOT NULL,     -- 1 active, 2 unlinked
Candidate:   created_at      TIMESTAMP(6) NOT NULL,
Candidate:   PRIMARY KEY (user_id, device_id)
Candidate: );
Candidate: Shard: HASH(user_id). This is the prekey bundle store. The server is
Candidate: a relay for public keys, which is consistent with it never being able
Candidate: to read a message.

Candidate: CREATE TABLE ratchet_state (
Candidate:   user_id      BIGINT NOT NULL,
Candidate:   device_id    BIGINT NOT NULL,
Candidate:   contact_id   BIGINT NOT NULL,   -- or conversation_id for group send-keys
Candidate:   state        BLOB     NOT NULL,   -- serialized ratchet chain
Candidate:   message_index INT     NOT NULL,
Candidate:   updated_at   TIMESTAMP(6) NOT NULL,
Candidate:   PRIMARY KEY (user_id, device_id, contact_id)
Candidate: );
Candidate: Shard: HASH(user_id), durable, three replicas, no TTL eviction.
Candidate: This is the row most candidates put in a cache and that is a mistake.
Candidate: If ratchet state is lost, the session is broken and you need a key
Candidate: re-exchange, which is invisible in a one-to-one chat and visibly
Candidate: disruptive in a group because it forces a group rekey to every member.
Candidate: Treat it as correctness data, not as a cache.

Candidate: CREATE TABLE delivery_state (
Candidate:   conversation_id BIGINT NOT NULL,
Candidate:   seq             BIGINT NOT NULL,
Candidate:   device_id       BIGINT NOT NULL,
Candidate:   delivered_at    TIMESTAMP(6) NULL,
Candidate:   read_at         TIMESTAMP(6) NULL,
Candidate:   PRIMARY KEY (conversation_id, seq, device_id)
Candidate: );
Candidate: Per device, not per user. The requirement is that each of a user's
Candidate: three devices acknowledges independently, and a single "delivered
Candidate: boolean" per conversation cannot express that or produce correct
Candidate: ticks. This table is written on the receipt path, which is roughly
Candidate: one write per delivery, so it is the highest volume table in the
Candidate: system after messages, and it is the one I will trim hardest, keeping
Candidate: it for the last thousand sequences of a conversation and rolling older
Candidate: receipts into a compacted high-water mark.

Candidate: Actually, the better design is a high-water mark plus exceptions. Most
Candidate: conversations have strictly monotonic delivery, so `delivered_upto_seq
Candidate: BIGINT` per device plus a sparse exception table for out-of-order
Candidate: cases is two orders of magnitude smaller. I would build the exception
Candidate: table and only fall back to per-message rows when I detect a gap.

Candidate: Presence is not in the database. CREATE conceptually: key
Candidate: `presence:{user_id}`, a hash of {state, last_seen_epoch, device_count},
Candidate: TTL 60 seconds, in Redis, with the gateway refreshing it on heartbeat.
Candidate: Best-effort is fine here, so a single replica is acceptable, and the
Candidate: cost of losing a replica is that presence is wrong for up to 60
Candidate: seconds.

Candidate: Now APIs.

Candidate: WS /ws  the long-lived connection. Connect carries an auth token, the
Candidate: client receives a connection id and a resume token, then the protocol
Candidate: carries send, ack, receipt, presence, typing and sync frames.
Candidate: Frame format is length-prefixed protobuf, not JSON, because at a
Candidate: hundred and thirty events per second per node with small frames,
Candidate: JSON parsing shows up in profiles and protobuf is roughly a third of
Candidate: the bytes. Study separately: binary protocol encoding with length
Candidate: prefixes, and the framing rules that let you multiplex send, ack and
Candidate: control on one socket without a head-of-line blocking bug.

Candidate: POST /v1/messages
Candidate:   request:  { conversation_id, client_msg_id, ciphertext, type, reply_to_seq?, media_ref? }
Candidate:   response: { conversation_id, seq, server_time, state: "ACCEPTED"|"DUPLICATE" }
Candidate:   semantics: idempotent on client_msg_id. ACCEPTED means durably stored on a quorum. DUPLICATE means we already have it and here is the original seq, which is how a client retry learns the ordering it was missing.

Candidate: GET /v1/conversations/{id}/messages?after_seq={n}&limit=100
Candidate:   response: { messages: [...], has_more, server_time }
Candidate:   semantics: this is the resync and history endpoint both, and after_seq rather than offset is not a nicety, it is what makes reconnect cheap. A client that was offline for an hour sends after_seq equal to its last contiguous sequence and gets a bounded delta, not a scan.

Candidate: GET /v1/conversations            -> the app's first screen, from the user-keyed membership index
Candidate: GET /v1/conversations/{id}/presence  -> batched, max 200 user ids per call
Candidate: POST /v1/presence               -> coarse opt-in status, separate from the heartbeat
Candidate: POST /v1/media/uploads         -> presigned, same as photos in [[../instagram/interview|Instagram]]
Candidate: POST /v1/devices                -> link a device, returns this device's prekey bundle
Candidate: POST /v1/devices/{id}/sessions -> fetch a contact's prekey bundle
Candidate: DELETE /v1/devices/{id}         -> unlink, revokes prekeys
Candidate: POST /v1/reports               -> user-invoked report, the only path plaintext leaves a device

Candidate: Notice there is no send-receipt websocket topic for the sender. Receipts
Candidate: are pulled and batched, over a separate topic, because a receipt storm
Candidate: from a high-traffic conversation is a self-inflicted amplification
Candidate: loop and there is no user-visible latency benefit to pushing it
Candidate: instantly.

## Act 4 - High-Level Architecture

Interviewer: Draw it.

Candidate: Four tiers plus a backbone. I will read it as: presence routing, the
send path, the durable core, and the delivery path.

Candidate: +---------------------------------------------------------------------------+
Candidate: |                     CLIENTS  (phone, tablet, desktop)                   |
Candidate: |            WebSocket, TLS, length-prefixed protobuf frames             |
Candidate: +-----------------------------------+-----------------------------------+
Candidate:                                     |
Candidate:                                       v
Candidate: +---------------------------------------------------------------------------+
Candidate: |                        GATEWAY TIER  (~39,000 nodes)                    |
Candidate: |  global anycast LB  ->  hash ring: hash(user_id) -> gateway pool         |
Candidate: |  per node: 20k connections, 300MB resident, 100-200 send qps            |
Candidate: |  owns: session record, ratchet handshake relay, presence heartbeat      |
Candidate: |  does NOT own: message bodies, ordering, delivery queue of record       |
Candidate: +-----------------------------------+-----------------------------------+
Candidate:                                     |
Candidate:          +--------------------------+--------------------------+
Candidate:          |                          |                          |
Candidate:          v                          v                          v
Candidate: +------------------+   +----------------------+   +------------------+
Candidate: | MESSAGE          |   | PRESENCE / SESSION   |   | EVENT BACKBONE   |
Candidate: | STORE            |   | Redis + directory     |   | Kafka            |
Candidate: | 8000 shards      |   | presence TTL 60s      |   | MessageAccepted  |
Candidate: | conv_id -> shard |   | conn -> gateway       |   | ReceiptUpdated   |
Candidate: | primary + 2 rep  |   | online device count   |   | PresenceChanged  |
Candidate: +--------+---------+   +----------------------+   +--------+---------+
Candidate:          |                                              |
Candidate:          |  +-------------------+                      |
Candidate:          |  |  MEDIA            |                      |
Candidate:          |  |  object storage   |  presigned, direct    |
Candidate:          |  |  never via gateway |  client to client     |
Candidate:          |  +-------------------+                      |
Candidate:          v                                              v
Candidate: +------------------+                        +---------------------+
Candidate: | SEQUENCER        |                        | DELIVERY WORKERS   |
Candidate: | colocation: one  |------------------------|  - resolve members  |
Candidate: | per message shard|                        |  - push "new msgs"  |
Candidate: | batched ranges   |                        |  - receipts topic   |
Candidate: +------------------+                        +---------------------+

Candidate: The three things I want you to notice. One, media never touches the
Candidate: gateway, same as photos. Two, the gateway holds no durable message
Candidate: state, so a gateway node is disposable and losing thirty-nine of them
Candidate: is a capacity event, not a data event. Three, the sequencer is
Candidate: colocated with the message shard, so ordering never crosses a network
Candidate: hop to a shared service, which is what keeps per-conversation total
Candidate: order cheap.

## Act 5 - Request and Data Flow

Interviewer: Send path. Every hop.

Candidate: One, the sender's device encrypts with the double ratchet and sends a
Candidate: send frame with client_msg_id, ciphertext, type and any media_ref
Candidate: over its WebSocket to its gateway.

Candidate: Two, the gateway does almost nothing. It authenticates the frame,
Candidate: appends it to a per-conversation in-flight buffer keyed by the
Candidate: conversation hash, and forwards to the sequencer for that conversation.
Candidate: The gateway does not assign the sequence and does not write to the
Candidate: database, and that is a deliberate division so that the largest
Candidate: tier in the system has no correctness responsibility.

Candidate: Three, the sequencer allocates the next sequence number for that
Candidate: conversation. Here is an optimization that matters: the sequencer
Candidate: hands out ranges, not single numbers, allocating up to sixty-four at a
Candidate: time per conversation per sender connection. That turns a round trip
Candidate: per message into a round trip per sixty-four messages, so a user
Candidate: sending a fast burst is not paying a network hop per message. Two
Candidate: devices of the same user get ranges from the same counter, so order is
Candidate: still global to the conversation, not per device. Unused numbers from
Candidate: a crashed device are simply gaps, and gaps are harmless because clients
Candidate: track highest contiguous sequence, not a count.

Candidate: Four, the write goes to the message store shard, and I acknowledge to
Candidate: the sender only after a quorum, which I will defend in a moment. The
Candidate: unique constraint on (sender_id, client_msg_id) is checked here, and a
Candidate: duplicate returns DUPLICATE with the original seq.

Candidate: Five, the shard writes a MessageAccepted event to Kafka. This is the
Candidate: durability and fanout boundary. The gateway pushes the ack frame to
Candidate: the sender's device at this point, so the sender's tick appears after
Candidate: the quorum, not before.

Candidate: Six, delivery workers consume MessageAccepted, resolve the
Candidate: conversation's members, look up each member's currently connected
Candidate: gateway from the session directory, and push a notification frame.

Candidate: And here is the decision I want to defend because it saves enormous
Candidate: memory: the frame I push is a pointer, not a payload. It carries
Candidate: conversation_id, sender_id, the new seq, and the type. The device then
Candidate: calls GET messages?after_seq to fetch content. So the gateway never
Candidate: holds a message body, a one kilobyte body times seven hundred million
Candidate: connections is seven hundred gigabytes of buffer that I no longer have
Candidate: to reason about, and a client that fetches by sequence can never
Candidate: receive a stale body. The cost is one extra round trip, which on a
Candidate: connection the client already has open is a few milliseconds, and it
Candidate: is paid on a network the client is already paying for.

Candidate: Seven, the receiving device fetches, decrypts locally, and sends a
Candidate: delivered receipt. That is a write to delivery_state and a
Candidate: ReceiptUpdated event. The sender's devices poll the high-water mark in
Candidate: batch.

Interviewer: Receive path for an offline user.

Candidate: There is no separate offline path, and that is the point. The
Candidate: delivery worker's session lookup returns "not connected", and that is
Candidate: where the work ends. Nothing is buffered on behalf of the offline
Candidate: user, no per-offline-user queue exists anywhere, and the message is
Candidate: already durably in the store. When the user reconnects, the gateway
Candidate: sees the resume token, reads the last acknowledged sequence from the
Candidate: client, and the client immediately issues its sync request. The
Candidate: offline case is not a special case, it is just a sync that starts
Candidate: later.

Interviewer: And a user who has been offline for a year?

Candidate: Gets the last five hundred messages of that conversation, because of
Candidate: the trim policy, plus a marker telling them history was trimmed. The
Candidate: alternative, keeping everything, is twenty-two petabytes a year
Candidate: instead of one and a half, and the user-visible difference is
Candidate: essentially nil because nobody reads a year-old chat. I will say
Candidate: plainly that this is a business trade, not a technical truth, and if
Candidate: the product wanted full history the architecture is unchanged, only the
Candidate: retention tiering is removed.

Interviewer: Good. Now delivery semantics. Walk me through what happens when a
Candidate: message is delivered twice, or delivered out of order.

Candidate: Twice. The client keeps a bounded set of recently seen message ids,
Candidate: keyed by client_msg_id, per conversation, and drops any frame whose id
Candidate: is already present. Bounded at a few hundred per conversation. The
Candidate: duplicate is visible for at most one round trip and is dropped on
Candidate: arrival. Note that I dedupe on the sender-generated id, not on my own
Candidate: seq, because if the server assigned a fresh seq to a duplicate then the
Candidate: server did not dedupe at all.

Candidate: Out of order. The client buffers by seq and renders contiguous runs,
Candidate: fetching any gap. A gap is normally a withheld message, so on a gap
Candidate: the client issues a bounded sync for that conversation. If the gap is
Candidate: a permanent hole from a crashed sender holding a range, the message
Candidate: store resolves it: I maintain a per-conversation "hole registry" as a
Candidate: sparse set of allocated but unwritten ranges, so the client can be told
Candidate: "these sequences will never exist" instead of waiting forever. Without
Candidate: that, range batching plus a device crash produces a client that hangs
Candidate: on an invisible gap, and I have seen exactly this bug.

Candidate: And the contract, stated once: at-least-once on every internal hop,
Candidate: idempotent at the client and enforced at the store by a unique
Candidate: constraint, which together give exactly-once effect. I do not claim
Candidate: exactly-once delivery and I would push back on any design that does.

Interviewer: Encryption. Now I want to hear the protocol named properly, and I
want to know what happens on multi-device.

Candidate: X3DH for the initial key agreement, which uses a prekey bundle per
Candidate: device that the server stores and hands out on request, and the double
Candidate: ratchet for the ongoing session, which gives forward secrecy and
Candidate: post-compromise security, with a separate ratchet chain per device pair.
Study separately: the Double Ratchet and Signal Protocol key hierarchy, message
Counter: numbers, skipped-key storage and the header encryption that hides the
Candidate: sender from a malicious server, because the operator in your threat
Candidate: model is watching metadata and the sender field is the most valuable
Candidate: thing in it.

Candidate: Multi-device has the hard consequence. Every device has its own
Candidate: identity key and its own prekey bundle, and a message to a user is
Candidate: encrypted separately to each of their devices. So one logical message
Candidate: becomes N ciphertexts with N ratchet sessions, and the server fans out
Candidate: N of them. That multiplies encryption cost and key state by the device
Candidate: count, average one point three, worst case five, and it is the main
Candidate: reason device count is capped.

Candidate: Group chats use a sender-key construction. The sender generates one
Candidate: symmetric key per group, wrapped separately to each member's device,
Candidate: so encrypting to a thousand members is one symmetric encryption of the
Candidate: body plus a thousand key wraps, not a thousand full encryptions. The
Candidate: nasty part is membership change: adding a member requires rotating the
Candidate: sender key and distributing it to all existing members, which is a
Candidate: thousand-message event triggered by one user action. Removing a member
Candidate: does not require rotation but does not help either, because the removed
Candidate: member already has the old key. Study separately: CRDT-based
Candidate: convergence, because for pinned messages and a collaboratively edited
Candidate: group name, two offline admins editing at once is a real conflict and
Candidate: [[crdt|CRDTs]] are how you avoid a lock.

Interviewer: What does the server give up?

Candidate: Search across messages, which I would otherwise put in a full-text
Candidate: index. Moderation, which I replace with user-invoked reporting where
Candidate: the reporting device explicitly attaches the offending plaintext, plus
Candidate: metadata-based abuse detection, which is a great deal more effective
Candidate: than people assume: a new account messaging ten thousand strangers, or
Candidate: a burst pattern that matches a sockpuppet network, is visible in
Candidate: metadata alone. And server-side multi-device restore, because restoring
Candidate: history to a new device requires keys the server does not have, so that
Candidate: flow is a QR-style device-to-device transfer, which is a genuinely
Candidate: worse product and is the real cost of end-to-end encryption.

## Act 6 - Caching, Sharding, Replication, and Consistency

Interviewer: Sharding and replication. And the shard key for a one-to-one
conversation, because people get that wrong.

Candidate: The conversation id. For a group it is a random id. For a one-to-one
Candidate: I derive it deterministically from the two user ids, sorted and
Candidate: hashed, so both participants compute the same id with no lookup. That
Candidate: matters because otherwise every send requires a conversation resolution
Candidate: round trip before I can route, and at three and a half million sends per
Candidate: second that lookup is a real cost and a real latency floor. The
Candidate: downside is that the id leaks the fact that a particular pair of users
Candidate: has a conversation to anyone who guesses, which I would normally
Candidate: address with a per-pair rotating salt, and I flag it here because in
Candidate: your threat model the operator sees ids and this is a genuine weakness.

Candidate: Eight thousand shards, HASH(conversation_id) with a directory, virtual
Candidate: shards from day one, primary plus two replicas in the same region.
Candidate: Membership is a separate concern: I shard the membership table by
Candidate: conversation id and keep a user-keyed index, because the app opens on
Candidate: "list my conversations" and that must be a single-shard read.

Candidate: Cross-shard reads: opening a conversation for the first time needs the
Candidate: last twenty messages across however many of a thousand group members'
Candidate: devices, which is one shard for the conversation but potentially
Candidate: several for the member devices. I keep device registry reads on a
Candidate: separate store sharded by user id, and I accept the cross-shard hop,
Candidate: bounded to a few, cached for seconds. Study separately:
Candidate: [[cross-shard-queries|Cross-Shard Queries]] for the fan-in patterns,
Candidate: because this is the one place a group read is genuinely multi-shard.

Candidate: Replication: primary plus two replicas, and I write with a quorum of
Candidate: two, meaning the primary and one replica, before I acknowledge. That
Candidate: costs about five milliseconds of extra write latency and buys survival
Candidate: of a single node failure with no acknowledged-message loss. Study
Candidate: separately: [[replication-overhead|Replication Overhead]] and
Candidate: [[quorum|Quorum]], and be able to state the R plus W greater than N
Candidate: read guarantee that follows, because the history read also needs to be
Candidate: a quorum read or I will serve a client a gap that a replica has not
Candidate: replicated yet.

Interviewer: Replica lag, specifically. An online recipient is missing messages.

Candidate: Three concrete cases. One, the delivery worker reads the session
Candidate: directory from a Redis replica and concludes the user is offline when
Candidate: they just came online, so the push is skipped. The message is not
Candidate: lost, the client syncs on its own reconnect, but the delivery is late.
Candidate: My fix is to read the session directory from the primary, or a quorum
Candidate: read, for this specific lookup, because the lookup is cheap and
Candidate: liveness-sensitive. Presence and session routing are the two places
Candidate: where reading a stale replica is genuinely wrong rather than merely
Candidate: slow.

Candidate: Two, a history read served from a replica that has not caught up
Candidate: returns fewer messages than after_seq implies exist, and the client
Candidate: concludes has_more is false and stops. My fix is the same one I used for
Candidate: writes, a quorum read, plus a server_time watermark in the response so
Candidate: the client can tell a gap from the end of history.

Candidate: Three, a receipt read from a replica that lags shows a lower seq than
Candidate: was actually read, so the sender's ticks go backwards. Receipts are the
Candidate: one place I would accept eventual reads and simply never decrement a
Candidate: tick client side, so a stale read is invisible.

Interviewer: Consistency. Give me the table and the principle behind it.

Candidate: Strong: message acceptance, because a sender's tick means stored. The
Candidate: duplicate-suppression constraint, because it is what makes idempotency
Candidate: real. Device registry and ratchet state, because losing them breaks
Candidate: sessions. Membership, because seeing a group you left is a visible
Candidate: correctness bug.

Candidate: Eventual with a bound: delivery to an online recipient, sub-second.
Candidate: Receipts, a few seconds. Presence, sixty seconds by policy. Trending
Candidate: emoji reactions, a few seconds. Message editing propagation, sub-second
Candidate: per conversation.

Candidate: The principle: the message body is immutable and the server's job is
Candidate: to durably record and deliver it, so the only thing that truly needs
Candidate: to be strong is the moment of acceptance and the order of acceptance.
Candidate: Everything observable after that is a notification that a fact exists,
Candidate: and facts do not change. That is a much smaller consistency surface
Candidate: than a typical database design, and it is a direct consequence of
Candidate: choosing a system where the payload is opaque bytes to the server.

Interviewer: Availability and failure. Primary message store shard dies. Then a
Candidate: gateway dies. Then the recipient's socket is slow.

Candidate: Message store primary dies. Health checks detect it, the role is
Candidate: promoted to a replica with an epoch bump so the old primary is
Candidate: fenced, otherwise I have two writers and every duplicate suppression
Candidate: constraint is now being enforced by two nodes independently. Fencing
Candidate: is the answer, and it is the single most commonly skipped step in this
Candidate: scenario. My RTO is about twenty seconds for a shard, and my RPO is
Candidate: zero for acknowledged messages because I acknowledged on a quorum of
Candidate: two, so at least one surviving replica has them. That RPO of zero is
Candidate: the entire reason I chose quorum over primary-only acknowledgement,
Candidate: and it is worth five milliseconds of latency to every message in the
Candidate: system.

Candidate: In-flight sends, the messages between the client's send frame and the
Candidate: quorum, are lost. The client retries with the same client_msg_id, so
Candidate: if the first attempt actually landed, the retry gets DUPLICATE with
Candidate: the original seq, and if it did not, it gets a fresh seq. Either way
Candidate: no duplicate is visible and no message is lost. That is why the client
Candidate: generates the id before sending, and why I refused to let the server
Candidate: generate it.

Candidate: Gateway node dies. Every connection on it drops. Clients reconnect with
Candidate: exponential backoff and jitter to avoid a synchronized thundering herd
Candidate: at the LB, re-register in the session directory, and issue a sync with
Candidate: after_seq. Data impact is zero, because the gateway held no message
Candidate: bodies. Capacity impact is real: I need spare capacity for the
Candidate: reconnects, so I run at about seventy percent connection capacity and
Candidate: I pre-warm the replacement pool. The connection drop is visible to the
Candidate: user as a brief spinner, and a client-side queue of unsent messages
Candidate: with local disk persistence covers the gap.

Candidate: Slow recipient socket. This is backpressure, and I want to be precise
Candidate: because the intuitive answer, buffer the messages, is wrong at this
Candidate: scale. Seven hundred million connections times an unbounded buffer is
Candidate: an out-of-memory kill of a subset of my fleet. So: each session has a
Candidate: bounded outbound queue, say one megabyte. If it fills, I do not buffer
Candidate: and I do not drop messages, I disconnect that session and the client
Candidate: resyncs from the store with after_seq. I lose the notification, I keep
Candidate: the message, and the client is provably correct because the store is
Candidate: authoritative. Study separately: [[backpressure|Backpressure]] and
Candidate: [[load-shedding|Load Shedding]], and understand why "disconnect and
Candidate: resync" is a legitimate and often superior strategy to buffering when
Candidate: the source of truth is durable and the client can re-derive its state.

Candidate: The one thing I must not do is silently drop messages from a full
Candidate: queue, because a dropped message with no gap marker is a message the
Candidate: client will never know to look for.

Interviewer: Now the large group, and I want you to show the fanout math.

Candidate: Average group is four, so fanout is trivial. The problem is the tail.
Candidate: A thousand-member group, one message, is a thousand deliveries. Let me
Candidate: put a number on the danger. If one percent of group messages go to
Candidate: thousand-member groups, that is twelve billion group messages a day,
Candidate: one percent is a hundred and twenty million messages a day to big
Candidate: groups, which is one thousand three hundred and eighty-nine per second,
Candidate: times a thousand deliveries each, which is one point three nine million
Candidate: deliveries per second from that segment alone, plus it would mean
Candidate: writing a thousand delivery_state rows per message, so a hundred and
Candidate: thirty-nine million delivery writes per second. The storage tier would
Candidate: be the first thing to die, and it would die before the gateway tier.

Candidate: So a size threshold, same pattern as fanout elsewhere. Below 256
Candidate: members, materialize delivery rows eagerly, and push to every member's
Candidate: connected device. At or above 256, do not materialize. Write the
Candidate: message once, maintain a per-group activity high-water mark that every
Candidate: member's device reads, and push a single light "this group has new
Candidate: messages" notification per device rather than per message. Each device
Candidate: then pulls on demand and on a low-frequency poll.

Candidate: What I lose is immediacy for large groups, and I accept it because in
Candidate: a thousand-person group, two seconds versus fifty milliseconds is
Candidate: undetectable. What I save is unbounded. I would also apply a
Candidate: per-conversation rate limit, so a group cannot be used to amplify into
Candidate: the delivery tier, and that is [[rate-limiter|rate limiting]] as a
Candidate: data-structure problem rather than a per-user token bucket.

Interviewer: What are your actual bottlenecks, ranked.

Candidate: One, gateway connection capacity and memory. Thirty-nine thousand nodes
Candidate: and file descriptor and socket buffer pressure in the kernel. This is
Candidate: the first thing to fail and the hardest to fix quickly, because adding
Candidate: capacity means adding machines, not adding a cache.

Candidate: Two, the delivery write amplification from group fanout, addressed
Candidate: above with the 256 threshold and the high-water mark design.

Candidate: Three, sequence allocation for a single hot conversation, such as a
Candidate: broadcast-ish group receiving a message every few hundred milliseconds.
Candidate: The sequencer shards by conversation so this is one shard taking the
Candidate: whole conversation's traffic. It is cheap per operation, but the shard
Candidate: becomes a serialized point, and range batching helps here more than
Candidate: anywhere else. If it became real I would shard the sequence space
Candidate: into stripes and merge on read.

Candidate: Four, presence read amplification. A user with five hundred contacts
Candidate: opening a chat list issues a batched presence query, and if enough
Candidate: users do it at once, a small set of very large groups keys get very hot
Candidate: in Redis. I mitigate with batching at two hundred ids, a short TTL so
Candidate: it is nearly free to be wrong, and a per-request cap on ids.

Candidate: Five, the push provider. At one point two seven million deliveries per
Candidate: second across push networks, the external dependency is a hard ceiling
Candidate: and a hard availability risk, so I need a circuit breaker, a
Candidate: per-platform quota, and a graceful degradation to pure pull.

Candidate: Six, per-conversation history reads for a user with two hundred active
Candidate: conversations on app open, which is two hundred range scans. Mitigated
Candidate: by a materialized per-user conversation summary list so the first screen
Candidate: is one read, and history is fetched on demand.

## Act 7 - Trade-offs, Scaling, and Follow-ups

Interviewer: Trade-offs. The four that actually hurt.

Candidate: One, gateway affinity. I hash user_id to a gateway so that a user's
Candidate: devices land on one node and message routing is a local map lookup
Candidate: instead of a directory query on every frame, and so that presence
Candidate: heartbeats are localized. I paid for it with skewed load, since a
Candidate: user with five devices and heavy sending is a hotter node than
Candidate: average, and with the fact that a node loss drops exactly the sessions
Candidate: of its hash segment. I would mitigate with virtual nodes for even
Candidate: distribution and a replica set per segment for fast reassignment, and I
Candidate: would accept the cost because the alternative, a directory lookup on
Candidate: every frame, is on the hot path of a billion connections. That is
Candidate: [[sticky-sessions|Sticky Sessions]] bought with an explicit awareness
Candidate: of what sticky sessions cost, rather than adopted by accident.

Candidate: Two, push pointer versus push payload. Pointer costs the device one
Candidate: extra round trip and saves the gateway all message body memory and
Candidate: guarantees no stale body. I took the pointer, and the deciding factor
Candidate: was that a stale or re-ordered pushed body is a correctness problem
Candidate: while an extra round trip on an already-open connection is not.

Candidate: Three, durability versus acknowledgement latency. I acknowledge after
Candidate: a two-of-three quorum rather than after the primary, adding about five
Candidate: milliseconds to every message in the system and buying RPO zero
Candidate: against a single node failure. For text messaging, five milliseconds
Candidate: is below the perceptual floor. If the product were trading or
Candidate: order-entry I would still take the quorum, which tells you how much I
Candidate: actually believe in tuning this away.

Candidate: Four, encryption state in a cache versus in durable storage. Caching
Candidate: ratchet state is much cheaper and much faster. It is also a
Candidate: correctness bug generator, because an eviction is indistinguishable from
Candidate: a session break. I chose durable, three replicas, no eviction, and I
Candidate: accept the write amplification of persisting a state blob on every
Candidate: message.

Interviewer: Scaling to ten times.

Candidate: Connections scale by adding gateway nodes to the ring, which is a ring
Candidate: membership change and a rebalance, not a migration, because the ring
Candidate: is consistent-hashed. Message store scales by splitting the directory.
Candidate: The thing that does not scale linearly is encryption work, because
Candidate: ratchet operations are asymmetric cryptography and at ten times volume
Candidate: the sequencer and encryption tier become CPU-bound rather than
Candidate: memory-bound, which inverts the shape of the whole problem. I would
Candidate: expect hardware acceleration for the symmetric bulk of the work and
Candidate: careful batching, and I would expect encryption to be the first thing
Candidate: to need its own scaling story.

Candidate: And the other inversion: at ten times, the hot retention window is
Candidate: still bounded but the cold archive is thirty-four petabytes a year, and
Candidate: at some point the erasure-coded archive tier becomes the dominant cost
Candidate: and the dominant restore-time problem. I would want to design the
Candidate: archive for cheap verify and slow restore, because an archive nobody can
Candidate: restore is a compliance liability, not a saving.

Interviewer: Operational questions, quickly. Deploy to thirty-nine thousand
Candidate: nodes with nine hundred million live connections.

Candidate: Rolling deploy with connection draining. Mark a node as draining, stop
Candidate: accepting new connections, let live sessions finish, and when a
Candidate: connection closes for any reason the client reconnects and lands
Candidate: elsewhere. Draining a node is a one-to-three minute operation at that
Candidate: connection count, so a deploy is a slow rolling operation over
Candidate: hours, not minutes, and I would accept that and use feature flags to
Candidate: decouple behaviour change from deploy. The alternative, killing
Candidate: connections, would put every client on the node into a reconnect storm
Candidate: simultaneously. Study separately: [[connection-draining|Connection
Candidate: Draining]] and [[feature-flags|Feature Flags]] as the two things that
Candidate: make a fleet this size shippable.

Interviewer: And if you lose a whole availability zone.

Candidate: Gateways in that zone go with it. The ring drops those nodes, clients
Candidate: reconnect to the surviving zones, and message store primaries with their
Candidate: replicas promote zone-wide. Because the message store has replicas in
Candidate: three zones and acknowledges on a quorum, I lose no acknowledged
Candidate: messages. What I do lose is latency during the reconnect storm, and the
Candidate: reconnect storm is the dangerous part, so I use backoff with jitter,
Candidate: admission rate limiting on the edge, and a deliberate, brief
Candidate: degradation where a client that cannot reconnect falls back to
Candidate: long-poll rather than a socket, and to pull-sync only. RTO for a zone
Candidate: is under a minute, RPO zero, and the cost is a degraded experience for
Candidate: the duration of the reconnect.

Interviewer: What is your monitoring story?

Candidate: Per-node connection count and memory, because those are the constraint
Candidate: I identified and they are the leading indicators. Outbound queue depth
Candidate: per session, and the count of sessions disconnected for backpressure,
Candidate: which is a user-visible symptom that no error rate shows. Duplicate
Candidate: suppression rate, which rising means client retries or a producer
Candidate: retry bug. Out-of-order and gap-fill rates per conversation, which
Candidate: should be near zero and any spike means the sequencer or the hole
Candidate: registry is broken. Sync-after-reconnect latency, which is the honest
Candidate: proxy for how the system feels after an incident. And the
Candidate: delivery-to-ack latency broken into encryption, sequencer, quorum
Candidate: write, Kafka, and push, so that when the p99 moves I know which of five
Candidate: stages moved. Distributed tracing on a sampled percentage of sends,
Candidate: correlated with client_msg_id, which is the only identifier that exists
Candidate: end to end across the client, the gateway, the store and the receipt.

Interviewer: Anything you would tell a junior to go study separately?

Candidate: Five. Study separately: the Double Ratchet and Signal Protocol,
Candidate: including prekey distribution, skipped keys, and header encryption,
Candidate: because if I get that wrong the design is decorative. Study separately:
Candidate: [[exactly-once-effect|Exactly Once Effect]], which is the reconciliation
Candidate: of at-least-once delivery with idempotency, and it is the concept that
Candidate: turns my delivery semantics from a claim into a proof. Study separately:
Candidate: [[stateless-vs-stateful-services|Stateless versus Stateful Services]],
Candidate: because my gateway is the only stateful tier and the reason it is safe
Candidate: to lose one is the reason the whole architecture is simple. Study
Candidate: separately: [[rpo-rto|RPO and RTO]], so that when you say "RPO zero"
Candidate: in an interview you can immediately say what you gave up to get it,
Candidate: which is a two-of-three quorum write and five milliseconds. And study
Candidate: separately: [[gossip-protocol|Gossip Protocol]], because failure detection
Candidate: and ring membership at thirty-nine thousand nodes is not a database
Candidate: problem and pretending otherwise is how fleets split-brain.

## Act 8 - Final Summary

Interviewer: Ninety seconds. Monday.

Candidate: Clients hold a WebSocket to one of about thirty-nine thousand gateway
nodes, chosen by consistent hashing on user id with a replica set per segment.
Each node holds twenty thousand connections and about three hundred megabytes, and
runs at roughly a hundred and thirty events per second, so the fleet is sized by
connections and memory, not throughput.

Candidate: Send path: client encrypts with the double ratchet, generates a
ULID, sends ciphertext to its gateway, gateway forwards to a sequencer
colocated with that conversation's message shard, the sequencer hands out a
sequence from a batched range, the shard writes with a two-of-three quorum, we
ack the sender, and we publish MessageAccepted.

Candidate: Delivery: workers consume the event, resolve members, look up each
member's gateway, and push a pointer, never a payload. The device fetches by
after_seq and decrypts locally. Offline users are not a special case, they are a
sync that starts later.

Candidate: Ordering is a single server-assigned monotonic sequence per
conversation, assigned by a colocated sequencer using range allocation, with a
hole registry so clients learn which gaps are permanent. Deduplication is a unique
constraint on sender plus client ULID, enforced at the store, plus a client-side
seen-set. Devices and ratchet state are durable three-replica rows because losing
them breaks sessions.

Candidate: Media goes sender to object storage directly by presigned URL; the
server carries only a reference and a hash. Presence is a 60-second TTL Redis
hash refreshed by heartbeat and never on the message path.

Candidate: Failure: shard primary dies and we promote with a fencing token and
RPO zero; gateway dies and clients reconnect and resync with zero data impact;
a slow socket fills its one-megabyte queue and we disconnect and resync rather
than buffer, because the store is authoritative; and a group above 256 members
skips per-member delivery rows entirely and uses a high-water mark plus pull.

Candidate: The two complaints from support: battery and drops are addressed by a
two-tier liveness model, 25-second ping in foreground and backing-off ping plus
push in background, with the gateway treating missed heartbeats as soft death.
Delivery delay at scale is addressed by the 256-member materialization threshold,
by horizontal scale on gateway nodes, and by backpressure that sacrifices
notification latency but never message correctness.

Interviewer: That is the design I would build. Thank you.
