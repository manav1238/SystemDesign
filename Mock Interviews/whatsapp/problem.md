---
title: WhatsApp - Real-Time Messaging Platform
status: active
tags: [hld, mock, whatsapp]
---

# WhatsApp - Real-Time Messaging Platform

## Problem Statement

Design a real-time messaging platform in the style of WhatsApp. Users register
with a phone number, exchange messages in one-to-one conversations and group
conversations, see when a contact is online, send photos and voice notes, and
receive messages on multiple devices at the same time.

You are a senior backend engineer on the messaging infrastructure team. You have
one 45-minute system design interview to present a scalable design.

> **The brief**
>
> The product must feel instant. A user sends a message and the recipient sees
> it appear within a second while they are online. If the recipient is offline,
> the message waits and is delivered the moment they reconnect, with no loss and
> no visible duplicates. A user may have a phone, a tablet and a laptop
> connected to the same account simultaneously, and each device must see every
> message, including messages the user sends from another device.
>
> Messages are end-to-end encrypted. The server routes ciphertext and never
> holds a key that can decrypt a message body. Group conversations support up to
> a thousand members.
>
> There are two explicit complaints from the support team: (1) battery drain and
> dropped connections on mobile when the app holds a long-lived connection
> open, and (2) delivery delays during large events, when everyone is sending at
> once.

### Explicitly In Scope

- Registration and authentication by phone number
- One-to-one conversations and group conversations up to about 1000 members
- Text message send, receive, and per-conversation history
- End-to-end encryption with multi-device support
- Online presence and last-seen
- Media messages, primarily images and voice notes
- Read receipts and delivered receipts
- Message delivery to all of a user's devices
- Multi-device linking and unlinking

### Explicitly Out of Scope

- Voice and video calling, which is a separate real-time media system
- Channels and broadcast lists to very large audiences
- Payments
- Status updates, which are closer to the Stories design in
  [[../instagram/interview|Instagram]]

### Scale Assumptions Given to You

- 2 billion registered users, 900 million daily actives
- 60 billion messages per day
- 80 percent one-to-one, 20 percent group
- Average group size 4, with a long tail up to 1000
- Average text message 1 KB; 10 percent of messages carry media averaging
  200 KB
- Peak to average ratio of 2.5, and a 2x burst on top of peak for a global event

## Clarifying Questions You Should Ask

A strong candidate spends the first five minutes here. Messaging has several
requirements that look like implementation details and are actually the whole
design, and the ones you fail to ask about are the ones that will be used
against you.

**About correctness**

- "Is exactly-once delivery a hard requirement, and if not, what is the
  observable contract you want? I am going to argue for at-least-once delivery
  with client-side deduplication by message id, because exactly-once end to end
  is not achievable across a network, a broker, a database and a handset."
- "How many duplicates and reorders is the user allowed to observe, even
  briefly? If the answer is zero, I need a global order per conversation and
  that has a real cost."
- "Does message order matter within a conversation, or only between
  conversations? I assume total order within a conversation and no order across
  conversations, and I want to confirm that."
- "If the sender is offline and a message is sent from their other device, does
  the same conversation ordering apply? And if a user joins a group while
  offline, what do they see?"

**About encryption and multi-device**

- "Is the threat model a curious server operator, or an attacker who
  compromises the server? That decides whether the server may see metadata, and
  whether signal ratchet state can be a correctness dependency or just a cache."
- "How is a new device linked, and what does the user see on the old device? Is
  a linked device able to read history sent before it was linked?"
- "If a user unlinks a device, what happens to messages that device already
  holds? This is the hardest question in the product and I want to know the
  product answer before I design the technical one."
- "Since the server cannot read messages, server-side content moderation and
  search are impossible. Is that accepted, and what is the substitute?"

**About presence and receipts**

- "Is presence exact or approximate? I am going to assume a best-effort signal
  with a 30 to 60 second TTL, because exact online status costs a write per
  state change on a hot key and buys very little."
- "Do read receipts need to be per-message or per-conversation, and does a
  receipt from one device mark the conversation read for all devices?"

**About media and scale**

- "Does media travel through our servers or directly from sender to recipient?
  I strongly believe directly, and I want to confirm the product accepts that a
  recipient can see a failed transfer."
- "What is the retention policy? Keeping every message forever is a different
  business from keeping the last 200 per conversation and archiving the rest."
- "What is the acceptable delivery latency for an online recipient, and does that
  have to be under 500 milliseconds end to end including the recipient's client
  render?"
- "What is the battery complaint actually about, heartbeat frequency or
  connection lifetime? The fixes are very different."
- "Is this single region or multi-region, and do users on one continent talk to
  users on another as a normal case?"

## What You Are Evaluated On

### Phase 1: Requirements and Constraints

The bar here is that you identify delivery semantics, per-conversation ordering,
multi-device fanout, and the encryption constraint as *design inputs that
invalidate the obvious design*, not as features to implement later. The obvious
design, a WebSocket gateway plus a database, is wrong in three specific ways and
you must find them without being told.

Look for: an explicit statement that exactly-once end to end is not achievable
and what the achievable contract is instead, a decision on what "ordering" means
and what it does not, recognition that end-to-end encryption rules out
server-side search and content moderation, and a clear position on presence being
eventual and cheap.

### Phase 2: Scale Estimation

Real numbers, arithmetic shown, and the two estimates that actually drive the
architecture: concurrent persistent connections, and message write throughput
including group fanout. Most candidates estimate the read and write counts and
then miss that the binding constraint is connection count, which is what
determines the size of the gateway fleet.

Look for: messages per second at average and peak, deliveries per second after
accounting for devices and group size, concurrent online connections, gateway
fleet size, bytes per day of message storage, media storage, and the working
set after applying a hot-retention window. The insight to reach is that each
gateway handles only tens of messages per second but tens of thousands of
connections, so the fleet is sized by memory, not by throughput.

### Phase 3: API and Data Design

Conversations, messages, presence, receipts, media, and device management. The
message schema is the one to get right: it needs a per-conversation sequence
number assigned by the server, a client-generated message id for deduplication,
and a per-device delivery state that is separate from the message itself.

Look for: the ordering token, the idempotency key, per-device delivery rows
rather than a boolean, a group membership table sharded by group id, and an
explicit decision on whether the gateway is stateful.

### Phase 4: Architecture and Data Flow

A [[websockets|WebSocket]] gateway tier with connection pooling, a session
directory that routes a user to a gateway, a message sequencer, a durable
message store, an offline message delivery path, and an event backbone. Trace
the send path and the receive path separately, hop by hop, and be explicit about
where the ciphertext is created and where it is decrypted.

Look for: media bypassing the gateway entirely, the gateway holding no durable
message state, the sequencer being the only ordering authority, and the
delivery path re-reading from the message store for offline users rather than
holding messages in gateway memory.

### Phase 5: Deep Dive, Trade-offs, and Failure

Gateway failure and reconnection, replica lag on the message store, a group with
a thousand members, backpressure when a recipient's socket is slow, duplicate
delivery, key-state loss, and the durability trade-off between acknowledging
fast and acknowledging safely.

Look for: at-least-once plus idempotent clients as the achievable contract,
fencing on failover, a message store where the primary writes are acknowledged
on a quorum, backpressure implemented as disconnect-and-resync rather than
unbounded buffering, and a stated RPO and RTO.
