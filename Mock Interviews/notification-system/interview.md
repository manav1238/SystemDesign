---
title: "Notification System — Interview Transcript"
status: active
tags: [hld, mock, notification-system, transcript]
---

# Notification System — Interview Transcript

Target duration: 45 minutes. The lesson in this transcript is that a notification system's design
is dominated by its *external* constraints: provider quotas, cost per message, and duplicate
sensitivity. Arithmetic first, architecture second.

---

## Phase 1: Requirements Clarification

**Interviewer:** Today's problem is a notification system. Multi-channel push, email, and SMS,
driven by events from other services, at the scale of 50 million daily active users and about 750
million notifications per day. Before you design, what do you need to ask me?

**Candidate:** Let me start with the split that changes everything, which is transactional versus
marketing.

**Interviewer:** Go on.

**Candidate:** "Order shipped" and "someone commented on your post" are transactional. A product
launch announcement is marketing. Are these the same system?

**Interviewer:** Same system, different constraints. Transactional must be fast and reliable.
Marketing is high volume, best-effort, and has a budget.

**Candidate:** Then I would still build them on the same pipeline, because rebuilding the template
and preference engine twice is a real cost, but I would partition and prioritize them at the queue
layer, and I would give them separate rate-limit buckets against the provider. A marketing campaign
must never be able to exhaust the SMS quota that transactional 2FA depends on. That is the central
threat model for this system, and it is a resource-contention problem, not a correctness problem.

**Interviewer:** Good instinct. Keep going.

**Candidate:** Next: latency per category. Is a comment notification allowed to arrive in a minute,
while order shipped must be in two seconds?

**Interviewer:** Order shipped, security alerts, and payment failures: 5 seconds. Everything else:
under 5 minutes is fine.

**Candidate:** That is a really useful answer, because it licenses two completely different
mechanisms. Sub-5-second notifications must go through the streaming path: event bus, consume,
render, hand to provider, all asynchronous and low-latency. The under-5-minutes category can go
through a batching path, because I am going to batch anyway for efficiency. And the digest category
is a third path with a scheduled trigger. One system, three trigger strategies, and I will name them
explicitly so nobody later argues about which one a given event should use.

**Interviewer:** When a user is in quiet hours, drop, queue, or digest?

**Candidate:** All three, depending on urgency. Quiet hours apply only to the low-urgency
categories; a security alert bypasses quiet hours by design, because the entire point of a
security alert is that it arrives at 3am. For a comment during quiet hours, hold it and release at
window open. For a low-urgency category, fold it into the digest. So quiet hours is not a boolean
on the notification, it is a routing decision that depends on urgency.

**Interviewer:** And if the user has the app open right now?

**Candidate:** Suppress, or better, mark it read. This is a large fraction of traffic. If a user is
actively on the comment screen when someone comments, sending a push is pure annoyance. So the
orchestrator checks active session presence before enqueueing a push. Note the parallel to the chat
system: this is an ephemeral presence read, it must be a fast cache lookup, and it is allowed to be
several seconds stale because being wrong just means one redundant push.

**Interviewer:** Is a duplicate notification a bug or an annoyance?

**Candidate:** It depends on the channel, and I want to make that explicit because it changes the
design. For push, a duplicate is an annoyance. For email, an annoyance. For SMS, it is a real cost
of about half a cent and, more importantly, it is a regulatory and trust problem, because repeated
marketing SMS drives opt-out and complaints. So I hold a hard exactly-one-effective-send guarantee
for SMS and email, and best-effort for push.

**Interviewer:** How do you classify a category's urgency?

**Candidate:** I would make it explicit data rather than code, attached to the event type, so that
adding a new event type is a config change, not a deploy. And the default for a newly added,
unclassified event type should be the conservative one, which is low-urgency plus digest, because
I do not know what I do not know.

**Interviewer:** Anything about retention I should know?

**Candidate:** I need to be able to answer "what notifications did we send to this user on this
day, and what happened to them", for user support, for abuse investigation, and for the
deliverability reputation that email and SMS providers score you on. So I need a per-notification
lifecycle record, retained long enough to be useful, which for SMS and email means at least a year
because complaint rates are scored over rolling windows. But the volume is high, so I will tier it
rather than store everything hot forever.

**Interviewer:** Good. Design it.

---

## Phase 2: Scale Estimation

**Interviewer:** Numbers.

**Candidate:** 50 million DAU, 750 million notifications per day.

**Interviewer:** Per second?

**Candidate:** 750 million over 86,400 seconds is about 8,680 per second on average. But
notifications are far spikier than chat, because they are driven by external events and by business
cycles rather than by human typing. A product drop, a flash sale, or a sports match can produce a
20x spike for twenty minutes. So average is 8,700 per second and I plan for a peak of roughly
150,000 per second, which is about 17x average. I would rather be pessimistic on peak here than on
chat, because the spikes are externally triggered and I cannot smooth them.

**Interviewer:** That is the decision rate. What is the send rate?

**Candidate:** Each notification decision fans out to channels. Not every user has every channel
enabled, so let me assume an average of 1.4 channels per notification after preference filtering,
because many users disable email, and SMS is rare. So peak sends are 150,000 times 1.4, about
210,000 sends per second. And that splits roughly 55 percent push, 35 percent email, 10 percent SMS
by volume, which gives me approximately 115,000 push per second, 74,000 email per second, and
21,000 SMS per second at peak.

**Interviewer:** Now the interesting question. Can you actually do that?

**Candidate:** Here is where the arithmetic changes the design, and this is the point I most want a
candidate to reach. My own compute is nowhere near the problem. 210,000 sends per second is
ordinary for a fleet of a few hundred machines. The constraint is the providers.

