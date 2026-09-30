# Payment System Design — Full Mock Interview Transcript

Target: Staff / Senior Engineer system design. Interviewer: Principal Engineer, Payments.
Candidate duration: 45 minutes of design conversation (the full session including questions, whiteboard, and follow-ups runs about 85 minutes).

Every line below is spoken dialogue. Diagrams, schemas, and API contracts are whiteboard artifacts the candidate produces while speaking, and are reproduced in fenced blocks between the lines that introduce them.

---

## Session Setup

Interviewer: Thanks for joining. You have a payment system in front of you. The prompt is on the shared doc. Take two minutes to read it, then let's talk. There are no trick requirements in there — everything listed is something the business has actually asked for. Think out loud. I will interrupt.

Candidate: Understood. Let me read it, then I will start with questions before I draw anything.

---

## Phase 1 — Requirements Clarification

Interviewer: Go ahead.

Candidate: Before I design anything I want to separate three things that get conflated: the money semantics, the correctness guarantees, and the operational shape. Let me ask about money semantics first.

Interviewer: Sure.

Candidate: Is this marketplace-style? Do we hold customer funds for a period before paying the merchant out, or is it authorize-then-immediately-capture, and capture is normally the same request?

Interviewer: Both. Subscriptions and utility bills are authorize now, capture at the end of the billing period. Marketplace orders capture at fulfilment and settle to the merchant on a weekly cycle. Digital goods in-app are capture immediately.

Candidate: So there are three payment shapes, and the difference is when funds become final and when they leave our control.

Interviewer: That's right. Capture makes funds final from the customer's side. Settlement to the merchant is a separate event, sometimes days later.

Candidate: Next question. Do we keep a stored customer balance, or do we only store movements and derive the balance?

Interviewer: Movements only. A stored balance is allowed as a materialized read model, but it is not the truth.

Candidate: That is the answer I was hoping for, and it is a load-bearing one, so let me state it back: the ledger journal is the source of truth, balances are a projection. If we lose a balance row we can rebuild it. If we lose a journal posting we have lost money.

Interviewer: Correct, and that asymmetry is exactly why the design is hard.

Candidate: Can a single payment be captured in several chunks, and can a refund exceed what has been captured?

Interviewer: Yes to both. Partial captures are common in shipping and split-fulfilment. Over-refund is not allowed — the refund amount cannot exceed captured minus already refunded. We reject that at the API.

Candidate: So the invariant is sum(captures) minus sum(refunds) is never negative, and never exceeds the authorized amount for an authorization-style payment. Is that a per-payment invariant enforced transactionally?

Interviewer: It must be. If you let it go negative you have created money from nothing and reconciliation will never close.

Candidate: Then it is a per-payment aggregate invariant and it dictates that all state transitions for one payment live in one place transactionally. I will come back to that when I pick a shard key.

Interviewer: Good. Currencies?

Candidate: Roughly forty. Two questions: do we transact in the merchant's local currency, and when is the FX rate decided?

Interviewer: Presentment currency is local. The rate is locked at authorization and stored on the payment, so a capture three days later uses the original rate. Settlement converts at the PSP's rate on the settlement date and we book the difference as an FX gain or loss.

Candidate: So I need two fields: the locked customer-facing rate, and a settlement-date rate whose difference posts to an FX gain-loss account in the ledger. That is a nice illustration of why the ledger needs more account types than "customer" and "merchant".

Interviewer: Yes. What is a "day" for the settlement report?

Interviewer: Whose timezone?

Candidate: I will ask. But my instinct: the ledger is timestamped in UTC always, and the settlement batch is partitioned by a settlement date that is derived from the merchant's contractual settlement timezone, stored as an explicit column. Never infer it from a server default.

Interviewer: Agreed. UTC in storage, contractual local date as a stored column, batch keyed on that column.

Candidate: Now correctness and failure. The one that scares me: the PSP call times out. I do not know if the money moved.

Interviewer: Correct, and you will get that timeout eventually. What do you do?

Candidate: I never retry a capture without first resolving the ambiguity. Concretely, the outbound call carries a stable idempotency key that I generate and persist before the call. On timeout, the payment is not moved to failed. It goes to a distinct state that I will call "unknown" or "pending_confirmation", and a resolver worker queries the PSP status endpoint with the same idempotency key. Only the resolver may move it to succeeded or failed. Retries reuse the original key, so the PSP collapses them.

Interviewer: Does the PSP guarantee that?

Interviewer: The one we integrate with does, and it is a contractual requirement in our integration spec, not an assumption. What happens if you talk to a PSP that does not?

Candidate: Then you fall back to a reconciliation-by-poll: the resolver calls a list-transactions endpoint for a time window and matches on our reference, and if that is also unavailable, the payment stays unknown and the merchant is told to hold fulfilment. Unknown is a legitimate, alertable state. It is not a failure state to be papered over.

Interviewer: How long will you tolerate unknown?

Candidate: I'd set a hard SLO. Something like 99.9% of payments reach a terminal state within 60 seconds, and anything still unknown after 15 minutes raises a P2 and gets human reconciliation.

Interviewer: Fine. Next: how much retry traffic do you see from your own clients?

Candidate: I want a number, because it drives write amplification. In mobile networks I'd guess 10 to 20 percent of attempts are client retries, not new business.

Interviewer: Call it 15 percent, and it is worse on bad networks, up to 30 percent in some markets.

Candidate: So my real write rate is 1.15 times business payment volume, and on top of that, every retry still costs me a full round trip to the PSP unless I collapse it. That makes the idempotency key not a nice-to-have, it is the difference between 1.15x PSP cost and 1.15x PSP cost plus a 15 percent duplicate charge rate. I'll come back to this.

Interviewer: Who is your top merchant by volume, and does anyone exceed a meaningful share?

Candidate: How many merchants, and is the distribution power-law?

Interviewer: Forty thousand merchants. The top one is about four percent of volume, and during a sale window on the fifteenth it peaks at nine percent. There is also a small number of "instant payout" merchants that we debit at unusual times.

Candidate: That is a hot partition waiting to happen. Four to nine percent of 1,400 peak TPS is 56 to 126 TPS on one merchant. That is a hot shard, and it is fixable, but I need to call it out early rather than discover it later.

Interviewer: Multi-region, or single region for now?

Candidate: Single region, multi-AZ, with a warm standby in a second region. Active-active for payments is a much larger project and it interacts badly with a strongly consistent ledger. I would rather be honest that the RTO is minutes, not milliseconds.

Interviewer: Acceptable for this exercise. Anything else before you estimate?

Candidate: Yes — the fraud question. Can fraud decision come after authorization?

Interviewer: Yes, and most of it does. We authorize first because that reserves the money, then score, then either let it stand, void it, or step it up to 3-D Secure.

Candidate: Then fraud cannot be on the synchronous critical path in a way that blocks the money. I will design it as an async consumer of the authorization event with a decision deadline.

---

## Phase 2 — Functional Requirements

Interviewer: Great. Walk me through your functional requirements.

Candidate: Let me list them and then group them by pipeline, because the grouping is the architecture.

Candidate: Create and authorize a payment against a funding source. Capture, full or partial, once or many times. Cancel or void an uncaptured authorization. Hold funds for an order. Release a hold. Forfeit a hold. Refund, full or partial, to the original instrument or to wallet credit. Wallet top-up, which itself is a payment. Wallet-to-wallet transfer, which is a single journal entry with two postings and no PSP involvement at all. Fetch payment status, fetch a list of payments for a payer, fetch a wallet balance, fetch a paginated statement or ledger extract. Receive a signed webhook from the PSP and reconcile it into our state. Emit webhooks to merchants on every terminal state change. Score asynchronously. Produce a daily reconciliation report against the PSP statement file. Produce a per-merchant settlement report. Intake a chargeback, which reverses a settled payment and creates a liability.

Candidate: That is fifteen capabilities. Now the ones that are actually load-bearing and that I think most candidates get wrong:

Candidate: One, wallet-to-wallet transfer is not a payment. It is a single ledger entry. No PSP, no idempotency key against a third party, just a double-entry posting, and it must be atomic across both accounts or not exist.

Candidate: Two, "fetch wallet balance" is a read of a materialized projection, not a sum over the journal. Summing the journal on read is fine as a rebuild job and catastrophic as a query path.

Candidate: Three, the webhook to the merchant is a separate, independently-failing delivery problem from the webhook from the PSP. They are inbound and outbound and they have opposite reliability postures. Inbound PSP webhooks must be treated as untrusted, unordered, duplicated, and possibly replayed from months ago. Outbound merchant webhooks need retry, backoff, DLQ, and a per-endpoint circuit breaker.

---

## Phase 3 — Non-Functional Requirements

Interviewer: Non-functional requirements. What are the ones you will design against?

Candidate: Let me rank them, because ranking is the point.

Candidate: Correctness and auditability are non-negotiable. A single lost or duplicated financial movement is an existential problem, not a bug. That forces: integer minor units only, an append-only journal, and durable writes before any external call whose result we cannot reconstruct.

