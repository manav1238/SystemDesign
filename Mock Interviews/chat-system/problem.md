---
title: "Design WhatsApp / Chat System"
status: active
tags: [hld, mock, chat-system]
---

# Design WhatsApp / Chat System

## Problem Statement

Design a real-time messaging system like WhatsApp or iMessage.

Users register with a phone number, can see which of their contacts are online right now, can
start one-to-one and group conversations, send text messages that arrive on the recipient's device
in under a second, and can scroll back through the full history of any conversation on any device
they are logged into.

The system must show a typing indicator while a user is composing a message, show a "last seen"
timestamp for offline users, show delivered and read receipts per message, and show blue ticks once
the other person has actually read the message in a one-to-one chat.

Messages must be delivered in order within a conversation, and nothing a user has sent may ever
disappear silently. If a user is offline when a message is sent, the message must be waiting for
them and delivered the moment they reconnect.

Scale it to roughly 500 million daily active users sending on the order of 60 billion messages per
day, including large group chats with up to 500,000 members.

## Clarifying Questions You Should Ask

- Is message delivery latency a hard requirement, and what is the target? Under 500 ms end to end,
  or one second is acceptable?
- Is ordering required only within a conversation, or globally across a user's conversations?
- If a conversation has 500,000 members, does a new message need to be materialized for every
  member, or is it acceptable to store it once and expand the membership at read time?
- Do read receipts need per-message rows, or is "last read message id per conversation" enough
  visually while still being cheap to write?
- Is the typing indicator allowed to be lossy? I would argue it must be best-effort and never
  persisted, since it has no value one second after it is stale.
- Do we need end-to-end encryption, and does that change the design, since the server can no longer
  scan message content for spam or illegal content?
- Multi-device: is each device independent, so "read" means read on at least one device, or does the
  account owner need to see per-device read state?
- Is a message ever editable or deletable, and if so does that require tombstones rather than hard
  deletes for consistency across replicas?
- What is the retention requirement, and do we need compliance exports that let a user export all
  their own messages?
- Do we optimize for a user's own sending experience, or for low battery and data usage on the
  recipient's phone, which is a very different engineering priority?

## What You Are Evaluated On

### Phase 1: Requirements

Clarify functional requirements (1:1 and group chat, presence, history, receipts, typing, last seen)
and non-functional requirements (sub-second latency, ordering, durability, high availability, huge
fanout). Confirm the strictest ones and push back on the rest.

### Phase 2: Estimate

Compute messages per second, peak QPS, delivery fanout, storage per year, and the read/write ratio.
Show the real numbers, including the cost of read receipts and the difference between average and
peak group size.

### Phase 3: High-level Design

Draw the architecture: WebSocket gateway tier, stateless chat service, message store, presence
service, fanout layer, and the [[message-queue|queue]] that decouples them. Explain the path a
message takes from sender to receiver.

### Phase 4: Deep dive

Go deep on message ordering per conversation, sharding by conversation id, read receipts and
last-seen, ephemeral versus persisted state, presence without hammering storage, backpressure when
a client is on a slow link, and cache stampede on a hot conversation.

### Phase 5: Trade-offs and follow-ups

Defend the choices: why [[websockets|WebSocket]] over long polling, why fanout at read time versus
write time, why not put presence in [[redis]] forever, how you survive a primary database failure
and replica lag, what happens when one conversation has 500,000 members, and what you would change
if traffic doubled overnight.

Study separately: Push Notifications (APNs/FCM gateways), E2E encryption, and mobile offline sync
conflict resolution.