**Candidate:**
```text
Provider          Peak capacity                    My peak demand    Verdict
FCM (Android)     ~1,000,000 msg/sec (project)     115,000/s         11% used, fine
APNs (iOS)        ~50,000-200,000/sec (pooled)      80,000/s          tight, throttled per token
SendGrid-like     ~1,000-10,000 email/sec/account   74,000/s          need multiple subaccounts
SMS provider      ~50-100 msg/sec per long code    21,000/s          200x OVER capacity
```

**Candidate:** So the SMS channel is over capacity by a factor of roughly 200, and no amount of
horizontal scaling on my side changes that, because the limit is the provider's. This means three
things architecturally. First, SMS must be admission-controlled by a global token bucket, not sent
eagerly. Second, I must have a fallback path, so when SMS capacity is exhausted a high-value
transactional SMS degrades to push rather than being dropped. Third, the honest answer to the
capacity question is that SMS throughput is a product decision about cost, not a scaling decision,
and I should raise that with the business before building anything.

**Interviewer:** Storage.

**Candidate:** The notification log is the big one. 750 million a day. If a row is roughly 500
bytes with user id, template id, rendered subject, channel, provider, status, timestamps, and
metadata, that is 375 gigabytes a day, so about 137 terabytes a year. In a relational store with
indexes that realistically becomes 250 to 300 terabytes a year. I will not keep that hot.

**Candidate:**
```text
Hot tier   30 days    11 TB    full fidelity, queryable
Warm tier  12 months  137 TB   columnar / data warehouse, compressed ~5x
Cold       beyond      object storage, query via Athena/BigQuery

TEMPLATES:   ~20,000 templates x 10 KB = 200 MB   trivial
PREFERENCES: 50M users x ~300 bytes  = 15 GB      trivial, redis + db
DEDUPE KEYS: 750M/day x 40 bytes    = 30 GB/day   ttl 24h -> 30 GB live
```

**Candidate:** The asymmetry here is instructive. Templates and preferences, which are the parts
everybody wants to over-engineer, are 200 megabytes and 15 gigabytes. The log, which nobody thinks
about, is 137 terabytes. Design effort should be allocated by bytes and by risk, not by how
interesting the component sounds.

**Interviewer:** Read to write ratio, and where is the read traffic?

**Candidate:** Very read-skewed, and it is almost entirely two queries. One, the notification inbox
or history for a user, which is a paged list of their recent notifications. Two, and this is the
one that actually costs me, rendering the template, because that happens on every single send and
touches the template store 210,000 times a second.

**Interviewer:** So your template path is your hottest read.

**Candidate:** Correct, and it is cached aggressively, which I will cover in the deep dive. The
other read I care about is preferences, evaluated once per notification before channel selection,
also 150,000 times a second at peak. Both are tiny datasets with huge read multipliers, which is
the textbook caching case.

---

## Phase 3: High-Level Architecture

**Interviewer:** Draw the system.

**Candidate:** Four stages, connected by two asynchronous boundaries. Stage one is event
production, stage two is the orchestrator that decides and renders, stage three is per-channel
senders, stage four is the provider gateway layer, plus a feedback path for delivery status.

**Candidate:**
```mermaid
flowchart TD
    OS[Order svc]
    SO[SOCIAL svc]
    AU[AUTH svc]
    EB[EVENT BUS Kafka<br/>partitioned by tenant<br/>durable, replayable<br/>ordered within partition, fan-out]
    RES[Recipient resolver<br/>fan-out]
    PRE[Preference +<br/>urgency evaluator]
    DED[Dedupe + quiet hours<br/>suppress?]
    TPL[Template builder + cache]
    BAT[Batching / digest scheduler]
    QP[QUEUE PER CHANNEL<br/>push | email | sms<br/>+ priority class]
    DQ[DIGEST QUEUE<br/>scheduled]
    PS[Push Sender]
    ES[Email Sender]
    SS[SMS Sender]
    DB[Digest Builder]
    PG[STAGE 4: PROVIDER GATEWAY<br/>token bucket per provider<br/>circuit breaker per provider<br/>provider failover + channel fallback<br/>idempotency key on every outbound call]
    APN[APNs]
    FCM[FCM]
    ESP[Email ESP]
    SMSG[SMS Gateway]
    PH[Provider webhooks / postbacks]
    DL[Delivery log / status store]

    OS -->|domain event| EB
    SO -->|domain event| EB
    AU -->|domain event| EB
    EB --> RES
    RES --> PRE
    PRE --> DED
    PRE --> TPL
    DED --> BAT
    TPL --> QP
    BAT --> DQ
    QP --> PS
    QP --> ES
    QP --> SS
    DQ --> DB
    PS --> PG
    ES --> PG
    SS --> PG
    DB --> PG
    PG --> APN
    PG --> FCM
    PG --> ESP
    PG --> SMSG
    APN --> PH
    FCM --> PH
    ESP --> PH
    SMSG --> PH
    PH -->|feedback / status| DL
```

**Candidate:** Three things in that diagram carry the design. First, the two asynchronous
boundaries. Orchestrator to sender is a queue, and sender to provider is another boundary inside the
gateway. Second, per-channel queues, which is what gives me priority isolation and lets a broken
SMS provider back up only the SMS queue. Third, the feedback path, which most candidates forget:
delivery status flows *backwards* from the provider and has to be written somewhere queryable by
the original sending service.

**Interviewer:** Why per-channel queues rather than one queue?

**Candidate:** Four reasons, in order of how much they matter. Priority isolation: a burst of
marketing push must not delay a 2FA SMS, and one queue means head-of-line blocking by construction.
Routing: the SMS sender needs a different consumer group, different scaling, and a different rate
limit than the push sender, which is impossible to express with one shared queue. Failure isolation:
if the SMS provider is returning errors and I am filling the SMS retry backlog, I want exactly one
queue to back up, not the one that also carries email. And independent autoscaling: push volume
tracks user activity while email volume tracks campaign schedules, and one queue forces one scaling
signal. One queue would be simpler and it would be wrong in four separate ways.