Candidate: Availability target is 99.95% on payment creation, read availability can be 99.9% from cache and read model. Latency budget: p99 under 400 milliseconds for "accept a payment", not "settle a payment", because settlement is asynchronous by nature.

Candidate: The synchronous path must not wait on fraud, not wait on the PSP beyond a bounded budget, and not wait on downstream analytics or notification. I will put a hard timeout on the PSP call — say 8 seconds end to end including connect — and above that I declare unknown and let the resolver work.

Candidate: Durability: synchronous replication, acknowledged by a quorum, before I tell a client "accepted". No write acknowledged without being durable on a quorum of replicas.

Candidate: Isolation: the payment aggregate is the consistency boundary. Inside one shard, serializable or at minimum read-committed with explicit SELECT FOR UPDATE on the payment row. I am not going to try to make two shards agree.

Candidate: Recoverability: RPO zero for the ledger, RTO under five minutes for a regional failure, and a tested reconciliation that can rebuild any read model from the journal.

Candidate: Auditability: every state transition writes an append-only event with actor, timestamp, source IP, idempotency key, and reason code. No updates, no deletes, and the archive is WORM after 90 days.

Interviewer: Anything about cost or compliance that constrains the architecture?

Candidate: Cost: ledger retention is 7 years, so I will tier — hot on NVMe for 90 days, warm on SSD, cold in object storage with a manifest. Object storage is 20 to 30 times cheaper per terabyte and I only need to query cold data for audits.

Candidate: Compliance: I will not touch PAN data. The client talks to the PSP directly for card entry and hands me a token. That is not just a security control, it removes PCI scope from my entire system and it is a hard architectural boundary. I also need field-level encryption on the customer's stored instrument metadata, and a secrets manager for PSP credentials with rotation.

Study separately: [[encryption-and-keys|Encryption and Key Management]]

---

## Phase 4 — Scale Estimation

Interviewer: Now estimate. What are the inputs and what comes out?

Candidate: I will read the inputs back: 50 million registered users, 15 million DAU, about 2 payments per DAU per day, a 15 percent client retry rate, forty thousand merchants, forty currencies.

Interviewer: Correct, plus a 4x peak-to-average ratio concentrated in a three-hour evening window and on the fifteenth of the month.

Candidate: DAU to MAU is 30 percent, which is high but plausible for a super-app, and it means the registration base is not my binding constraint. Payments per day is 15 million times 2, so 30 million business payments per day.

Candidate: Average write rate is 30 million over 86,400 seconds, which is 347 payments per second. Let me do that division carefully: 30,000,000 divided by 86,400 is 347.2. So call it 350 average TPS.

Candidate: Peak is 4x, so 1,400 TPS. During the sale window on the fifteenth, with the top merchant at 9 percent, that is 126 TPS on a single merchant.

Candidate: Now write amplification. 15 percent retries on top means 34.5 million write attempts per day, 400 per second average. But I want to count the internal writes, not the client attempts, because that is what hits my database.

Candidate: Each business payment generates on average four state transitions — initiated, authorized, captured, and one terminal bookkeeping event — and each transition writes: one updated payment row, one append-only payment event, two to four ledger postings, and one outbox row. That is roughly 4 times (1 plus 1 plus 3 plus 1) equals 24 internal row operations per business payment, so 30 million times 24 is 720 million row operations per day. That is 8,300 rows per second on average and about 33,000 per second at peak.

Interviewer: That is a lot. Is that the real number?

Candidate: It is the honest number and it is why I will not co-locate everything in one database forever. But the important part is that most of those rows are append-only and never read on the hot path. Only the payment row is read-modify-write. So the read-write shape is asymmetric in the best way: 8,300 appends per second and about 1,400 update-and-read cycles per second.

Candidate: In-flight concurrency. The PSP p50 is 800 milliseconds, p99 is 3 seconds. At 1,400 peak TPS with an average PSP dwell of 2 seconds, I have 2,800 concurrent outbound HTTPS calls. If my payment service runs on an eight-core node that gives me roughly 32 usable inbound threads, I would need about 88 nodes just to hold connections, before doing any work. That is the single most important number in this design.

Interviewer: What do you do with that number?

Candidate: It forces a fully asynchronous contract. The client asks to pay, I durably record the intent, and I return 202 Accepted with a payment id and a status URL. Completion is delivered by PSP webhook or by my own resolver poller. The client polls or listens over server-sent events. I never hold a request thread for the duration of a 3-D Secure human challenge, which can be 45 seconds. Under no circumstance is this a synchronous REST call to completion.

---

## Phase 5 — Traffic Estimation

Interviewer: Read traffic.

Candidate: Per business payment I count: one list-page fetch, two or three detail or status refreshes while the client waits, one wallet balance read, and a partial statement read for a fraction of users. Call it nine reads per payment, and note the retries also cause status polls, so in bad-network markets it is more like twelve.

Candidate: 30 million times 9 is 270 million reads per day. 270,000,000 divided by 86,400 is 3,125 reads per second average, and at 4x peak that is 12,500 reads per second.

Candidate: Webhook traffic in both directions. Outbound: 30 million terminal-state webhooks per day, but fanned out, so if a payment has 2.2 merchant subscriptions on average that is 66 million deliveries per day, 764 per second average. Inbound from the PSP: one webhook per state transition, 120 million per day, 1,400 per second, and these arrive in bursts when the PSP has an incident and replays.

Interviewer: Bursts are worth quantifying.

Candidate: If the PSP has a 10 minute degraded period and holds webhooks, that is 600 seconds times 1,400 per second equals 840,000 inbound webhooks arriving as fast as the ingress will let them. I need a bounded, rate-limited consumer with backpressure and a DLQ, not a thread pool that melts.

Candidate: Reconciliation traffic: 30 million payments per day, so the daily PSP statement file has up to 30 million lines. Parsing and three-way matching that is a batch job — 30 million lines at, say, 50,000 lines per second per worker is 600 worker-seconds, so ten workers for ten minutes. That is a real batch, not a cron that runs in a minute.

Candidate: Analytics: 30 million payments plus 120 million ledger postings per day going into the warehouse. I will publish from the outbox bus, never query the OLTP database for analytics. If I let the warehouse hammer the primary with a 30 million row daily extract, I have built a self-inflicted denial of service.

---

## Phase 6 — Storage Estimation

Interviewer: Storage.

Candidate: Per business payment: payment row about 1.0 KB with the instrument metadata, FX snapshot, and state; payment events, 4 times 0.35 KB, equals 1.4 KB; ledger postings, average 3 times 0.25 KB, equals 0.75 KB; outbox events, about 5 times 0.4 KB, equals 2.0 KB. Total raw is about 5.15 KB per payment. Let me round to 5 KB.

Candidate: 30 million times 5 KB is 150 GB per day of raw rows. Times 365 is 54.75 TB per year, call it 55 TB. Over a 7-year retention that is roughly 385 TB.

Candidate: Now replication and B-tree overhead. B-tree with fill factor and index overhead multiplies by about 1.5, and two extra replicas means 3.6 times the raw in the cluster. 385 TB times 3.6 is about 1.4 PB across replicas, or 385 TB of distinct data. That is fine for a clustered OLTP store, but it is not fine on one class of disk for seven years, which is why I tier.

Candidate: Tiering. Hot tier: 90 days of data, which is 90 times 150 GB equals 13.5 TB. That fits comfortably on NVMe. Warm tier: months 4 to 24, so 21 months times 5.1 GB per day — let me recompute, 150 GB per day times 640 days is 96 TB on cheaper SSD. Cold tier: years 3 to 7, about 273 TB in object storage at roughly 5 dollars per TB per month, so about 1,365 dollars per month versus roughly 16,000 dollars per month on provisioned SSD for the same bytes. That is the justification for the archive, and I can state that number to the business.

Candidate: The audit trail. Same volume as payment events, 120 million rows per day, 42 GB per day, 15 TB per year. Append-only, no index beyond a surrogate key, and after 90 days I write it to WORM object storage with a hash chain so any tampering is detectable. Let me say I chain each event to the previous event's hash per payment, so a deleted or edited row breaks the chain verifiably.

Candidate: Cache sizing. Balances: 50 million accounts times about 300 bytes for key plus value plus hash overhead is 15 GB. I will cache balances because every checkout reads one and it is the hottest read in the system. Payment status: caching all 30 days of payments would be 900 million entries times 400 bytes is 360 GB, too much for a commodity Redis cluster, so I cache only in-flight payments plus 24 hours of history. 24 hours is 30 million entries times 400 bytes is 12 GB. Comfortable.

Interviewer: Money representation. Where does that show up in the schema?

Candidate: Every amount is a 64-bit integer of minor units plus an ISO 4217 currency code. Never a float, never a decimal stored as a float. And I keep an exponent table, because JPY and KRW have zero minor units and BHD, KWD, and TND have three. If I hardcode "cents", I will be off by a factor of a thousand in Kuwait. The conversion from major to minor happens at the edge API layer, using the currency's exponent, and I reject an amount that is not representable in minor units rather than rounding it silently.

