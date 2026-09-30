---
title: "Design a Notification System"
status: active
tags: [hld, mock, notification-system]
---

# Design a Notification System

## Problem Statement

Design a notification system for a large consumer platform. Users receive notifications over
multiple channels: mobile push (Android and iOS), email, and SMS.

The platform's services emit events such as "order shipped", "someone commented on your post",
"a video you follow was published", or "your account needs verification". A notification service
consumes these events, decides whether each user should be notified, renders the message content
from a template, and delivers it through the correct channel to the correct device or address.

Users must be able to configure per-category preferences: which categories they want, which
channels for each category, and quiet hours during which nothing should be delivered immediately.
Categories that are low-urgency should optionally be rolled up into an hourly or daily digest
instead of being sent one by one.

The system must never send the same notification twice, must survive provider outages without
losing notifications, must respect the rate limits that push, email, and SMS providers impose, and
must track and report delivery status so that the sending services can see whether a notification
was delivered, bounced, or opened.

Scale it to 50 million daily active users receiving on the order of 750 million notifications per
day, and design it so a single marketing campaign to ten million followers does not degrade
day-to-day transactional notifications.

## Clarifying Questions You Should Ask

- What is the latency requirement per notification type? Is "comment on your post" allowed to
  arrive in one minute while "order shipped" must arrive in two seconds?
- Which channels are transactional and which are marketing? They have very different permission
  models and very different costs per message.
- Is a duplicate notification a product bug or merely an annoyance? For SMS it costs real money and
  is legally sensitive, so I would like to know the tolerance for duplicates.
- When a user is in quiet hours, do we drop the notification, queue it until the window opens, or
  fold it into the digest? Those are three different products.
- Do we send a notification if the user already has an app open and is looking at the screen?
  Most platforms suppress those, and that is a large fraction of traffic.
- How long do we keep delivery records, and does the compliance story require us to be able to
  prove what was sent to a given user at a given time?
- Do we have per-tenant or per-team notification quotas, or is the quota only against the external
  provider?
- If a user has no valid device token or the bounce rate on their address is high, do we
  auto-disable that channel for them?
- Does a digest need to be re-generated if new items arrive after it was sent, or is it a fixed
  batch of what existed at send time?
- What happens on a partial failure, where 40 percent of a campaign's SMS messages succeed? Do we
  retry all, only the failed, or stop?

## What You Are Evaluated On

### Phase 1: Requirements

Separate transactional from marketing traffic, establish per-category latency and channel
requirements, and pin down what counts as a duplicate, what quiet hours mean, and what happens when
the user is currently looking at the app.

### Phase 2: Estimate

Estimate events per second, per-channel send volume, notification log volume and retention cost,
and then identify that the real ceiling is the external provider's rate limit rather than your own
compute. Show the arithmetic for each.

### Phase 3: High-level design

Draw the pipeline: event producers, the event bus, the notification orchestrator with preference
evaluation and template rendering, per-channel sender services, a provider gateway layer, and the
delivery-status feedback path.

### Phase 4: Deep dive

Go deep on the template builder, fanout at scale, preference and quiet-hours evaluation, dedupe
keys, immediate versus digest batching, the retry ladder with backoff and dead-lettering, provider
failure isolation, per-provider rate limiting, and delivery tracking.

### Phase 5: Trade-offs and follow-ups

Defend the choices: why not call the provider from the request thread, one queue versus per-channel
queues, why not cron everything, how you survive a primary database failure and replica lag, what
happens on a cache stampede of templates, and what changes if traffic doubles.

Study separately: Push Notifications (APNs/FCM gateways), third-party email/SMS provider quotas and
failover, and per-tenant noisy-neighbor isolation.