**Interviewer:** Why a queue between the orchestrator and the senders, rather than calling the
provider directly?

**Candidate:** This is the most important decoupling decision in the system and I would defend it
hardest. If the orchestrator calls the provider synchronously, then a provider that takes three
seconds to respond, or is rate-limiting me, or is simply down, propagates directly into the domain
service that produced the event. "order shipped" would fail, and the user would see a failed order
because a push provider was slow. That is an unacceptable coupling, and it is why notifications are
always asynchronous from the producer's point of view. The producer's job ends at "the event is
durably recorded", and I do not owe the user a delivery guarantee from the order service. The queue
also gives me retry and dead-lettering, which the direct call does not, and it absorbs the 17x peak
spike. See [[loose-coupling]] and [[asynchronous-processing]].

**Interviewer:** Walk me through a single notification end to end.

**Candidate:** A user gets a comment. The social service commits the comment in its own database
and, in the same transaction, writes an outbox row. An outbox relay publishes
`comment.created` with the comment id, the author id, and the target post id, to Kafka, keyed by
target post id so that ordering of comment events for one post is preserved. The notification
orchestrator consumes it, and step one is recipient resolution: fetch the post owner, the previous
commenters, and the mentioned users, and apply the fan-in rules such as the previous commenter, or
the parent comment author, and apply suppression rules for self-comments and blocked users. Step two
is per-recipient preference and urgency evaluation. Step three is dedupe. Step four is quiet-hours
routing. Step five is template rendering with the locale and variables. Step six is one enqueue per
enabled channel. Then the sender consumes, the gateway applies the rate limit and the circuit
breaker, calls the provider with an idempotency key, and writes the initial status. Later the
provider's webhook arrives and I update the terminal status.

**Interviewer:** You said outbox. Why not just publish to Kafka in the request?

**Candidate:** Because a dual write, database plus broker, is not atomic. If I commit the database
transaction and then the Kafka publish fails, the notification is lost silently and permanently. If
I publish first and the database transaction then rolls back, I have notified users about something
that did not happen. The [[outbox-pattern]] makes both writes part of one local transaction, and a
relay process publishes to the broker afterward. It costs me eventual consistency of roughly a
second and a relay process, and it converts silent permanent loss into possible duplicate delivery,
which I can then fix with dedupe. That trade is obviously correct, and in general I would rather
have at-least-once and dedupe than at-most-once and silence. See [[event-driven-architecture]] and
[[idempotent-consumer]].

---

## Phase 4: Deep Dive

### Templated notification builders

**Interviewer:** Templates. Design that.

**Candidate:** A template is a versioned, per-locale, per-channel, per-category artifact with
placeholders. I would store them in a relational store because they need review workflow, and I
would treat them as immutable and versioned rather than edited in place.

**Candidate:**
```sql
CREATE TABLE templates (
  template_id     VARCHAR(64)   NOT NULL,
  version         INT           NOT NULL,
  category        VARCHAR(64)   NOT NULL,
  channel         VARCHAR(16)   NOT NULL,
  locale          VARCHAR(16)   NOT NULL,
  subject_template TEXT,
  body_template   TEXT          NOT NULL,
  variables       JSON          NOT NULL,   -- declared contract
  status          VARCHAR(16)   NOT NULL,   -- draft | active | retired
  checksum        VARCHAR(64)   NOT NULL,
  created_at      BIGINT        NOT NULL,
  PRIMARY KEY (template_id, version)
);

CREATE UNIQUE INDEX idx_templates_active
  ON templates (category, channel, locale, version)
  WHERE status = 'active';
```

**Candidate:** Four decisions in there. Versioned, so a render in flight always completes against
the version it started with, and so I can roll back a bad copy change. `variables` as a declared
JSON contract, so a template cannot reference a field the code does not supply, which turns a
runtime blank or crash into a build or publish-time validation error. `checksum`, so that identical
renders can be deduped and cached, and so I can detect accidental mutation of a live template. And
a partial unique index for exactly one active version per category, channel, and locale, enforced by
the database rather than by a code path someone will forget.

**Candidate:** The render path:

**Candidate:**
```text
render(template_ref, variables, locale, channel):
  tpl = templateCache.get(category, channel, locale)
  if miss -> templateStore.loadActiveVersion()
  vars  = validate(tpl.variables, variables)
  body  = tpl.body_template | render(vars)
  return Rendered( subject, body, tpl.version, tpl.checksum )
```

**Interviewer:** You are rendering 210,000 times a second. What is hot?

**Candidate:** The template store lookup, and I solve it with a whole-template cache in
[[redis]] plus a local in-process copy. The full working set is 20,000 templates at 10 KB, which is
200 MB, so it fits in the memory of every orchestrator node at once. I load the entire active set at
startup, keep it in a local immutable map, and have a version-poller or a pub-sub invalidation
channel push updates. So the hot path is an in-memory map lookup, not a network hop, and the
authoritative store is only consulted on a cold start.

**Interviewer:** Template cache stampede. All orchestrator nodes restart at once, or you publish a
template change to 500 nodes at once. What happens?

**Candidate:** Three specific defenses, because a stampede here is 210,000 render threads all trying
to read the same 20,000 rows. One, warm the cache on deploy and stagger rollouts, so a full-cluster
restart never becomes a simultaneous cold start. Two, single-flight coalescing per template key, so
500 nodes missing the same template produce one store read. Three, and this is the one I care most
about, serve the last-known-good rendered content from a local snapshot rather than blocking on the
store. A slightly stale subject line is a far better outcome than a failed notification, because
the fallback is not "return an error", it is "send a generic fallback template". Degradation here
should be invisible, not an error. See [[cache-warming]] and [[graceful-degradation]].

**Interviewer:** Logic inside templates?