---

## Phase 7 — API Design

Interviewer: Design the API. What does a client actually call?

Candidate: I will use a versioned REST surface with an explicit idempotency contract, and I will show the money-moving calls. The rule is that every endpoint that can cause a duplicate financial effect requires an `Idempotency-Key` header, and I will reject the request with 400 if it is missing on a create.

Candidate: Create and authorize a payment:

```http
POST /v1/payments
Idempotency-Key: 7f3a9c1e-2b44-4d0a-9c11-8e5f0a3b6d21
Content-Type: application/json
Authorization: Bearer <merchant-scoped token>

{
  "amount": { "value": 4599, "currency": "USD" },
  "instrument": { "type": "card", "token": "tok_1P9xQ2" },
  "payer_id": "usr_8812",
  "merchant_id": "mch_0042",
  "order_reference": "ORDER-99ac",
  "capture_mode": "manual",
  "statement_descriptor": "ACME*STORE 44",
  "metadata": { "cart_id": "cart_712" }
}

202 Accepted
Location: /v1/payments/pay_01HQ8X2K
{
  "id": "pay_01HQ8X2K",
  "status": "processing",
  "amount": { "value": 4599, "currency": "USD" },
  "status_url": "/v1/payments/pay_01HQ8X2K",
  "expires_at": "2026-09-29T18:04:00Z"
}
```

Candidate: The 202 with a status URL rather than a 200 with a result is deliberate, because the result is not knowable within a request budget. I also return an idempotency receipt so the client can distinguish "I created this" from "I replayed your earlier request".

Candidate: Capture, refund, and read:

```http
POST /v1/payments/{payment_id}/captures
Idempotency-Key: <uuid>

{ "amount": { "value": 4599, "currency": "USD" } }
→ 201 { "capture_id": "cap_01HQ8X3M", "status": "succeeded" }

POST /v1/payments/{payment_id}/refunds
Idempotency-Key: <uuid>

{
  "amount": { "value": 1500, "currency": "USD" },
  "reason": "damaged_item",
  "destination": "original_instrument",
  "reference": "RMA-4412"
}
→ 201 { "refund_id": "ref_01HQ8X4N", "status": "pending" }

GET /v1/payments/{payment_id}
→ 200 {
      "id": "pay_01HQ8X2K",
      "status": "captured",
      "amount": { "value": 4599, "currency": "USD" },
      "captured": [{ "id": "cap_01HQ8X3M", "value": 4599 }],
      "refunded": [],
      "refundable": { "value": 4599, "currency": "USD" },
      "timeline": [
        { "state": "initiated", "at": "2026-09-29T17:59:31Z" },
        { "state": "authorized",  "at": "2026-09-29T17:59:33Z" },
        { "state": "captured",    "at": "2026-09-29T18:00:02Z" }
      ]
    }
```

Candidate: Money movement is idempotent at three levels, and I want to be precise about which one does the work. Level one, the `Idempotency-Key` on my API collapses duplicate client requests inside my own store. Level two, the stable PSP idempotency key, generated and persisted before the outbound call, collapses duplicate outbound calls at the PSP. Level three, the natural uniqueness of the journal, where a repeated posting attempt finds an existing entry and no-ops. Level one alone is insufficient because my service can crash after committing the key and before the PSP call, and then the retry has a key but no payment. Level two alone is insufficient because the PSP call is not the only side effect — my outbox and my ledger also move.

Candidate: The idempotency store semantics:

```http
POST /v1/payments
Idempotency-Key: 7f3a9c1e-...

→ first call   : 202 { "id": "pay_01HQ8X2K", ... }
→ replay 1     : 202 { "id": "pay_01HQ8X2K", ... }   (identical body = same response)
→ replay 2     : 409 { "error": "idempotency_key_reuse_mismatch",
                       "message": "key was used with a different request body" }
→ in flight    : 409 { "error": "idempotency_request_in_progress" }
→ 24h later    : 201 { "error": "idempotency_key_expired" }  → treat as new, alert
```

Candidate: I store a hash of the canonicalized request body alongside the key. Same key plus different body is a client bug, and silently replaying is worse than failing. Keys expire after 24 hours, which is comfortably longer than any sane client retry window but bounded so the table does not grow forever. I also record the key's source IP and merchant so I can alert on a key being reused across merchants, which is a strong fraud signal.

Candidate: Internal service contracts. The orchestrator to the ledger is a synchronous function call, not a network call, and I will explain why when I show the architecture. The fraud worker and webhook dispatcher are event consumers. The PSP adapter is an anti-corruption layer — one interface, `authorize`, `capture`, `void`, `refund`, `getStatus`, with per-PSP implementations, so the rest of the system never sees a PSP-specific field.

Candidate: The outbound webhook contract to merchants:

```http
POST https://merchant.example/webhooks/payments
X-Pay-Event-Id: evt_01HQ8X5P
X-Pay-Signature: t=1759172402,v1=5257a869e7ecebeda32affa62cdca3fa51cad7e77a0e56ff536d0ce8e108d8bd
X-Pay-Delivery-Attempt: 3

{
  "id": "evt_01HQ8X5P",
  "type": "payment.captured",
  "created_at": "2026-09-29T18:00:02Z",
  "livemode": true,
  "data": { "payment_id": "pay_01HQ8X2K", "amount": { "value": 4599, "currency": "USD" } }
}
```

Candidate: I use a timestamped HMAC-SHA256 signature so a replayed capture is detectable, and the merchant can verify without a network call back to me. I do not retry non-2xx forever; I retry on timeouts, 5xx, and 429, and I stop after a bounded number of attempts and DLQ it. Terminal delivery state lives in a webhook delivery table the merchant-facing portal can replay from.

---

## Phase 8 — High-Level Architecture

Interviewer: Draw it.

Candidate: Here is the whole system. I will walk it top to bottom, then zoom into the write path.

```text
                          +--------------------------+
                          |   Mobile / Web Client    |
                          |  tokens, no PAN, no card |
                          +------------+-------------+
                                       | HTTPS
                                       | Idempotency-Key
                                       v
+------------------------------------------------------------------------------------+
|  EDGE:  CDN -> WAF -> API Gateway                                                  |
|  TLS termination, OAuth/JWT validation, per-merchant rate limit, request signing   |
+---+------------------+-------------------+--------------------+------------------+
    |                  |                   |                    |
    v                  v                   v                    v
+-----------+  +--------------+   +---------------+    +------------------+
| Payment   |  | Wallet       |   | Merchant      |    | Webhook Hub      |
| API       |  | Service      |   | Portal (BFF)  |    | delivery + logs  |
+-----+-----+  +------+-------+   +---------------+    +--------+---------+
      |                |                                     |
      |     +----------+                                     | retry + backoff
      v     v                                                v
+--------+---+  +------------------+          +-------------------------------+
| Payment     |  | Fraud Worker    |          | Outbound Webhook Dispatchers   |
| Orchestrator|--| async scoring   |          | per-endpoint, DLQ, breaker   |
| state machine|  +--------+---------+          +---------------+---------------+
+---+-----+---+           |                                    |
    |     |               |                                    |
    |     |               v                                    v
    |     |     +------------------+            +-------------------+
    |     |     | Feature/Risk     |            | External PSP      |
    |     |     | Store (300ms SLA)|            | + 3-D Secure      |
    |     |     +------------------+            +-------------------+
    |     |
    |     |  ONE ATOMIC TRANSACTION (same shard, same DB)
    |     v
    |  +--------------------------------------------------------------+
    |  |  PAYMENT DB  (clustered OLTP, sharded by payer_id)          |
    |  |    payments | payment_events(append-only) | idempotency_keys |
    |  |    ledger_entries(append-only, double-entry)                 |
    |  |    outbox_events | refund_allocations | settlement_batches  |
    |  +----+----------------------------+---------------------------+
    |       | durable append              | outbox relay polls
    |       v                             v
    |  +------------------+     +------------------------------+
    |  | Balance /        |     |  Message Broker (Kafka / SQS) |
    |  | Statement        |     |  partitioned, replayable    |
    |  | Read Model       |     +---+---+---+---+---+----+-----+
    |  | (async, rebuild) |         |   |   |   |   |    |
    +--+------------------+         |   |   |   |   |    |
       |                            v   v   v   v   v    v
       |            +---------+ +--------+ +--------+ +--------+ +--------+
       +----------->| Analytics| | Notif. | | Fraud  | |Webhook | | Recon  |
                    | (warehouse)| | (SMTP) | | decision| | fanout | | trigger|
                    +-----------+ +--------+ +--------+ +--------+ +--------+

  SEPARATE STORAGE LANES
  +---------------------------+   +---------------------------+   +--------------------+
  | Archive / WORM Object     |   | PSP Statement Files       |   | Merchant Reports    |
  | 90d+ events, 7y retention |   | (S3) -> batch matcher     |   | (Parquet -> signed)  |
  +---------------------------+   +---------------------------+   +--------------------+

  ASYNC EDGE PATHS (not on the synchronous path)
  PSP --webhook--> Ingress --> verify signature --> dedupe --> queue --> orchestrator
  Orchestrator --outbound--> PSP adapter (timeout 8s, circuit breaker, PSP idempotency key)
  Unresolved payments (>15m) --> Resolver poller --> PSP status API
```