**Candidate:** No arbitrary logic, deliberately. Mustache-style substitution and a small fixed
function allowlist for things like pluralization and date formatting. The reason is a failure mode I
have seen repeatedly: a template becomes a program, the template now has its own bugs, its own
performance profile, and its own injection surface, and a 10 KB template that used to be a database
row is now something that can exhaust a thread pool. Templates stay data.

### Fanout from events

**Interviewer:** Ten million followers get notified. Walk me through it.

**Candidate:** This is the fanout problem again, but it is genuinely different from the chat
version, and I want to explain why. In chat, a user eventually *pulls* the message, so read-time
fanout works because there is a read to defer the work to. A notification has no pull. If I do not
create the per-user record, the user is never notified. So notification fanout must be resolved at
write time, or resolved at send time with the per-user existence of the notification being
implicit.

**Candidate:**
```text
Option A: FULL EAGER FANOUT
  insert 10,000,000 rows into notification_log   (user_id, campaign_id, ...)
  then a publisher streams them into the channel queues
  writes: 10M immediately
  reads : trivial per-user lookup
  cost  : 10M x 500B = 5 GB, and ~116k rows/sec for ~90 sec

Option B: SHARED NOTIFICATION + LAZY MATERIALIZATION
  write 1 row: {campaign_id, channel, rendered_content_id}
  push a campaign notification by PUBLISH to a "campaign:followers:{id}" topic
  sender service, per recipient, checks the campaign row + dedupe,
     and only writes notification_log if the user has not already been sent it
  writes: 1 row
  cost  : the dedupe key becomes the per-user work unit
```

**Candidate:** I use B, and the reason is that Option A's 116,000 rows per second lands on the
same primary that is serving every other notification in the system, so a marketing campaign
becomes a self-inflicted denial of service on transactional notifications. With B, the campaign's
116,000 units per second flow through a dedicated topic that the sender consumes at whatever rate
the provider quota permits, and it simply takes longer. A campaign that finishes in five minutes
instead of ninety seconds is not a product problem, but a transactional notification delayed by
ninety seconds is.

**Candidate:** The catch with B is that you can no longer answer "did user X get this campaign"
without either materializing or keeping a per-user dedupe record. So I keep a per-user
notification-state record, which is the same cost as the log row, and the honest answer is that
some per-user work is unavoidable. The gain from B is that the writes are *ordered, throttled, and
paced by the provider quota* instead of being an instantaneous 5 GB burst against the primary.

**Interviewer:** What about a viral post with a million comments in an hour?

**Candidate:** Two mitigations. First, per-entity rate limiting at the orchestrator: no single
entity, meaning one post or one video, generates more than a configured number of notifications per
minute, and beyond that the excess is aggregated into a single "1,000 people commented on your
post" summary notification. Aggregation is better than rate limiting for this case, because it
conveys the same information at a fraction of the volume. Second, campaign-level and per-user
cooldown windows, so one user is not notified 200 times in an hour about one viral thread.

### Preferences, urgency, quiet hours

**Interviewer:** Design the preference evaluation.

**Candidate:**
```sql
CREATE TABLE user_preferences (
  user_id     VARCHAR(64)  NOT NULL,
  category    VARCHAR(64)  NOT NULL,
  push_enabled  BOOLEAN     NOT NULL DEFAULT TRUE,
  email_enabled BOOLEAN     NOT NULL DEFAULT TRUE,
  sms_enabled   BOOLEAN     NOT NULL DEFAULT FALSE,
  quiet_hours_start SMALLINT,          -- local minutes, NULL = never
  quiet_hours_end   SMALLINT,
  timezone     VARCHAR(64)  NOT NULL,
  digest_mode  VARCHAR(16)  NOT NULL DEFAULT 'immediate',  -- immediate | hourly | daily
  updated_at   BIGINT       NOT NULL,
  PRIMARY KEY (user_id, category)
);
```

**Candidate:** Three things I got right by writing them down. First, preferences are per category
per channel, not one global on/off, because "I want order updates but not comments" is the actual
user desire and a global switch cannot express it. Second, quiet hours are stored as local wall-clock
minutes plus a timezone, not as UTC instants, because "do not disturb between 10pm and 7am" means
10pm *where the user is*, and if the user travels, their quiet hours travel with them. Third,
`digest_mode` is stored on the preference row, not on the category, because the same category
should be immediate for one user and daily for another.

**Interviewer:** Evaluation cost, 150,000 times a second.

**Candidate:** Two-level read. The entire preference dataset is 15 gigabytes and after
denormalization into a per-user blob it is maybe 4 gigabytes of hot working set, so I keep a
per-user compiled preference object in [[redis]] with a 10 minute TTL, and a local LRU on each
orchestrator node for the hottest users. The evaluation itself is then pure in-memory logic: check
the category, intersect with enabled channels, check urgency against quiet hours, choose immediate
or digest. Note the TTL is longer than the notification's own staleness budget, because a stale
preference means at most one unwanted notification, which is a far better failure than a
missed-order notification caused by a cache miss. Consistency here should be tuned toward
over-notifying rather than under-notifying for high-urgency categories.

**Interviewer:** Quiet hours and timezones. Daylight saving, a user in two zones, the 2am edge?

**Candidate:** Store the IANA timezone name and compute the current local time per evaluation,
never store a fixed UTC offset, because a fixed offset is wrong twice a year and wrong for half the
year. For the edge case, I wrap the window comparison as a function that returns three states:
inside the window, outside the window, and unconfigured, and the decision logic treats "the local
time is within a few minutes of the boundary" as outside, to avoid a notification at exactly the
quiet-hours start. A one-minute early push is worse than a one-minute late push.

**Interviewer:** Precedence when a category is disabled but the event is a security alert?