Candidate: Nine components plus four storage lanes. Let me name them so the diagram is not decorative: Payment API, Wallet Service, Merchant Portal BFF, Webhook Hub, Payment Orchestrator, Fraud Worker, Ledger and Payment DB with a materialized read model, Message Broker, and the async resolver. Behind that, the archive, the PSP statement files, and the reconciliation job.

Candidate: The single most important structural decision is the box in the middle. The payment state and the ledger postings are written in **one database transaction on the same shard**. That is not an implementation detail, it is the reason the whole design is tractable.

---

## Phase 9 — Request and Data Flow

Interviewer: Walk me through a card payment end to end.

Candidate: Eight steps.

Candidate: Step one, the client calls `POST /v1/payments` with an idempotency key and a PSP token. The gateway authenticates the merchant, rate-limits per merchant, and forwards to the Payment Orchestrator.

Candidate: Step two, the orchestrator claims the idempotency key. It inserts into `idempotency_keys` with a unique constraint on (merchant_id, key). If the insert conflicts, it reads the stored response and returns it. This is a compare-and-set on the database doing the work, not an application-level check, because a check-then-insert has a race.

Candidate: Step three, in the same transaction, it inserts the `payments` row in state `initiated`, generates the PSP idempotency key — this is a deterministic function of my payment id, which is the trick that makes step five's retry safe — and inserts the outbox event `payment.initiated`.

Candidate: Step four, the transaction commits. Durability is a quorum acknowledgement. **This is the last point at which the client could be told anything.** After this, whatever happens, I can reconstruct what I intended to do.

Candidate: Step five, the outbox relay reads unpublished outbox rows and publishes to the broker with the event id as the broker key. A separate worker, the PSP command worker, consumes `payment.initiated`, calls the PSP adapter with the persisted PSP idempotency key, and on success emits `payment.authorized`.

Candidate: Step six, on `payment.authorized` the orchestrator opens a new transaction: lock the payment row, transition to `authorized`, insert the ledger postings, insert the payment event, insert the outbox event. The ledger postings for an authorization move value from the PSP clearing account to the customer liability account. No money has left, but a liability now exists.

Candidate: Step seven, Fraud Worker consumes `payment.authorized`, calls the risk service with a 300 millisecond budget, and emits `fraud.decision` as allow, review, step_up, or block. Allow is a no-op. Block emits a `void` command to the PSP and, if the void fails, a `refund` command — the compensating action.

Candidate: Step eight, for the synchronous-feeling UX, the client polls `GET /v1/payments/{id}` or subscribes to server-sent events fed by the outbox broker. I return the read-model version, which may be up to a second stale, and I include the authoritative state when the read model is behind by less than 500 milliseconds. If the client ever needs ground truth, the timeline in the response comes from the authoritative table.

Interviewer: The dual-write problem. You write to the database and then publish to the broker. Walk me through what happens if the process dies in between.

Candidate: That is exactly the failure that motivates the outbox pattern, and I want to be explicit that I am not using it as a nice pattern, I am using it because a plain "commit, then publish" has a hole and a plain "publish, then commit" has the mirror hole.

Candidate: Commit-then-publish: the row commits, the process dies before publishing. Now the payment exists as `initiated` forever and the PSP is never called. The customer is charged nothing but the order is stuck.

Candidate: Publish-then-commit: the event is on the broker, the transaction rolls back. Now a consumer acts on a payment that does not exist. Worse, because the consumer cannot tell.

Candidate: The outbox pattern closes both. I insert the outbox row inside the same transaction as the business row, so it is atomically committed or not. A relay then polls the outbox and publishes. If the relay dies mid-publish, the event is published twice, not zero times, and every consumer is idempotent on event id. So the guarantee I get is **at-least-once delivery plus idempotent consumers, which yields effectively-once business effect**, not exactly-once delivery. I want to say that precisely, because brokers do not give me exactly-once end to end and claiming it would be a mistake.

Candidate: The relay itself needs care: I poll with `SELECT ... FOR UPDATE SKIP LOCKED` so multiple relay instances do not double-publish the same batch, and I have a sweeper that alerts on outbox rows older than 30 seconds, which is a direct detector for the stuck-payment scenario.

Study separately: [[outbox-pattern|Outbox Pattern]] and [[exactly-once-effect|Exactly-Once Semantics]]

Interviewer: What about the inbound direction, the PSP webhook?

Candidate: Symmetric problem. PSP posts `payment.succeeded` to my ingress. I verify the HMAC signature against a per-PSP secret from the secrets manager, reject on failure with 401 and log. I dedupe on the PSP's event id with a unique constraint, and I acknowledge with 200 **only after the event is durably queued**, not after I have processed it. If processing fails, I return 5xx and the PSP redelivers; my dedupe table makes that safe. I never trust the payload's status field alone — I re-derive state through the state machine, so a PSP that sends events out of order or replays a month-old event cannot move a captured payment back to processing. The state machine rejects illegal transitions.

---

## Phase 10 — Database Design

Interviewer: Schema. And I want to see the ledger, specifically.

Candidate: Primary key strategy first, because it dictates everything else. I use a globally unique, time-sortable identifier — a ULID or a snowflake-style scheme — as the primary key. It is monotonically increasing, so B-tree inserts land on the rightmost page instead of scattering, and it is roughly time-ordered so I can range-scan by time without a secondary index. I do not use UUIDv4 as a primary key for exactly that reason. Distributed ID generation has trade-offs; I will come back to it in follow-ups.

Candidate: The payment table:

```sql
CREATE TABLE payments (
    payment_id           ULID        PRIMARY KEY,
    merchant_id          BIGINT      NOT NULL,
    payer_id             BIGINT      NOT NULL,
    payer_country        CHAR(2)     NOT NULL,

    amount_minor         BIGINT      NOT NULL,
    currency_code        CHAR(3)     NOT NULL,
    fx_rate_locked       DECIMAL(20,10),
    fx_rate_settled      DECIMAL(20,10),

    instrument_type      SMALLINT    NOT NULL,   -- 1 card, 2 wallet, 3 bank mandate
    instrument_ref       VARCHAR(64),
    instrument_country   CHAR(2),

    status               SMALLINT    NOT NULL,   -- state machine code
    capture_mode         SMALLINT    NOT NULL,   -- 0 automatic, 1 manual
    authorized_minor     BIGINT      NOT NULL DEFAULT 0,
    captured_minor       BIGINT      NOT NULL DEFAULT 0,
    refunded_minor       BIGINT      NOT NULL DEFAULT 0,

    psp_id               SMALLINT    NOT NULL,
    psp_payment_ref      VARCHAR(64),
    psp_idempotency_key  VARCHAR(64) NOT NULL,
    attempt_count        SMALLINT    NOT NULL DEFAULT 0,

    order_reference      VARCHAR(64),
    statement_descriptor VARCHAR(32),
    metadata             JSONB,

    created_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at           TIMESTAMPTZ,
    settlement_date      DATE,

    CONSTRAINT captured_within_authorized
        CHECK (captured_minor <= authorized_minor),
    CONSTRAINT refunded_within_captured
        CHECK (refunded_minor <= captured_minor),
    CONSTRAINT amount_positive
        CHECK (amount_minor > 0)
);

CREATE INDEX idx_payments_payer_created
    ON payments (payer_id, created_at DESC);
CREATE INDEX idx_payments_merchant_created
    ON payments (merchant_id, created_at DESC);
CREATE UNIQUE INDEX uq_payments_psp_ref
    ON payments (psp_id, psp_payment_ref)
    WHERE psp_payment_ref IS NOT NULL;
```

Candidate: The three counters `authorized_minor`, `captured_minor`, and `refunded_minor` are the crux. They are maintained inside the same transaction that appends the ledger postings, and the `CHECK` constraints are the last line of defence. A negative refundable amount is impossible even if application code has a bug, because the database refuses the row.

Candidate: The state machine lives in the status code and is enforced in application code with an explicit transition table, because a `CHECK` constraint across a growing transition set gets unreadable:

```text
    initiated ──> authorized ──> captured ──> settled
        |             |             |
        |             v             v
        +--------> failed      refunded (partial or full)
                      |
                      v
                   voided
```

```text
    initiated ──> processing ──> authorized ──> captured ──> settled
        |              |              |            |           |
        |              |              v            v           v
        |              |           failed      refunded    chargeback
        |              |              |
        |              v              v
        |          unknown          voided
        |              |
        |              v
        |      resolver -> authorized | failed
        |
        v
     cancelled
```

Candidate: `processing` is the pre-PSP-call state, `unknown` is the ambiguous-PSP-outcome state, and the transition out of `unknown` is **only** permitted by the resolver. I encode that as a transition guard: the command carries an actor, and only the actor named `resolver` or `psp_webhook` may exit `unknown`. Anything else attempting it gets a 409 and an alert.

Candidate: The payment events table, append-only:

```sql
CREATE TABLE payment_events (
    event_id         ULID       PRIMARY KEY,
    payment_id       ULID       NOT NULL,
    sequence_no      INT        NOT NULL,
    from_state       SMALLINT,
    to_state         SMALLINT   NOT NULL,
    actor            VARCHAR(32) NOT NULL,   -- client | psp_webhook | resolver | fraud | ops
    reason_code      VARCHAR(32),
    amount_delta_minor BIGINT,
    idempotency_key  VARCHAR(64),
    source_ip        INET,
    trace_id         VARCHAR(64),
    payload          JSONB,
    occurred_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    prev_event_hash  BYTEA,
    event_hash       BYTEA      NOT NULL,
    CONSTRAINT uq_payment_event_seq UNIQUE (payment_id, sequence_no)
);
```

Candidate: No update, no delete, ever. Grant is INSERT-only and SELECT-only at the database role level, and the table is replicated into WORM storage after 90 days. The `prev_event_hash` chain means a retroactive edit to any event is detectable by recomputing the chain. That is the tamper-evidence argument I would make to a regulator.

Candidate: Now the ledger. This is the part I care most about.

```sql
CREATE TABLE ledger_accounts (
    account_id     BIGSERIAL   PRIMARY KEY,
    account_code   VARCHAR(64) NOT NULL UNIQUE,  -- 'LIAB:customer:usr_8812'
    owner_type     SMALLINT    NOT NULL,         -- customer | merchant | psp | platform | fx
    owner_id       BIGINT,
    currency_code  CHAR(3)     NOT NULL,
    account_type   SMALLINT    NOT NULL,         -- asset | liability | revenue | expense
    is_active      BOOLEAN     NOT NULL DEFAULT TRUE,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ledger_entries (
    entry_id        ULID       PRIMARY KEY,
    journal_id      ULID       NOT NULL,   -- groups the postings of one business event
    account_id      BIGINT     NOT NULL REFERENCES ledger_accounts(account_id),
    direction       SMALLINT   NOT NULL,   -- +1 debit, -1 credit
    amount_minor    BIGINT     NOT NULL,
    currency_code   CHAR(3)     NOT NULL,
    payment_id      ULID,
    merchant_id     BIGINT,
    posted_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    business_date   DATE       NOT NULL,
    description     VARCHAR(128),
    reversal_of     ULID                  -- points at the entry being reversed
);
CREATE INDEX idx_entries_account_posted ON ledger_entries (account_id, posted_at DESC);
CREATE INDEX idx_entries_journal        ON ledger_entries (journal_id);
CREATE UNIQUE INDEX uq_entries_reversal ON ledger_entries (reversal_of)
    WHERE reversal_of IS NOT NULL;   -- a reversal can happen exactly once
```

Candidate: Double-entry means every journal has postings that sum to zero within a currency. Authorize 4,599 USD for a customer against a card:

```text
  journal J-1001  (authorize)
    +4599  DR  ASSET:psp_clearing:stripe          (we now hold a receivable)
    -4599  CR  LIAB:customer:usr_8812            (customer's held funds)
    sum = 0
```

Candidate: On capture, the customer liability is extinguished and the value becomes a payable to the merchant, net of the platform's commission and the PSP's processing cost. Let me be careful with the signs, because this is where double-entry implementations go wrong: a debit reduces a liability, a credit reduces an asset.

```text
  journal J-1002  (capture, commission 8% = 368, PSP cost 1.9% = 87)
    +4599  DR  LIAB:customer:usr_8812            (liability extinguished)
      -368  CR  REV:platform:commission          (revenue recognised)
       -87  CR  EXP:platform:psp_fees            (cost of the rail)
    -4144  CR  LIAB:merchant:mch_0042            (merchant payable)
    sum = +4599 - 368 - 87 - 4144 = 0
```

Candidate: On refund, I do not edit anything. I post a new journal that mirrors the original postings of the amount being refunded:

```text
  journal J-1003  (refund 1500, pro-rata reversal of J-1002)
    +1352  DR  LIAB:merchant:mch_0042            (reduce what we owe merchant)
     +120  DR  REV:platform:commission_reversal  (un-recognise revenue)
      +28  DR  EXP:platform:psp_fees_reversal    (the fee is not returned)
    -1500  CR  LIAB:customer:usr_8812            (customer gets the money back)
    sum = +1352 + 120 + 28 - 1500 = 0
```

Candidate: Note the fee lines. The PSP does not refund its processing fee, so a refund is not symmetric with the capture. That asymmetry is exactly the kind of detail that is invisible until finance files a bug, and it is why the reversal is an explicit journal and not "undo the last one".

Candidate: The invariant check is the single most important query in the system, and it runs continuously, not just nightly:

```sql
SELECT journal_id, currency_code,
       SUM(CASE WHEN direction = 1 THEN amount_minor ELSE -amount_minor END) AS net
FROM ledger_entries
GROUP BY journal_id, currency_code
HAVING SUM(CASE WHEN direction = 1 THEN amount_minor ELSE -amount_minor END) <> 0;
```

Candidate: Zero rows, always. If that query ever returns a row, money was invented. I run it per-shard continuously as a cheap streaming check and in full as a nightly batch, and it pages the on-call.

Candidate: The idempotency table, and the outbox:

```sql
CREATE TABLE idempotency_keys (
    merchant_id       BIGINT      NOT NULL,
    idempotency_key   VARCHAR(64) NOT NULL,
    request_hash      BYTEA       NOT NULL,   -- sha256 of canonicalized body
    payment_id        ULID,
    response_status   SMALLINT,
    response_body     JSONB,
    state             SMALLINT    NOT NULL,   -- in_flight | completed
    source_ip         INET,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at      TIMESTAMPTZ,
    PRIMARY KEY (merchant_id, idempotency_key)
);
CREATE INDEX idx_idem_expiry ON idempotency_keys (created_at);   -- for TTL sweeps

CREATE TABLE outbox_events (
    outbox_id       BIGSERIAL   PRIMARY KEY,
    event_id        ULID        NOT NULL UNIQUE,
    aggregate_type  VARCHAR(32) NOT NULL,
    aggregate_id    ULID        NOT NULL,
    event_type      VARCHAR(64) NOT NULL,
    payload         JSONB       NOT NULL,
    headers         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at    TIMESTAMPTZ,
    attempts        SMALLINT    NOT NULL DEFAULT 0,
    next_attempt_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_outbox_unpublished
    ON outbox_events (next_attempt_at, outbox_id)
    WHERE published_at IS NULL;
```

Candidate: The partial index on unpublished rows is the important one. The relay's hot query touches only the small unpublished set, not the 700 million published rows, so relay latency stays flat as the table grows. I also partition this table monthly by `created_at` and drop old partitions after retention, because a mutable table is the enemy of an append-only design.

Interviewer: How do you shard this?

Candidate: This is the decision I want to be most careful about, so let me lay out the tension and then my answer.

Candidate: The payment aggregate is the unit of transaction: authorized, captured, and refunded counters must move atomically. So **all rows for one payment must live on one shard**. The choice is what the shard key is, and the three candidates are payment id, merchant id, and payer id.

Candidate: Sharding by payment id is perfect for the aggregate and terrible for every query. A customer's payment list would be a cross-shard scatter-gather across 34 shards. Rejected.

Candidate: Sharding by merchant id is great for merchant reporting, which is the operational hot path for reconciliation and settlement. But a single top merchant at 9 percent of peak traffic is 126 TPS on one shard, and a customer with purchases from twenty merchants fans out. I can fix the hot merchant, but I would rather not have to.

Candidate: Sharding by payer id co-locates the entire payment history and the wallet balance for one customer, which is the dominant read path, makes the wallet transfer a single-shard transaction, and spreads load naturally. Cost: merchant-scoped queries fan out, and a merchant with a million customers is a scatter.

Candidate: I shard by `payer_id`, using consistent hashing with 256 virtual nodes per server. And I solve the merchant-report problem honestly: merchant-scoped reporting is an async read-model concern served from the data warehouse, not a query on the OLTP database. Reconciliation is per-PSP-settlement-batch, not per-merchant-scan.

Candidate: And the hot merchant: at 9 percent of 1,400 TPS, if the shard key is payer, the top merchant's payments spread across 34 shards because its customers are distinct. The hot merchant is only a hot *partition* if I had sharded by merchant. So choosing payer as the key happens to dissolve the problem. I would still add per-merchant rate limits so one merchant cannot consume a disproportionate share of the platform, which is a fairness and cost control, not a correctness fix.