**Candidate:** Policy order, in writing, because this is a question that otherwise gets answered by
whoever wrote the code most recently. Urgency overrides user preference for a small allowlist of
categories, and that allowlist is config, not code. But the override is bounded: it overrides
*channel* and *quiet hours*, and I will still honor a channel the user has explicitly disabled
completely, because silently re-enabling SMS for a user who deleted the app is a trust violation. So
the precedence is: hard disable always wins, then urgency override beats quiet hours, then
quiet hours, then digest grouping.

### Dedupe

**Interviewer:** How do you guarantee no duplicates?

**Candidate:** The dedupe key is the interesting design decision, and it has to be a *content*
identity, not a request identity, because the same logical notification arrives from multiple
sources: the outbox relay retries, the Kafka consumer rebalances, a campaign is re-published, a
user double-taps. So the key is derived from the meaning of the notification.

**Candidate:**
```text
dedupe_key = H(
  user_id,
  category,
  channel,
  entity_id,          -- the order id, the comment id, the campaign id
  kind,                -- "new" | "reminder" | "digest"
  time_window          -- digest bucket or reminder slot, else 0
)

at enqueue:
  SET notification:dedupe:{key} 1 NX EX 86400
  result == OK  -> enqueue
  result == nil -> drop, increment dedupe_hit counter
```

**Candidate:** Three properties I want to defend. The `time_window` slot means a legitimate
re-notification after an hour still sends, so this is not a permanent block, it is a 24-hour
identity window. The `kind` field means a "reminder" of the same order is not suppressed by the
original notification, which would be a bug I have seen. And the dedupe check is a `SET NX` at
*enqueue* time rather than at send time, so the duplicate is dropped before it ever becomes a
provider call, which is the only place where a duplicate actually costs money.

**Interviewer:** Isn't `SET NX` with a 24 hour TTL on 750 million keys a day a problem? 30 GB
and heavy write amplification on Redis.

**Candidate:** Yes, and I would shard it and be honest about the eviction risk. First, only
transactional SMS and email need hard dedupe; push can use a much shorter window, maybe 5 minutes,
which cuts the live key count by two orders of magnitude. Second, I hash-partition the dedupe keys
across many Redis shards so the write is spread, rather than hot on one. Third, and this is the
honest answer, a Redis flush loses dedupe state and produces duplicates, so Redis dedupe is a
strong optimization and not a correctness guarantee. The real guarantee is the same one I used in
the chat system: the provider call itself carries an idempotency key, so even a fully duplicated
send is collapsed by the provider. Layered, and the strongest layer is the one closest to the side
effect. See [[idempotency]] and [[request-deduplication]].

**Interviewer:** And if Redis is down?

**Candidate:** Fail open for push, fail closed for SMS. Fail open means a duplicate push, which is
cheap and recoverable. Fail closed means if I cannot verify dedupe I do not send an SMS, because an
unverified SMS might be a duplicate charge and a spam complaint. Asymmetric failure modes deserve
asymmetric fallback, and stating that out loud is the actual skill.

### Immediate versus digest

**Interviewer:** Digest versus immediate. When do you batch, and how?

**Candidate:** The decision is driven by three factors, in order: does the user care about the
latency, does batching actually save meaningful cost, and does the content need to be aggregated to
make sense.

**Candidate:**
```text
NEVER DIGEST  : order status, security alert, payment failure   (perceived latency matters)
ALWAYS DIGEST : "12 people liked your photo", low-urgency social
ADAPTIVE      : comments, follows  -> digest if volume per user
                per hour exceeds a threshold, immediate otherwise
                ("adaptive" is the real answer for most social categories)

WINNER'S CHOICE  : use the preference's digest_mode
```

**Candidate:** The "12 people liked your photo" case is the one worth arguing for, because it shows
that the right answer is not a batching window, it is *aggregation*. Digesting a hundred individual
likes into one summary notification is strictly better for the user, strictly cheaper, and less
annoying. That is not a delivery optimization, it is a product design that happens to also be
cheap. If I can express the category as a summary, I should, and the batching queue becomes
unnecessary.

**Interviewer:** Implement the digest.

**Candidate:** A scheduled aggregator plus a materialized send.

**Candidate:**
```text
bucket = floor(epoch_minutes / 60) for hourly, / 1440 for daily

every 10 min, for each active bucket:
  collect unsent notification_items for users with digest_mode matching
  group by (user, channel)
  render ONE digest template with an item list (cap 20 items + "and N more")
  SETNX dedupe(user, 'digest', bucket)  -> send once
  mark items as included
  write the digest as ONE notification_log row referencing the item ids
```

**Candidate:** Three important properties. The digest is one notification with an idempotency key
keyed on the bucket, so a retried aggregator run cannot send it twice. The item cap prevents a
digest from becoming a 400-line email. And the digest references item ids rather than copying
content, so the log does not duplicate the payload bytes, which matters because I already calculated
that the log is 137 terabytes a year.

**Interviewer:** A digest that goes out at 9am, and then new items arrive at 9:05. What happens?

**Candidate:** They go into the next bucket. I would not re-send, because a digest that re-sends
five minutes later is the definition of spam, and the user is very likely to have already dismissed
the first one. If the product wants a "while you were away" nudge, that is a separate, explicitly
lower-frequency summary, not a re-send of the same digest.

### Retry, backoff, dead-letter

**Interviewer:** Provider returns a 500. Now what?

**Candidate:** First, classify the error, because retrying the wrong class of error is worse than
not retrying at all.

**Candidate:**
```text
RETRYABLE (transient)
  429 rate limited  -> honor Retry-After, else backoff
  500, 502, 503, 504, network timeout, connection reset
  action: exponential backoff with jitter, max 5 attempts

NOT RETRYABLE (permanent)
  400 malformed, 401/403 bad credentials, 422 invalid recipient
  action: fail fast, mark FAILED, update the user's channel health

SIDE-EFFECT UNKNOWN (the dangerous class)
  timeout AFTER the request was sent
  action: retry WITH the same provider idempotency key
          the provider collapses the duplicate; we may not double-send
```