Interviewer: And the ledger? It is sharded the same way?

Candidate: The ledger is the subtle one, so let me be careful. A journal's postings touch two accounts, and those accounts may belong to different owners — a customer liability and a merchant payable. If I shard `ledger_entries` by `account_id`, then a single journal's postings land on two different shards, and I cannot post them atomically without a distributed transaction. That is the trap.

Candidate: Two options. Option one, shard by `journal_id` so all postings of one journal are co-located, which makes the journal atomic on a single shard. The cost is that per-account balance queries now fan out, and balances are the hottest read in the system.

Candidate: Option two, shard by `account_id` and accept that a journal spanning two accounts is two single-shard transactions, which is unacceptable because a half-posted journal is exactly the money-invention bug the invariant check exists to catch.

Candidate: I take **option one: the journal is the shard unit, keyed by journal_id**, and balances are never computed on the read path. They come from an async projection that consumes `ledger.entry.posted` events and maintains per-account balances, and any account's postings, even if they are scattered across journal shards, are all fully represented in the projection because the projection consumes the global stream. Then per-account balance reads are O(1) from a materialized store, and the journal atomicity guarantee is preserved.

Candidate: The residual risk is a projection that lags or a partition that loses a posting. The projection lag I handle with a watermark: the balance response carries an `as_of_sequence` so a client can tell it is looking at a slightly stale balance, and the nightly job rebuilds every balance from the journal and diffs. The lost posting I handle because the journal is the source of truth and the projection is rebuildable — I can always recompute.

Candidate: And the nightly rebuild is itself a consistency test. If the rebuilt balance differs from the projected balance, the projection has a bug. If the invariant query returns non-zero, the journal has a bug. Those are two different alerts and I want both.

---

## Phase 11 — Caching

Interviewer: Where do you cache, and what is the failure mode you are accepting?

Candidate: Three caches, and I will say what each one buys and what it costs.

Candidate: One, **wallet and account balances in Redis**, all 50 million accounts, about 15 GB. This is the highest-value cache in the system because every checkout reads a balance, the write rate is low relative to the read rate, and a stale balance by a few hundred milliseconds is harmless for display. Invalidation is event-driven: the projection writes the new balance, then deletes or sets the Redis key with a short TTL of 60 seconds. I use write-through with a short TTL rather than write-behind, because a lost write-behind buffer is an incorrect balance and I would rather pay a synchronous write on the cold path.

Candidate: Two, **payment status for in-flight payments**, TTL of 60 seconds. When a payment enters `processing` I set the key. On every transition I overwrite it. A poll reads it. This makes the client polling loop cheap. I deliberately do not cache terminal states, because the number of them is unbounded and they are immutable anyway — the database answer is already fast and the read model covers listing.

Candidate: Three, **merchant-scoped read model for statements and dashboards** in a read-optimized store, which is really the CQRS read side rather than a cache.

Candidate: Cache stampede, because you will ask. The scenario: the top merchant's status endpoint has a 60 second TTL on a very hot key, it expires, and 20,000 concurrent requests all miss and all hit the database. Mitigations, in order of value: single-flight or request coalescing so only one request actually loads while the rest wait on a promise; a soft TTL with a background refresh so the key is never actually absent; TTL jitter so keys do not all expire together; a negative cache for not-found to stop penetration; and a stale-while-revalidate read tier so a slightly stale value is served while one request refreshes.

Interviewer: Cache stampede on the balance projection specifically. What if the projection consumer crashes and the cache goes cold at once?

Candidate: The balance cache is rebuilt by the projection, not lazily per key, so a consumer crash leaves the Redis keys intact with a stale value plus a lagging watermark rather than an empty cache. The dangerous case is Redis itself failing over and losing a shard. Then reads fall through to the read model at maybe 40 milliseconds instead of 2, which is a latency increase, not an outage, because the read model is always there. I do not lose correctness, I lose freshness, and that is the correct trade.

Interviewer: Negative caching and cache penetration. Someone scripts a million lookups of non-existent payment ids.

Candidate: Rate limit at the gateway by merchant and by IP, and the balance and payment-id caches are populated only by real reads so unknown keys would otherwise all miss. I add a bloom filter in front of the payment-id existence check, 15 GB becomes 200 MB, false-positive rate under one percent, and a false positive is a wasted read, never a wrong answer.

---

## Phase 12 — Messaging

Interviewer: Messaging. What is on the bus and what are the delivery semantics?

Candidate: The bus is a partitioned, replayable log — Kafka if I have the operational appetite, a managed queue like SQS or Pub/Sub if I want less. I want replayability because reconciliation and analytics both need to reprocess history, and I want ordering only within a partition, keyed by payment id, so per-payment events are ordered and cross-payment events are not.

Candidate: Topics and their keys:

```text
  payment.events        key = payment_id    120M msg/day
  ledger.events         key = journal_id     30M msg/day
  fraud.commands        key = payment_id     120M msg/day
  fraud.decisions       key = payment_id     120M msg/day
  webhook.outbound      key = merchant_id    66M msg/day
  reconciliation.tasks  key = settlement_date
  projection.rebalance  key = shard_id       (operational)
```

Candidate: Delivery semantics: at-least-once end to end. The producer side gets at-least-once from the outbox relay. On the consumer side every consumer is idempotent: it keeps a processed-event table or, better, uses a natural idempotency key in its own write so a duplicate is a no-op. For the balance projection, the update is `set balance = balance + delta` with the delta keyed by entry_id, and I maintain a seen-entry set so the same entry is never applied twice. That is the difference between at-least-once and exactly-once effect.

Candidate: What actually needs stronger guarantees? The payment state transition. A duplicate `payment.authorized` must not double-post the ledger. So the consumer's write is a compare-and-set on the payment state — `UPDATE payments SET status = 'authorized' WHERE payment_id = ? AND status = 'initiated'` — and the ledger postings are in the same transaction as that update, and the unique constraint on `(payment_id, sequence_no)` in `payment_events` plus a unique `journal_id` makes a second attempt fail loudly. The database, not my application logic, is the exactly-once mechanism.

Candidate: Ordering. I key by payment id, so all events for one payment land on one partition and are processed in order. I do not partition by payer, because that would couple unrelated payments and create a hot partition for a heavy user. I do not partition by amount or by timestamp, because a flash sale would then order events from different payments arbitrarily, which is fine, but a merchant-scoped partition would concentrate.

Candidate: What if a partition is hot? A single very high-volume merchant does not matter because I key by payment. A single very heavy *customer* — a bot hammering with 200 concurrent payments — would land on one partition. Mitigations: sub-partition by payment id suffix when a hot key is detected, or shard the consumer group and accept that per-payment ordering is preserved within a sub-partition. In practice I detect it with a per-key rate metric and I have a documented runbook.

Candidate: Poison messages. Every consumer has bounded retries with exponential backoff and jitter — attempts at 0, 1, 4, 16, 64, 256 seconds — and after 6 attempts the message goes to a per-topic DLQ with the original payload, the failure reason, the attempt count, and the trace id. The DLQ is a work queue with an owner, a dashboard, and an alert, not a graveyard. Replay is a first-class operation: a tool that takes a DLQ range and re-publishes it after the bug is fixed.

Study separately: [[delivery-and-retry|Delivery Semantics, Retries, and DLQ]]

Interviewer: Consumer lag.

Candidate: Lag is my single most important queue health metric, more important than throughput. If the balance projection lags by 5 minutes, users see wrong balances and start complaining; if it lags by 200 milliseconds, nobody notices. So I alert on lag seconds per consumer group, not on message rate. The response ladder: scale consumers horizontally, which works if the bottleneck is a lock or a single slow dependency; if the bottleneck is the downstream system, I add a read buffer in front of it; and if it is irrecoverable I shed the least important consumers first — analytics and notifications — and I reserve capacity for the ones on the money path. I order the consumer groups by business criticality so shedding is a deliberate, pre-authorized action rather than a panic.

---

## Phase 13 — Replication and Consistency

Interviewer: Replication setup, and then the replica-lag question.

Candidate: Ledger and payment tables: primary plus two synchronous replicas across three availability zones in one region, with a quorum of two for writes. Acknowledgement means two of three durable. That is my RPO of zero within the region.

Candidate: Across regions: asynchronous replica in a second region with an RPO measured in seconds, used for read-after-failover and for regional analytics, never for a correctness-critical read. And I keep a WORM archive in object storage as the disaster-recovery floor.

Candidate: Read routing: reads that must be authoritative go to the primary. Reads that are for display go to the read model, which is fed by the log, so they are as fresh as the log and never blocked by replication.

Interviewer: Replica lag. What breaks?

Candidate: Four concrete breakages, and I want to name them because "replica lag is bad" is not an answer.

Candidate: One, a read-after-write violation. The client creates a payment, gets the 202, and immediately GETs the payment from a lagging replica and sees nothing. Fix: the write path stamps the response with a token, and reads carry it; the router sends the read to the primary if the token is newer than the replica's replay position. I also return the authoritative state inline in the 202 response so the common case never reads at all.

Candidate: Two, an idempotency check that reads a stale replica. I claim a key, crash, the client retries, and a replica does not yet see the key, so I create a duplicate payment. This is the dangerous one. Fix: **the idempotency key lookup goes to the primary, always.** I do not cache idempotency responses and I do not read them from a replica. Same for the balance-affecting `captured_minor` comparison, which is inside the write transaction on the primary anyway.

Candidate: Three, a monotonic-ID assumption. If a replica serves a list ordered by a time-sortable id, and the replica is behind, the user sees the list with a gap. Fix: pagination cursors on `(created_at, payment_id)` plus a "you are viewing as of" watermark, so a stale read shows a consistent-but-slightly-old prefix rather than a hole.

Candidate: Four, failover to a replica that is behind. If I promote a replica that is 30 seconds behind, I have lost 30 seconds of acknowledged writes, or my RPO-zero claim is false. Fix: promote only a replica whose replay position is within the quorum's last committed position, and if none qualifies, promote the best available and run the outbox-to-PSP reconciliation on the missing window — since my outbox rows are durable on the quorum, the *intent* is recoverable even if the last commit is not, and the resolver resolves it.

Interviewer: That last point is subtle and I like it. Push on it — is the outbox really enough?

Candidate: It is enough to recover *intent*, not to recover *truth*. The payment row for the last few seconds might be missing on the promoted replica while the outbox row is present. When the relay publishes, the command worker calls the PSP with the persisted PSP idempotency key; the PSP says "I already have that key, here is the result", and I re-apply the state transition. So the invariant that saves me is not the outbox, it is that **the PSP's idempotency key store is the authority on external effects**, and my outbox plus idempotency keys make the system replayable. I have to be honest that this narrows the window but does not make RPO literally zero in the cross-region asynchronous case.

---

## Phase 14 — Availability and Fault Tolerance

Interviewer: Give me the failure modes and what the user sees.

Candidate: Seven, and I will say what the user sees for each.

Candidate: One, **PSP is down or slow**. I put a circuit breaker in the PSP adapter, per PSP. Above a failure or latency threshold the breaker opens and the adapter fails fast. Payments do not go to `failed`, they stay in `initiated` and I return 202 with `status: pending_provider`. I queue the commands and let the command worker retry with backoff. User sees: accepted, pending, resolving on their own. The alternative — failing the payment — is strictly worse and creates support load and duplicate attempts.

Candidate: Two, **PSP is slow but not down**, p99 at 10 seconds. My 8-second budget trips, payments accumulate in `initiated`, the command worker backlog grows. Backpressure: I shed new payment creation past a threshold with a 429 and `Retry-After`, because unbounded queueing just converts a latency problem into a memory and connection problem. Overload protection: a bounded queue per PSP with a drop-oldest-then-reject policy, and a circuit breaker on the *queue depth* as well as on error rate.

Candidate: Three, **my primary database dies**. The orchestrator stops accepting writes — fail closed, which is the correct choice for money; the correct choice for a search index is fail open. Clients get a 503 with a retry hint. The replica is promoted within about 30 seconds, the orchestrator reconnects, and the in-flight command worker retries pick up. Any payment stuck in `initiated` or `unknown` older than the failover timestamp is swept by the reconciler, which asks the PSP for the truth. User sees: a 30 second outage on new payments, and no incorrect payments.

Candidate: Four, **the broker is down**. The outbox keeps accumulating, the command workers drain and then stall, and payments sit in `initiated`. Nothing is lost because the outbox is durable in the database. I alert on oldest-unpublished-outbox-age. User sees: accepts, then eventually resolves once the broker recovers. Outage duration equals broker recovery, and the blast radius is the write path only, because reads do not depend on the broker.

Candidate: Five, **replication lag explodes**, say 10 minutes. Balances and dashboards show stale data, the read model is stale, and the idempotency path is unaffected because it is on the primary. If lag on the ledger exceeds a hard threshold, I stop the settlement and reconciliation jobs, because reconciling against a stale replica produces false mismatches and I would rather delay than mislead.

Candidate: Six, **a bad deploy doubles charges**. The single most important protection is that no deploy can change money semantics without an audit. Guards: the database constraints hold regardless of code, the ledger invariant check runs continuously, and a canary processes a small percentage of traffic against a shadow ledger that is compared entry by entry against the production ledger. A divergence pages before promotion.

Candidate: Seven, **the reconciliation job itself is wrong** and reports thousands of false breaks. I make breaks a first-class table with a triage state, not a page. Each break has a reason code, an owner, and a resolution path, and an operator can mark it known. The break rate is a monitored business metric.

---

## Phase 15 — Bottlenecks and Scaling

Interviewer: Where are the bottlenecks, in order?

Candidate: Ranked by how close I think they are to failure.

Candidate: One, the **PSP**. It is a hard external rate limit, probably a few hundred requests per second per credential, and it degrades before my own hardware does. Everything about my scaling strategy is downstream of not exceeding it. Mitigations: credential pool rotation, adaptive concurrency limits from observed latency, and pacing to the PSP's stated limits rather than to my capacity.

Candidate: Two, the **payment aggregate's shard contention**. All transitions for one payment serialize on one row and one shard. At 1,400 peak TPS spread over 34 shards that is 41 TPS per shard, trivial. The risk is skew from a hot customer, handled by splitting a hot payer across sub-shards.

Candidate: Three, **the ledger write path**. 8,300 appends per second average, 33,000 at peak, mostly appends so B-tree right-edge inserts, but the balance projection and the invariant check both read it. I keep the invariant check per-shard and streaming so it never scans globally on the hot path.

Candidate: Four, **the read model**, which is a fanout consumer and must keep up with 120 million events per day. It is a write-through projection with batched upserts, and it is the component I scale horizontally first.

Candidate: Five, **database connection pool exhaustion** under burst. A payment touch may open several connections across the orchestrator, projection, and invariant checker. I budget connections globally per instance, size the pool from measured per-request usage and the 8,000-connection server limit, and I use a separate pool for the hot write path so a slow read cannot starve a write. Pool sizing is a classic silent outage, so I alert on pool wait time, not just on pool utilization.

Study separately: [[database-connection-pooling|Database Connection Pooling]]

Interviewer: Traffic doubles overnight. What changes, in order?

Candidate: In order, cheapest to most expensive.

Candidate: One, autoscale the stateless tiers — API, orchestrator, projection, webhook dispatchers — on request rate with a scale-on lead time measured in minutes. This is free and I should already be running hot spare capacity.

Candidate: Two, add read replicas and shift display reads to the read model, which is already the plan. Read scaling is basically solved.

Candidate: Three, scale the message consumers, if lag allows. Lag is my gate, and I do not scale consumers past the point where the downstream lock becomes the bottleneck.

Candidate: Four, double the shard count. This is expensive: a reshard moves data and is a multi-hour operation, so I keep the virtual node count high enough that I have room to grow, and I reshard by adding shards and rebalancing asynchronously rather than a stop-the-world move.

Candidate: Five, the PSP rate limit, which I cannot scale. If doubling traffic doubles PSP calls, I am stuck. So the design must reduce PSP calls per payment: idempotency collapsing removes the 15 percent retry overhead, batching settlement and payout reduces per-transaction calls, and where a merchant is pre-funded I do not call the PSP at all.

Candidate: What I would *not* do is shard the ledger by account while payments are sharded by payer, because that reintroduces the cross-shard journal problem I designed away.

---

## Phase 16 — Trade-offs

Interviewer: Why not two-phase commit with the PSP?

Candidate: Four reasons, and the third one is the real one.

Candidate: One, the PSP does not offer XA, and no card or banking integration does. I cannot 2PC with a system I do not control.

Candidate: Two, even if it did, an XA coordinator holds a database connection and a lock for the entire duration of the transaction, which for an authorization with a 3-D Secure human challenge can be 45 seconds. At 1,400 TPS that is an untenable number of locked connections.

Candidate: Three, a distributed transaction coordinator is a single point of failure and a performance bottleneck, and its recovery log becomes one of the most critical and least tested pieces of infrastructure in the company.

Candidate: Four, 2PC also blocks, which is a coin flip rather than a correct outcome. The alternative is a **saga**: each step is its own local transaction and every step has a compensating action. Void the authorization to reverse it, refund to reverse a capture, re-credit the wallet to reverse a debit. The trade-off I accept is **compensating actions instead of atomic rollback**, so intermediate states are visible to users and the business logic for compensation is real code that must be tested. I mitigate with a decision deadline: a payment that has not reached a terminal state within 15 minutes is automatically voided or refunded, so compensation always happens.

Interviewer: Why not a distributed lock per customer to serialize charges?

Candidate: Three reasons.

Candidate: One, it serializes on the PSP round trip. Two concurrent payments for the same customer would each wait the full PSP latency, and I would be trading my highest-value latency path for a guarantee I can get more cheaply.