**Candidate:** That third class is the one candidates forget, and it is the intersection of
retry and idempotency. A timeout is not a failure, it is an unknown, and an unknown must be
resolved by retrying with the same key, never by generating a new one. If I generate a new key on
retry, every timeout becomes a guaranteed duplicate.

**Candidate:**
```text
attempt 0: t + 0s          jitter 0-1s
attempt 1: t + 2s          jitter 0-2s
attempt 2: t + 4s          jitter 0-4s
attempt 3: t + 8s          jitter 0-8s
attempt 4: t + 16s         jitter 0-16s
attempt 5: t + 32s         jitter 0-32s
  -> dead letter queue, with full payload, for replay by an operator
```

**Interviewer:** Why jitter?

**Candidate:** Without jitter, retries synchronize. A provider has a 30-second blip, ten thousand
messages fail at the same instant, and without jitter all ten thousand come back at exactly
t + 2s and knock the provider over again the moment it recovers. Jitter spreads the retry load
across the window and lets the provider come back gradually. This is the single most important
detail in any retry design and it costs one line of code. See [[retry-and-timeout]]. The exact
exponential schedule, by the way, is more ritual than science; the principles are exponential
growth, a cap, jitter, and a total time budget.

**Interviewer:** The dead-letter queue. What is in it, and who looks at it?

**Candidate:** The full original envelope: the notification payload, the channel, the provider
that was targeted, the error class, the attempt count, and the timestamps. The important design
point is that the DLQ is a *replayable* source, not a graveyard. An operator or an automated
replayer can push a message back onto the channel queue once the provider is healthy, and because
every message carries its original idempotency key, replay cannot duplicate the ones that actually
succeeded. That is the payoff of keeping the key in the envelope: the DLQ becomes safe to replay
aggressively. I alert on DLQ depth and on the *replay success rate*, because a DLQ that is filling
is a provider problem, and a DLQ that is never drained is an incident nobody notices.

**Interviewer:** What is the delivery-status model?

**Candidate:** A lifecycle with explicit terminal states, and the schema is deliberately narrow.

**Candidate:**
```sql
CREATE TABLE notifications (
  notification_id  VARCHAR(64)  NOT NULL,
  user_id          VARCHAR(64)  NOT NULL,
  category         VARCHAR(64)  NOT NULL,
  channel          VARCHAR(16)  NOT NULL,   -- push | email | sms
  provider         VARCHAR(32),
  template_id      VARCHAR(64)  NOT NULL,
  template_version INT          NOT NULL,
  state            VARCHAR(16)  NOT NULL,   -- see state machine
  idempotency_key  VARCHAR(128) NOT NULL,
  attempts         SMALLINT     NOT NULL DEFAULT 0,
  created_at       BIGINT       NOT NULL,
  terminal_at      BIGINT,
  error_class      VARCHAR(32),
  provider_ref     VARCHAR(128),
  PRIMARY KEY (notification_id)
);

CREATE UNIQUE INDEX idx_notifications_idem ON notifications (idempotency_key);
CREATE INDEX idx_notifications_user_time ON notifications (user_id, created_at DESC);
```

**Candidate:**
```text
CREATED -> QUEUED -> SENT -> DELIVERED -> OPENED -> CLICKED
              |         |          |
              |         v          v
              |       BOUNCED    UNSUBSCRIBED
              |         |
              v         v
          RETRYING -> FAILED -> SUPPRESSED
```

**Candidate:** Notice the unique index on `idempotency_key`. That is the last line of defense: even
if Redis dedupe is bypassed and a retry generates a fresh key by mistake, the store rejects the
duplicate. And notice the webhook-driven transitions, which is the feedback path in my diagram:
providers call back with a delivery receipt, an open pixel, or a bounce, and I match on
`provider_ref`, so I need that column indexed. I also track bounce and complaint rates per user and
per address, because a hard-bouncing address will never deliver and I should stop trying, which is
both a cost saving and a deliverability requirement.

**Interviewer:** What is provider failure handling at the gateway layer?

**Candidate:** Three mechanisms working together. A circuit breaker per provider, so that when SMS
is returning 503s I stop sending after a threshold of failures and fail fast instead of burning my
retry budget on a provider that is down. Quota and rate limiting per provider, so I never exceed
what they allow, which is the constraint I identified in Phase 2 where SMS was 200x over capacity.
And channel fallback, which is the interesting one: if SMS capacity is exhausted, a high-value
transactional notification degrades to push rather than being dropped.

**Candidate:**
```text
send(notification):
  for channel in priorityOrder(n, urgency):
     if breaker(channel).isOpen():
         record FALLBACK_SKIPPED
         continue
     if not rateLimiter(channel).tryAcquire():   # token bucket, per provider
         record SUPPRESSED_QUOTA
         continue
     try:
        resp = providerGateway.send(channel, n, idempotency_key = n.idempotency_key)
        record SENT, provider_ref = resp.id
        return
     on retryable:
        enqueue(channel_queue, attempt + 1, backoff)
        return
     on permanent:
        record FAILED, mark channel unhealthy for user
        continue
  record FAILED_ALL_CHANNELS
```

**Candidate:** Note the circuit breaker and the token bucket are per provider but the fallback is
per notification, and the fallback preserves the same idempotency key across channels, which is
fine because providers scope their keys per channel anyway. The important invariant is that a
notification is never left in a state where nothing is queued and nothing is recorded, so every
path terminates in either a recorded send or a recorded failure.

**Interviewer:** Rate limiting to the provider. Be specific.

**Candidate:** A distributed token bucket in [[redis]], per provider, with a refill rate set to a
conservative fraction of the published quota, say 70 percent, so that a miscalculation on my side
does not immediately trigger a provider-side ban. Tokens are acquired in batches rather than one at
a time, because a round trip per SMS is wasteful at 21,000 per second. I also keep a concurrent
in-flight limiter, because provider quotas are often expressed as "no more than N concurrent
connections" rather than as a rate, and a rate limiter alone will not protect me from 10,000
simultaneous hanging requests. This is [[distributed-rate-limiter]] and
[[circuit-breaker]].

**Interviewer:** And per-tenant fairness, so one customer cannot starve everyone else?

**Candidate:** Per-tenant sub-buckets carved out of the global bucket, and that is a real
requirement in a platform product where customers are other businesses. I would implement it as
weighted fair queueing over the per-channel queues rather than as independent buckets, because
independent buckets let a noisy tenant consume all of its own bucket and still burst. Weighted fair
queueing gives per-tenant a guaranteed minimum share and a capped maximum, so a noisy neighbor
degrades gracefully instead of starving anyone. This is [[tenancy-and-cells]] thinking at the
fairness layer rather than the deployment layer.

### Android, iOS, and email gateways

**Interviewer:** What is actually different per channel at the gateway?

**Candidate:** Enough that they are separate services, and I will name the differences rather than
hand-wave them. For FCM, the interesting part is token lifecycle: a token can be stale, and an
`UNREGISTERED` response means I must delete that device token, or I accumulate dead tokens and my
send success rate silently falls while my costs go up. So the gateway's response handling mutates
token state, which makes it a write path as well as a send path. For APNs, the hard constraints
are the payload size cap of 4 KB, the requirement for HTTP/2 with persistent connections, and
per-device token connection management, and APNs also returns a timestamp and an environment
identifier that must be echoed back on unsubscription, which is a classic source of silent
notification loss if you drop it. For email, the mechanics are entirely different: you are
submitting MIME to an ESP over its API, and the ESP then owns the actual SMTP conversation, so
complaint and bounce reputation is managed by the ESP but driven by my list hygiene, which means
unsubscribe handling and bounce suppression are my responsibility.

**Candidate:** The unifying design point across all three: the gateway is an adapter, and the
adapter owns everything provider-specific, including the provider's idiosyncratic failure
semantics, so the sender services above it stay provider-agnostic. If APNs changes an error code
tomorrow, I change one adapter, not the SMS sender.

**Interviewer:** Anything about payload size that bites?

**Candidate:** Push is capped at 4 KB for APNs and 4 KB for FCM, and this constrains design rather
than being an afterthought. It means a push cannot contain the notification content at all; it
carries an identifier and the client fetches the content, or it carries a short generic string plus
the identifier. So push is really a *notification to look*, not a *message to read*, and that has
consequences for locales, since translating into 30 languages does not fit in 4 KB. The practical
answer is a short, pre-translated, category-level string on the device or a small per-category
string table, plus the identifier for the specific item.

Study separately: Push Notifications (APNs/FCM gateways), APNs token lifecycle and payload limits,
and ESP complaint and bounce reputation management.

---

## Phase 5: Trade-offs and Follow-ups

**Interviewer:** Rapid "why not" round.

**Candidate:** Go.

**Interviewer:** Why not have the domain services call your notification API synchronously?

**Candidate:** Because it makes the provider's availability a hard dependency of an unrelated
business operation, it adds provider latency to a user-facing request path, and it prevents the
orchestrator from doing the work that actually requires centralization, like deduplication across
services and rate limiting against a shared quota. A per-service quota cannot be enforced if every
service calls the provider itself, so you end up with a provider account that is shared and
unmanaged. The one exception I would allow is a genuine emergency path that bypasses the queue and
the quota, like an account-compromise alert, and even that I would route through a minimal
out-of-band path rather than through the general system.

**Interviewer:** Why not cron everything instead of streaming?

**Candidate:** Because the sub-5-second categories cannot tolerate it. A cron at 1-minute
granularity is a median of 30 seconds and a p99 of 60 seconds of added latency, which fails a 5
second requirement outright. And a 1-minute cron on a 750-million-a-day workload means
synchronously generating, rendering, and dispatching 12.5 million notifications every minute, which
is a latency and connection-pool problem by itself. Batching should be a *choice per category*,
imposed where it adds value, never a global mechanism.

**Interviewer:** Why not a single database for templates, preferences, and the log?

**Candidate:** Same answer as in any system of this size: different access patterns, different
lifetimes, different scaling shapes. The log is 137 terabytes a year and is written once and read
by paging; templates are 200 megabytes and are read 210,000 times a second; preferences are 15
gigabytes and are read 150,000 times a second. Putting the log and the preferences in one database
means the log's write volume dictates the shard count, which then over-shards the preferences and
makes them slower. And the log wants columnar or time-partitioned storage for retention and
analytical queries, which is a poor fit for the transactional preference rows. The right structure is
a small transactional store for templates and preferences, a wide time-partitioned store for the
log, and [[redis]] for the hot read paths of both small ones.

**Interviewer:** Why not publish-subscribe directly from the domain services to the senders,
skipping an orchestrator?

**Candidate:** Because then the decision logic, the preference check, the dedupe, the quiet-hours
routing, and the template resolution are duplicated in every producing service, and they will
diverge, and the shared rate limit against the provider becomes unenforceable because there is no
single place that knows how much quota is left. Fanout and aggregation, and centralized policy
enforcement, are the two things an orchestrator exists to provide. The tradeoff I accept is a
network hop and a small amount of latency, plus the orchestrator becoming a critical service that
needs to be highly available, which is exactly why the bus sits in front of it so producers are
never blocked by it.

**Interviewer:** Traffic doubles. What breaks first?