Candidate: Two, and this is the important one, a lock is a **lease with a timeout**, and if the process holding it dies mid-PSP-call the lock expires while the PSP call is still in flight. The next request takes the lock and charges again. So the lock does not actually prevent the double charge; it only makes it less likely. I would be building a false guarantee.

Candidate: Three, locks and network partitions produce the two generals problem, and money is the one domain where I want the guarantee to be structural, not probabilistic. The structural guarantee is: a unique constraint on the idempotency key, a compare-and-set on the payment state, and the PSP's own idempotency key. All three are enforced by a system that does not fail open.

Candidate: Where I *do* use a distributed lock is narrow and non-financial: serializing a per-account projection rebuild, and ensuring one nightly settlement run per merchant. Both are operations, not money movements, and both are safe to retry.

Study separately: [[distributed-locks|Distributed Locks]]

Interviewer: Why is the ledger not just a column on the payment row?

Candidate: Four reasons.

Candidate: One, a payment and a balance are different grains. A payment is one transaction; a balance is the sum of every transaction touching an account, and a refund touches accounts that the original payment never touched.

Candidate: Two, an append-only journal is provably correct and cheap to audit; a mutable balance is a summary that can be silently corrupted and cannot be reconciled against history.

Candidate: Three, double-entry is what makes the system auditable at all. It gives me a mathematical invariant — postings sum to zero — that I can assert continuously and that a regulator or an auditor can verify independently. Without it, "why is this merchant's balance 4,000" has no answer.

Candidate: Four, it makes reversal correct. A refund is a new journal, not an update. History is never rewritten, so the audit trail, the dispute process, and the tax reporting all work without special cases.

Interviewer: Why asynchronous fraud instead of synchronous?

Candidate: Because synchronous fraud is a 100 to 400 millisecond tax on every payment and it puts my most fragile dependency, a model plus a feature store, on the critical path of money movement. If the risk service is down and I fail open, I take losses; if I fail closed, I stop taking payments, which is a worse business outcome than some fraud.

Candidate: My design authorizes first, which reserves funds, then scores with a 300 millisecond budget, and the decision arrives asynchronously with a **decision deadline**. If no decision arrives in time, the default is the safer one for the platform and the more expensive one for revenue: hold in review, do not capture. If the decision is block, void the authorization, and if the void fails, refund.

Candidate: The honest trade-off: this lets a small amount of fraud be authorized before it is caught, and my loss is the refund cost plus the fee, not the principal. I accept that. I also make the asynchronous path better than the synchronous one in one respect: the risk model can use features that are not available in 300 milliseconds, like device reputation over 30 days and graph features on the payer.

---

## Phase 17 — Failure Scenarios

Interviewer: Rapid-fire. What happens in each of these, and what does the user see?

Candidate: Ambiguous capture timeout. User sees pending, and I do not retry the capture. The payment enters `unknown`, the resolver polls the PSP status with the original idempotency key within 30 seconds, and the outcome lands. The client sees `captured` or `failed` typically within 60 seconds, and if not, the payment is flagged and a human reconciles it.

Candidate: Duplicate PSP webhook, five copies of `payment.succeeded`. The ingress dedupes on PSP event id with a unique constraint; four are rejected as duplicates. Even without dedupe, the state machine refuses the transition from `captured` to `captured` and the ledger posting is prevented by a unique journal id.

Candidate: Out-of-order PSP webhooks — `succeeded` then `pending`. The state machine refuses the regression and logs a reconciliation-later case. My transition table has no edge from `captured` to `processing`, so the payload is discarded as stale.

Candidate: PSP webhook replayed from six months ago for a completed payment. Same answer: no valid transition exists, discarded, and I keep a 90-day dedupe window with a 7-year audit record so I can still tell "duplicate" from "forged".

Candidate: My service crashes after committing the outbox row and before publishing. Nothing is lost. The relay finds the unpublished row on its next scan and publishes it.

Candidate: The relay publishes and crashes before marking `published_at`. The event is published twice. Consumers are idempotent, so it is a no-op the second time. This is why I have `published_at` and why I accept at-least-once.

Candidate: Ledger imbalance. The invariant query returns one row. Money was invented or destroyed. P1. I freeze settlement, identify the journal, and the only correct repair is a compensating journal, because editing history is forbidden. I would also check whether a partial shard restore or a bad migration caused it, since a bulk operation that bypasses the posting path is the most likely culprit.

Candidate: The projection drifts from the journal — balances are wrong by 2,000 on one account. The nightly rebuild diffs and pages. Fix is to rebuild the projection from the journal, which is exactly why the journal is the source of truth. User impact in the meantime: a wrong displayed balance, potentially causing a false decline, which is a real customer-facing harm, so the check is high priority.

Candidate: The settlement job runs twice because an operator re-triggered it. Settlement is protected by a per-merchant-per-cycle distributed lock plus a unique constraint on `(merchant_id, settlement_cycle)` in `settlement_batches`. Both, belt and braces, because a duplicate payout is a real loss.

Candidate: A merchant's webhook endpoint starts returning 500. The dispatcher backs off exponentially, then circuit-breaks that endpoint and keeps the events in a per-endpoint queue. Other merchants are unaffected because the breaker and the queue are per endpoint. I alert the merchant and expose a replay in the portal.

Candidate: A hot merchant at 9 percent of volume starts getting 429s. The gateway's per-merchant rate limiter is doing its job. I contact the merchant, they raise their limit or I shard their traffic, and meanwhile no other merchant is affected.

Candidate: Clock skew causes two events for the same payment to be ordered incorrectly by timestamp. This is why I never order by wall-clock; I order by `sequence_no` per payment, allocated under the row lock in the same transaction. Wall-clock is for humans and reporting, not for correctness.

---

## Phase 18 — Final Architecture Summary

Interviewer: Ninety seconds. Give me the whole thing.

Candidate: One sentence per layer.

Candidate: The client holds no card data; it tokenizes through the PSP and calls our API with an idempotency key. Our edge authenticates the merchant, rate-limits, and routes.

Candidate: The Payment API and Orchestrator own the payment state machine, which has eight states from `initiated` to `settled` plus `unknown` for ambiguous PSP outcomes. They enforce a single aggregate invariant: refunds never exceed captures, enforced by `CHECK` constraints as well as code.

Candidate: The ledger is a double-entry, append-only journal in the same physical database and the same transaction as the payment row, sharded by journal so every journal's postings are atomic on one shard. Balances are an async projection and are never computed on the read path. An invariant query asserts that postings sum to zero, continuously.

Candidate: The outbox is written in the same transaction as every state change, so the database and the broker can never disagree. Delivery is at-least-once and every consumer is idempotent, which gives effectively-once business effect, which is the only exactly-once that actually exists.

Candidate: Idempotency is enforced at three layers — a unique key in my own store, a persisted PSP idempotency key generated before the outbound call, and unique journal ids — so a crash at any point produces a retry, never a duplicate charge.

Candidate: Fraud is asynchronous with a decision deadline, because the money path should not depend on a model server, and it reverses via void-or-refund compensation.

Candidate: Merchants are notified by an outbox-driven webhook dispatcher with per-endpoint retry, backoff, DLQ, and a delivery log the merchant can replay from.

Candidate: Reconciliation runs daily against the PSP statement file as a three-way match on reference, amount, and fee, and it is the backstop that finds everything the real-time paths missed.

Candidate: Scale is 350 average and 1,400 peak payment writes per second, 3,100 average reads, 8,300 average internal row operations, roughly 55 TB of new data per year at 150 GB per day with seven-year retention, tiered hot for 90 days.

Candidate: The whole thing is sharded by payer id with 256 virtual nodes, multi-AZ with quorum writes for RPO zero inside a region, warm standby across regions, and a WORM archive as the floor.

Candidate: The three decisions I would defend hardest are: ledger and payment state in one transaction to avoid a distributed transaction entirely; sharding the journal rather than the account so journals stay atomic; and treating the PSP's idempotency key store as the authority on external effects, which is what makes the whole replay story work.

---

## Concepts To Study Separately

Candidate: I want to be explicit about where my own understanding is thin, rather than improvising confidently in the interview.
Study separately: [[outbox-pattern|Outbox Pattern]]
Study separately: [[exactly-once-effect|Exactly-Once Semantics]]
Study separately: [[distributed-transactions|Distributed Transactions]]
Study separately: [[saga-and-strangler|Saga Pattern and Compensating Actions]]
Study separately: [[idempotent-consumer|Idempotent Consumers]]
Study separately: [[replication-lag|Replication Lag and Read Routing]]
Study separately: [[soft-delete-audit-tables|Append-Only Audit Trails]]
Study separately: [[distributed-id-generation|Distributed ID Generation]]
Study separately: [[idempotency|Idempotency and Idempotency Keys]]
Study separately: [[idempotent-retry|Idempotent Retry]]
Study separately: [[oltp-vs-olap|OLTP vs OLAP Separation]]
Study separately: [[storage-tiering|Storage Tiering]]