**Candidate:** The SMS channel, and it breaks not because of CPU but because I exceed the provider
quota, at which point my token bucket starts rejecting, the fallback path activates, and a bunch of
transactional notifications arrive as push instead of SMS. That is actually the designed behavior
and it is fine, provided the fallback priority is right. Second thing to break is the notification
log primary, because 1.5 billion rows a day is a real write load against one primary, and that is
where sharding by user id comes in. Third, and this is the one I would miss if I only watched
throughput, is the dedupe keyspace: 1.5 billion keys a day, so I need more Redis shards and I need
to shorten the TTL for push, which is a correctness-for-cost trade I would rather make explicitly
under load than discover.

**Interviewer:** The notification log's primary database dies. Walk me through it.

**Candidate:** The log is append-heavy and read by user, so it is sharded by user id, which means
a user's history is a single-shard query. Each shard is a leader with followers under
[[database-replication|synchronous replication to one follower, asynchronous to the second]].
On leader failure, promote the synchronous follower, lose nothing acknowledged, and accept a 5 to
15 second write pause during failover. Because the orchestrator is downstream of a durable queue,
the producers are completely unaffected: the events are already committed in Kafka and the queue
buffers them. So the notification log primary failing is a *delay* incident, not a loss incident,
and that property comes from the queue being upstream. I would not be as relaxed about this if the
orchestrator wrote synchronously to the log, which is the real argument for keeping the log write
off the hot path and instead writing it from the sender as a side effect.

**Interviewer:** Replica lag. Where does it bite you specifically?

**Candidate:** In the delivery-status feedback path, and in exactly one place: the "unread
notification count" on the user's bell icon, which people will genuinely notice being wrong. If
the read hits a lagging replica it under-counts, so users see a badge flicker or a badge that
disappears and comes back. Two fixes. One, read the count from the leader, which is correct and
costs me a little read capacity, and for a single small integer key that is affordable. Two,
compute the count from a separate aggregated counter store rather than from the log, updated by the
sender's state transitions, which is naturally a small hot key. And I will be honest that a
notification badge is the canonical example of a value where eventual consistency is fine, so the
real fix is to stop treating the badge as a strong-consistency read. Replica lag also affects
template reads, but those are versioned, so a lagging replica returning the previous version
renders a slightly old subject line, which is harmless.

**Interviewer:** If you had to cut the system in half to save money, what goes?

**Candidate:** In order. First, SMS as a general-purpose channel, retained only for security and
transactional alerts, because SMS is the expensive, rate-limited, complaint-prone channel and most
of its volume is marketing. Second, per-notification social notifications, collapsed to a daily
summary, which is the aggregation argument again and the single biggest volume reduction available.
Third, delivery-status tracking beyond the terminal state, so I stop recording opens and clicks and
keep only delivered and bounced, which removes an entire webhook write path. Fourth, retention on
the log, cutting hot retention from 30 days to 7 and pushing the rest to the warehouse. And I would
not cut dedupe, retries, or the dead-letter queue, because those are the things that turn a provider
outage into a user-visible complaint, and the whole point of this system is to be boring.

**Interviewer:** What would a candidate most commonly get wrong here?

**Candidate:** Four things. Calling the provider from the request path, which is the coupling
mistake. Forgetting the side-effect-unknown retry class, which is where duplicate SMS actually come
from. Treating digest as a batching window rather than as aggregation. And, most importantly,
building a system that is entirely throughput-driven and not quota-driven, when the quotas are the
actual ceiling. The instinct to over-provision compute is the wrong instinct for a system whose
hard limit is a contract with three external vendors.

**Interviewer:** Final summary. Two or three sentences, the version you want me to remember.

**Candidate:** Domain services emit durable events through an outbox into Kafka, and a notification
orchestrator consumes them, resolves recipients, evaluates per-category per-channel preferences and
urgency against quiet hours, drops duplicates on a content-derived idempotency key, renders
versioned and cached templates, and enqueues one message per enabled channel into per-channel
queues that give priority and failure isolation. Each channel sender runs behind a provider gateway
that owns a token-bucket rate limit, a circuit breaker, and channel fallback, and it retries
retryable failures with exponential backoff and jitter into a replayable dead-letter queue while
provider webhooks drive the delivery-status lifecycle back into the log. And the entire design is
paced by external provider quotas rather than by local compute, which is why every channel has an
admission-control decision, a fallback path, and an honest answer about what happens when the quota
runs out.

**Interviewer:** Thank you.

**Candidate:** Thank you.

---

## Concepts to Review Alongside This Transcript

- [[message-queue]] and [[kafka-architecture]] for the durable event bus and the replayable DLQ
- [[outbox-pattern]] for the dual-write problem between the producer's DB and the broker
- [[delivery-semantics]] for at-least-once plus idempotency as the workable contract
- [[idempotency]] and [[request-deduplication]] for the content-derived dedupe key design
- [[retry-and-timeout]] and [[circuit-breaker]] for the retry ladder and provider isolation
- [[distributed-rate-limiter]] for the token bucket that enforces provider quotas
- [[fanout-and-aggregation]] for the 10M-follower campaign, and why digest is aggregation
- [[delivery-and-retry]] for the state machine and webhook-driven status transitions
- [[database-replication]] and [[replication-lag]] for the log's failover and badge-read behaviour
- [[sharding]] and [[shard-key]] for sharding the log by user id and the dedupe keyspace
- [[tenancy-and-cells]] for weighted fair queueing so one tenant cannot starve others
- Study separately: Push Notifications (APNs/FCM gateways), APNs token lifecycle, and ESP
  complaint and bounce reputation management

---

## What I Must Know

### Must Know
- [[message-queue|Message Queue]]
- [[event-driven-architecture|Event-Driven Architecture]]
- [[outbox-pattern|Outbox Pattern]]
- [[retry-and-timeout|Retry and Timeout]]

### Good to Understand
- [[delivery-and-retry|Delivery and Retry]]
- [[idempotent-consumer|Idempotent Consumer]]
- [[circuit-breaker|Circuit Breaker]]
- [[rate-limiter|Rate Limiter]]
