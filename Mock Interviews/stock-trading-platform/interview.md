---
title: Stock Trading Platform - Full Mock Interview Transcript
status: active
tags: [hld, mock, stock-trading-platform, interview]
---

# Stock Trading Platform - Full Mock Interview Transcript

A complete 45-minute session, annotated with the structure a strong answer follows. Read it twice: once for flow, once for the numbers.

**Target:** senior backend / trading infrastructure engineer
**Rubric:** the five phases from [[06-hld-interview-checklist|HLD Interview Checklist]]

---

## Phase 1: Requirements Clarification (0:00 - 7:00)

**Interviewer:** Today's problem is a retail brokerage backend. Users watch live quotes and place, modify, and cancel stock orders. Design the system.

**Candidate:** Before anything else I need to resolve what I am actually building, because "trading platform" could mean an exchange or a broker, and the difference is architectural, not cosmetic.

Am I the exchange, meaning I own the order book and run a matching engine, or am I a broker that routes to existing exchanges and consumes a market data feed? And second, do I need Level 1 data, meaning last trade, bid, and ask, or Level 2, meaning full depth of book?

**Interviewer:** You are a broker. You consume a consolidated market data feed from exchanges and you route orders out. Level 1 only for retail clients.

**Candidate:** That simplifies the read path enormously and it means my matching engine is about internalizing and routing rather than running a full exchange matching stack. Let me keep going.

**Interviewer:** Keep going.

**Candidate:** What is the actual latency target and where is it measured: to the client, to my matching engine, or to the exchange fill? And what is the order of the guarantee I need: is "no double execution" about not creating a duplicate order at intake, or about never over-filling a single order? Those are two different problems with two different solutions.

**Interviewer:** Measure to the client acknowledgement. No double execution means both, but the one that would be a legal incident is over-filling a single order. Order ids from clients must be idempotent though, a retrying client must not create two orders.

**Candidate:** Noted. So intake idempotency plus per-order fill caps that are enforced in one place. Next: recovery. If the whole cluster dies mid-session, can I replay truth from the exchange, or is my own order event log the source of truth?

**Interviewer:** Your log is the source of truth. The exchange will not give you a free replay of everything you routed.

**Candidate:** Then the log is mandatory and I have to design for deterministic replay. Next: availability. Is 99.99 percent single region with a maintenance window acceptable, or do they need active-active?

**Interviewer:** Single region is fine. 99.99 percent. But I want you to think about what degraded mode looks like, because "we are down" is not an acceptable product for an hours-long session.

**Candidate:** Agreed. If I cannot trade, I should at least still quote, and say so clearly. And I will need an explicit halt path.

**Interviewer:** Last questions from me. What is the order rate distribution across symbols, and is a single symbol ever going to be a large fraction of my load?

**Candidate:** That is the question I most wanted you to ask.

**Interviewer:** We see 5,000 symbols. Roughly 60 percent of order flow concentrates in the top 100 names. There are days when one name takes 5 percent of all orders on its own. And yes, the market-open burst is roughly 25x the intraday average.

**Candidate:** Then per-symbol partitioning is not just a scaling tactic, it is the correctness model, and I will show you why.

**Interviewer:** Fine. Start from requirements.

**Candidate:** Functional requirements:

- Ingest a live exchange feed and normalize it into a canonical quote
- Fan out quotes to hundreds of thousands of concurrent subscribers watching shared symbols
- Maintain an order book per symbol, match on strict price-time priority
- Order lifecycle: place, modify, cancel, with explicit state transitions
- Pre-trade risk: buying power, price collars against fat fingers, max order size, self-trade prevention
- Durable, replayable audit of every order and every trade
- Safe retries: a client timeout must never create a duplicate order or a double fill
- Survive loss of a node or a broker reconnect without losing or duplicating a trade

Non-functional:

- Order acknowledgement p99 under 20 ms end to end
- Quote tick to client under 100 ms p99
- Availability 99.99 percent, with a defined degraded mode
- Per-symbol strict ordering, total order within a symbol, no ordering requirement across symbols
- Zero over-fill, enforced in exactly one place
- Full session reconstructable from the log for audit

**Interviewer:** One thing I want you to be explicit about: is 99.99 percent really the right number, given that being down at the open is the worst possible failure?

**Candidate:** No, and I will push back on my own requirement. Ninety-nine ninety-nine allows about 52 minutes of downtime per year. Spread over 250 trading days that is roughly 12 minutes of trading lost, and if it lands on a volatile day, that is the day your regulators and your customers care about. The real requirement is not a percentage, it is that no single failure can take down more than a small set of symbols, and that the blast radius of a bad deploy is bounded. I will design for availability of individual symbols rather than availability of the platform, and treat the percentage as a consequence.

**Interviewer:** Good. Estimate.

---

## Phase 2: Scale Estimation (7:00 - 12:00)

**Candidate:** Ingest first. 5,000 symbols, a consolidated feed. During the 6.5 hour session, US equities tick maybe 10 to 30 times per second on active names, 1 to 2 on quiet ones. Call it an average of 10 ticks per second per symbol, 50,000 quote events per second ingest. That is a small number. Market data ingest is not the hard part of this problem.

Orders. 3 million DAU, say 12 orders per user per day, 36 million orders per day. Concentrated in a 6.5 hour session, 23,400 seconds, so 1,540 orders per second average during the session. Peak: 25 percent of the day's orders in the first five minutes, so 9 million orders over 300 seconds, 30,000 orders per second at the open.

**Interviewer:** Can a matching engine handle 30,000 orders per second?

**Candidate:** Easily, and this is the key insight people get wrong. A single-threaded in-memory engine matching a price-time priority queue with a sorted price level structure does on the order of a million to several million orders per second per core. My peak of 30,000 per second across 5,000 symbols is 6 orders per second per symbol on average. Even a name taking 5 percent of all orders is 1,500 per second, which is well under 1 percent of one core's capability.

So the matching engine is not the bottleneck and I should not spend the interview optimizing it. The bottlenecks are intake, fanout, and durability. I want to be explicit about that, because "we need horizontal scaling of the matching engine" is the wrong conclusion people reach.

**Interviewer:** So what is the bottleneck?

**Candidate:** Fanout, by a wide margin. Let me size it. 3 million DAU, call it 40 percent watching quotes concurrently, so 1.2 million concurrent connections. Each watches about 10 symbols, so 12 million symbol subscriptions across 5,000 symbols.

If I delivered every tick to every subscriber uncapped, it is 50,000 events per second times average subscribers per symbol. Average is 2,400, so 120 million messages per second. At 80 bytes each that is 9.6 GB per second, 77 Gbps, and 120 million message sends per second is not a throughput any gateway fleet wants.

**Interviewer:** So what do you do?

**Candidate:** Coalesce and throttle, and this is a product-level decision I should have raised in requirements. A retail user cannot perceive more than about 10 to 20 updates per second on a symbol, and above that it is wasted battery, bandwidth, and money. So the fanout service coalesces ticks per subscriber and delivers at most 20 updates per second per subscription, batched into binary frames.

1.2 million connections times 20 messages per second is 24 million messages per second, 1.9 GB per second, about 15 Gbps, and 1.2 million frames per second with 20 quotes batched per frame. That is a real, deployable number, and it is 5 times cheaper than the uncapped version for a perception difference of zero.

**Interviewer:** Storage.

**Candidate:** Order events: 36 million orders per day, each producing a handful of events, accept, accept, fill, cancel. Call it 3 events per order, 400 bytes each, so 43 GB per day. Trades, 11 million per day at 200 bytes, 2 GB per day. Call it 45 GB per day, 16 TB per year.

That is not much data. The interesting property is not volume, it is that it must be replayable and ordered, and that retention is split. Seven days hot in the log for replay and operational queries, then compacted to Parquet in object storage for analytics and regulatory history, where it is effectively retained forever at pennies per GB.

**Interviewer:** Latency budget.

**Candidate:** Let me build a p99 budget of 20 ms down to the components.

Edge and TLS termination, 1 ms. Order service receives, authenticates, and looks up the account, 2 ms. Idempotency check, a unique constraint insert, 1 ms. Risk checks, buying power and collars, 1 ms. Routing to the symbol shard, 0.5 ms. Matching engine, 5 to 50 microseconds, and I will not budget more. Local WAL append plus replicated ack, 2 to 5 ms, and this is the single largest non-network item. Client notification over the existing WebSocket, 1 ms. Total p99 about 10 to 12 ms, with the 20 ms target as headroom.

**Interviewer:** API design.

**Candidate:** Three surfaces. The order API, the quote stream, and the internal query API.

```text
POST   /v1/orders                    place an order
PATCH  /v1/orders/{clientOrderId}    modify (qty down, price down, never up on qty)
DELETE /v1/orders/{clientOrderId}    cancel
GET    /v1/orders/{clientOrderId}    status, authoritative
GET    /v1/orders?symbol=&day=       blotter, day trades and positions
GET    /v1/quotes?symbols=AAPL,MSFT  REST snapshot, for cold start
WS     /v1/stream                    live quote + order update stream
```

The place-order contract, and the idempotency key is the load-bearing part:

```json
POST /v1/orders
Idempotency-Key: 7f3c9a21-...
{
  "clientOrderId": "android-8842-000000173",
  "symbol": "AAPL",
  "side": "BUY",
  "type": "LIMIT",
  "quantity": 100,
  "limitPrice": "231.45",
  "timeInForce": "DAY",
  "accountId": "acct_55120"
}
-> 201
{
  "clientOrderId": "android-8842-000000173",
  "orderId": "ord_9f3a71c",
  "status": "ACCEPTED",
  "acceptedAtMicros": 661234567890123,
  "streamSeq": 482011993
}
```

Three things I built into that. The `Idempotency-Key` header is separate from `clientOrderId` so a client can retry a whole logical request; `clientOrderId` is the business identity. The response echoes both so the client never has to guess. And `streamSeq` is the position of this order in the symbol's total order, which is how the client reconciles asynchronous fill notifications against the acknowledgement.

Every non-2xx has a machine-readable code, not prose, because the client has to branch on it: `PRICE_COLLAR_VIOLATION`, `INSUFFICIENT_BUYING_POWER`, `SELF_TRADE_BLOCKED`, `SYMBOL_HALTED`, `SHARD_UNAVAILABLE`, and the last one matters because a client that cannot place an order needs to know whether to retry or to surface an error.

**Interviewer:** Architecture.

---

## Phase 3: High-Level Architecture (12:00 - 19:00)

**Candidate:** Here it is. Two planes: the quote plane, which is read-heavy and lossy-tolerant, and the order plane, which is write-heavy and correctness-critical. Keeping them separate is the main structural decision.

```text
 EXCHANGE FEEDS
      |
      v
 +------------------+     +-------------------+
 | Feed Handlers    | --> | Market Data       |
 | per exchange, HA |     | Normalizer        |
 +------------------+     +---------+---------+
                                    | canonical quote
                                    v
                          +---------+---------+
                          | quotes-by-symbol   |  (Kafka, key = symbol)
                          | 256 partitions     |
                          +----+------------+---+
                               |            |
                    +----------v---+   +----v---------------+
                    | Quote Engine |   | Book Snapshotter    |
                    | per-symbol   |   | every 1M events     |
                    | partition    |   +----+-----------------+
                    +------+-------+        |
                           |                v
                           |        +-------+-----------------+
                           |        | Snapshots (book state)  |
                           |        | compaction topic       |
                           |        +-------------------------+
                           v
                    +------+-------------------------------+
                    |        Quote Fanout Service            |
                    |  subscribes quotes, maintains        |
                    |  per-symbol subscriber sets,         |
                    |  coalesces to <=20 msg/s/conn        |
                    +-------------------+-------------------+
                                        | binary frames
            +------------------+--------+---------+------------------+
            v                  v                  v                  v
     +-------------+    +-------------+    +-------------+    +-------------+
     | WS Gateway  |    | WS Gateway  |    | WS Gateway  |    | WS Gateway  |
     | 30K conns   |    | 30K conns   |    | 30K conns   |    | 30K conns   |
     +-------------+    +-------------+    +-------------+    +-------------+
            40 gateways total, 1.2M concurrent connections


 ORDER PLANE

  client
    |
    v
 +-------------------+
 | Order Service     |  auth, validate, IDEMPOTENCY KEY, risk
 | stateless, N pods |
 +---------+---------+
           |  routed by hash(symbol)
           v
 +---------+-------------------------------+
 |  Matching Engine  (per-symbol shard)   |
 |  single-threaded, in-memory            |
 |  price-time priority                   |
 |  local WAL -> replicated               |
 +---------+-------------------------------+
           |  every state change, ordered
           v
 +-----------------------------------------+
 | orders-by-symbol (Kafka, key = symbol)  |
 | RF=3, min.insync.replicas=2, acks=all  |
 +----+-----------------------+------------+
      |                       |
      v                       v
 +-----------+      +--------------------+
 | Client    |      | Persistence        |
 | notifier  |      | Kafka -> Parquet   |
 | (order    |      | (object storage)   |
 |  updates) |      +--------------------+
 +-----------+
      |
      v
 +---------------------------+
 | Accounts / Positions      |
 | sharded Postgres, RF=3    |
 | ledger of record           |
 +---------------------------+
```

**Interviewer:** The quote plane and the order plane both touch the symbol. What stops them from disagreeing?

**Candidate:** Nothing, and I should be honest that they can. The quote plane reads the exchange feed. The order plane knows my own resting orders. My displayed bid and ask are the exchange's, not my book's, because that is what a Level 1 client expects. They are genuinely different facts and they are allowed to differ by a few milliseconds. What I must never do is derive the displayed quote from my own book, because then a matching outage becomes a quoting outage. The quote plane has no dependency on the order plane at all, which is a deliberate availability choice.

**Interviewer:** Walk me through the order flow.

**Candidate:** Nine steps, and steps 2 and 3 are where the correctness lives.

**Candidate:** One, the client sends the order with an idempotency key to any Order Service pod.

**Candidate:** Two, before anything else, the Order Service inserts the idempotency key into a unique-constrained table. If the insert conflicts, it returns the stored prior response verbatim and does no further work. This is the entire no-duplicate-order mechanism and it is one database constraint.

**Candidate:** Three, pre-trade risk. Buying power for buys, shortable availability if relevant, price collar so no order goes more than a configurable percentage away from the reference price, maximum order size, max position, self-trade prevention by looking at whether the account already has a resting order on the other side. Risk is synchronous and on the critical path, because accepting an order I cannot fill is worse than rejecting it. 1 millisecond, and I will come back to making that fast.

**Candidate:** Four, the Order Service resolves the symbol to its shard from a routing table cached locally with a short TTL, and forwards the order. Note that a gateway or a broker reconnect never lands on the same pod, and the routing table is versioned so a stale entry fails fast rather than sending an order to a shard that no longer owns the symbol.

**Candidate:** Five, the matching engine for that symbol receives it on a single-threaded executor. This is the key structural decision: one symbol, one thread, one total order, zero coordination. No locks, no consensus on the hot path, and the order of operations is the sequence number.

**Candidate:** Six, the engine matches against the book. For each match it emits a fill with a `filledQuantity` that is bounded by the order's own remaining quantity, computed inside the same single-threaded critical section as the state mutation. The remaining quantity is maintained in the engine's own state, so over-fill is structurally impossible rather than guarded against. If the engine crashes mid-sequence, the shard fails over and the order is recovered from the log, never from memory.

**Candidate:** Seven, the engine appends every state change to its local WAL and the append is replicated with `acks=all` to a replication factor of 3 with `min.insync.replicas=2`. Only after that acknowledgement does the engine return to the order service. That 2 to 5 ms is the price of a guarantee that the order will be found after a crash.

**Candidate:** Eight, the engine publishes the resulting order events to the partitioned log, keyed by symbol, so the partition ordering matches the engine's execution ordering.

**Candidate:** Nine, two consumers act on that. The client notifier pushes the status change down the existing WebSocket. The persistence path writes to the analytics store and updates the account ledger. Neither is on the critical path and neither can fail the order.

**Interviewer:** Book recovery. The engine crashes. What exactly comes back?

**Candidate:** Snapshot plus delta, and this is what the [[event-sourcing-cqrs|Event Sourcing and CQRS]] pattern is for, applied narrowly.

The engine writes a full book snapshot every one million events to a compaction topic. On recovery, the shard loads the latest snapshot, then replays the partitioned log from that snapshot's sequence number forward. For 5,000 symbols with a 30,000 per second peak, that is a second or two of replay. The critical property is that replay is deterministic: given the same starting snapshot and the same event sequence, the engine must reach the same book. That means the matching function must have no clock reads, no randomness, and no floating point in its comparison path. Prices are fixed-point integers in integer tenths of a cent from the parse onward, never floats.

The event log is the source of truth. The book is a derived view that I can throw away and rebuild. And because the log is partitioned by symbol and replication factor 3, the failover replica has the same events and can rebuild the identical book, which is how a failover is fast rather than a cold start.

**Interviewer:** The snapshotter you put on the quote plane. Why does that need to exist too?

**Candidate:** Same reason, different service. The fanout service keeps the latest quote per symbol in memory. If it restarts it replays the quote topic. If the topic retention is one hour and it was down two hours, it cannot rebuild, so it must be able to reconstruct current state from a snapshot. A periodic quote snapshot lets recovery be seconds rather than bounded by retention, and it also lets me seed a brand new gateway without waiting for traffic to warm the cache.

---

## Phase 4: Deep Dive (19:00 - 30:00)

**Interviewer:** Fanout in detail. This is where I think the real cost is.

**Candidate:** Agreed. Let me be concrete about the design.

Subscriptions live in the Fanout Service, in memory, in a per-symbol subscriber set. A user watching 10 symbols is 10 entries. The data structure is symbol to a set of connection handles, and each handle points at a per-connection output buffer, not at a user.

Per connection I keep last-sent values per symbol and a timer. On a tick, the service marks the connection dirty for that symbol and does not write. A flusher runs at 50 ms intervals, and for each dirty connection it packs the accumulated values into one binary frame and writes it. That is the coalescing. It means a symbol ticking 30 times per second reaches the user at most 20 times per second, and a symbol ticking 2 times per second still arrives immediately because the connection is marked dirty and flushed at the next tick of the flusher.

Frame format, deliberately not JSON:

```text
+--------+----------+---------+-----------+-----------+
| ver(1) | type(1)  | count(2)| sym(4) x | q(4)+ts(8) |
+--------+----------+---------+-----------+-----------+
```

Symbol as a 4-byte interned integer, price as a 4-byte fixed-point integer, timestamp as 8 bytes. 14 bytes per quote. A 20-quote frame is about 300 bytes. Versus roughly 150 bytes of JSON per quote, that is a 7x reduction, and JSON parsing on the client at 24 million messages per second is a battery problem, not just a CPU problem.

**Interviewer:** 40 gateways, 30,000 connections each. How do you get a user's connection to the right gateway?

**Candidate:** I do not use sticky sessions as a correctness mechanism, because that would make a single gateway a single point of failure for every user it holds. The flow is: a connection lands on any gateway, the gateway authenticates it, then the gateway sends a subscribe message for its symbol set to the Fanout Service. The Fanout Service stores the mapping from connection handle to gateway, and the gateway address is part of the handle. If a gateway dies, the client's WebSocket drops, the client reconnects to any healthy gateway, and re-subscribes. The state lives in the Fanout Service keyed by connection, and a connection that stops heartbeating is swept.

**Interviewer:** So a gateway failure drops every connection on it.

**Candidate:** Yes, and I will not pretend otherwise. 30,000 users get a reconnect. That is a real number and it is why reconnect has to be fast and cheap: a fresh connection gets a REST snapshot in one batched call and then resumes streaming, so the visible gap is under a second. And it is why I would consider smaller gateway failure domains, more gateways with fewer connections each, if the reconnect burst itself becomes a problem, since the failure cost here is proportional to connections per gateway and nothing else.

**Interviewer:** Risk checks in one millisecond. Buying power depends on position, which depends on fills, which are in the log. Explain.

**Candidate:** Correct, and this is the part I would get wrong if I designed it naively. Buying power is derived state, and the naive design reads it from the accounts database, which puts a shard-level read on every order.

I keep it in memory, in the Account and Position Service, which owns a shard of accounts and subscribes to the fill stream. It maintains positions and buying power in memory and applies fills as they are published. The Order Service reads buying power from a local in-memory replica of the owning account shard, so the read is a memory access. The same event stream that updates positions also invalidates the Order Service's copy.

The important honesty here: this makes the buying power check an eventually-consistent read. It is at-least as fresh as the last published fill, which is a few milliseconds behind the matching engine. That is acceptable for fat-finger and margin-call prevention and it is not acceptable for regulatory margin enforcement, which would need a synchronous, serialized read per account. I would separate those two paths if the regulator required it. And I cap the staleness with a rule: if a fill stream is lagging beyond a threshold for an account, the order service fails closed and rejects new buys for that account rather than risking an over-buy. Failing closed on risk, open on availability, is the right asymmetry.

**Interviewer:** And self-trade prevention, which also needs to look at resting orders.

**Candidate:** Same shape. The per-symbol shard owns the book, so the Account and Position Service that shadows it also knows which of my client's resting orders are on each side. A self-trade check is a lookup in that shadow. But the check happens in the matching engine, in the same single-threaded critical section as the match, because a check-then-act outside the engine is a race. The engine has the account id on every resting order it holds, so the check is a linear scan of the opposite side at that price level, which is small, and it is exact because it is inside the serialized section. This is the reason the engine, not the risk service, is the authority on self-trade.

**Interviewer:** Now the hard one. Traffic doubles.

**Candidate:** Let me go component by component and tell you which ones break, because only two actually do.

Quote ingest, 50,000 to 100,000 events per second. Nothing, a single consumer group scales.

Order intake, 1,540 average to 3,000, peak 30,000 to 60,000 per second. The Order Service is stateless and shards by account, so this is linear. The idempotency inserts go from 1,540 to 3,000 per second of unique keys against the accounts shard set, and each is a 400-byte row, so 1.2 MB per second of writes, about 100 GB per day. That is real and it is the number I would watch, because the idempotency table is a write-amplified table that also has to be retained long enough to cover client retries, which is 24 hours, after which I purge.

Matching, 30,000 to 60,000 orders per second. Still under 1 percent of the engine's per-core capacity. Nothing. This is the part that surprises people.

Fanout, 24 million to 48 million messages per second, 1.9 to 3.8 GB per second. This doubles. I add Fanout shards and gateways proportionally, roughly 80 gateways and twice the fanout partition count. It is embarrassingly parallel and I have no coordination requirement across shards, so this is clean.

Log write throughput, 90 GB per day to 180 GB per day. A properly sized cluster handles this with room, and the thing I would actually check is whether my retention and partition count still support my replay window, because if I keep 7 days and I am at 12 partitions, per-partition throughput is what determines how fast a failover replica can catch up.

**Interviewer:** So the two that break are the fanout cost and the idempotency table. Anything else?

**Candidate:** One more, and it is the one that has bitten every exchange I know of. A single hot symbol. You said one name can take 5 percent of orders, but there are real days when it takes 20 percent, and in crypto-adjacent names it is worse. At 20 percent of a 60,000 per second peak, that is 12,000 orders per second on one symbol, and one symbol is one thread. That is now 1 to 2 percent of a core, still fine.

The real hot-key risk is not the engine, it is the fanout. One symbol with 200,000 subscribers is 200,000 connections that all dirty on the same tick, and my flusher now has a 200,000-entry hot set at 50 ms intervals. That is fine in aggregate, but it is a lock convoy and a single-threaded fanout shard becomes a hotspot. My mitigations, in order: the coalescing already reduces it 5x. I shard the subscriber set by connection id within the symbol, so the write-out work parallelizes even though the match work does not. And I put a per-symbol egress rate limit at the fanout layer as a circuit breaker, so one pathological symbol cannot consume the whole fanout fleet's bandwidth and starve the other 4,999.

**Interviewer:** Primary dies. Take the worst one: the Postgres cluster holding accounts and positions.

**Candidate:** Replicated, so a replica is promoted. Account and position data is shardable by `account_id`, and I do that, so this is per-shard, not global. The consequence is that a shard is unavailable for writes, so new orders for accounts in that shard are rejected with a retryable error. The client sees a clean 503 with a code, not a timeout.

Now the important part: what happens to orders that are already accepted and resting? They live in the order log, not in Postgres. The matching engine has them in memory and in its WAL, and the engine is unaffected by the account database. So trading continues for already-accepted orders, and only new order entry for that account shard is blocked. If the promotion takes 30 seconds, the blast radius is new orders for a few percent of accounts for 30 seconds.

The resume path: after promotion, the Position Service replays fills from the log and reconciles against Postgres, because in-memory position state may be ahead of the promoted replica. I reconcile forward from the log rather than trusting the database, since the log is the source of truth per our earlier answer. And I would run this as a routine drill, because a failover you have never executed is a hypothesis.

**Interviewer:** Cache stampede. After a symbol shard fails over, everyone watching that symbol re-subscribes and every quote is a miss.

**Candidate:** Two different stampedes and I will separate them.

The quote plane: the Fanout Service keeps quotes in memory, and after a shard failover it is the fanout shard, not the quote engine, that is stressed. Every connection for that symbol re-subscribes, and the flusher re-reads state. In-memory state means there is no cache to stampede, which is an argument for keeping the hot path in memory rather than in a cache in front of a database. The only miss is a single read from the quote engine per symbol, coalesced across all re-subscriptions by the request coalescer.

The REST snapshot path is the real stampede. 200,000 clients reconnecting and all calling `GET /v1/quotes?symbols=AAPL` at the same instant. That is 200,000 identical requests. Three mitigations. The CDN and edge cache absorb it, because that URL is identical for every user asking for that symbol. A per-symbol request coalescer collapses concurrent identical snapshot requests into one origin read. And a token bucket on the snapshot endpoint with a large burst allowance, because a sharp rate limit here would make recovery worse than the problem.

**Interviewer:** "Why not" round. Why not Kafka Streams or Flink instead of a custom engine?

**Candidate:** The honest answer is that Kafka Streams could do it, and I would reach for it if this were a team without low-latency trading experience.

The reasons I would not, specifically. Determinism. A Streams topology doing order book state is a distributed state store, and a rebalance moves partitions, which means the book must be rebuilt at exactly the wrong moment, and the state store restore is a local file read. Latency. Even a well-tuned Streams task adds a per-record serialization and state-store hop that is a multiple of a purpose-built in-memory single-threaded engine with a flat array price level. And failure semantics. In Streams, a task crash triggers a rebalance of the whole topology; in a sharded engine, one symbol's shard fails over in seconds and the other 4,999 never notice.

The cost of my choice is real: I own the recovery code, the snapshot format, the determinism discipline, and the rebalancing when symbols are added. That is a real ongoing cost and it is why I would not choose it for a 500-symbol crypto venue with modest volume.

**Interviewer:** Why not Redis for the order book?

**Candidate:** Because matching is a read-modify-write over a priority structure, and Redis gives you neither atomic multi-key mutation nor an ordered price level with FIFO time priority, without Lua scripting or a custom module. Then the whole correctness argument rests on a Lua script being correct, which I cannot unit test as easily as a C++ or Java matching function, and which becomes the single most critical piece of code in the company with the least scrutiny.

Redis is excellent at the thing next door to this, which is session state, idempotency keys, rate limits, and the connection-to-gateway mapping. I would absolutely use it for those. I would not use it as the order book.

**Interviewer:** Why not make the matching engine stateless and scale it horizontally?

**Candidate:** Because the matching function is inherently stateful and order-sensitive. Price-time priority means the result depends on the exact arrival order, so you cannot shard by order hash, only by symbol, and even within a symbol you cannot have two engines because they would each see half the book and produce a wrong result. There is no stateless formulation of this problem short of moving the whole book to every replica and doing consensus on every match, which is slower and more complex than the single-writer model.

The single-writer-per-symbol model is not a scaling limitation, it is the algorithm. One symbol, one thread, one total order, no locks. I scale by adding symbols, and the exchange is generous with symbols.

**Interviewer:** Replica lag. The replication factor is 3 with acks=all. What if the third replica lags?

**Candidate:** `acks=all` with `min.insync.replicas=2` means the producer only needs two in-sync replicas. If one falls behind or dies, writes continue with two, so the system keeps trading, and the degraded replica stops being eligible as an in-sync replica and the cluster shrinks to two. That is the correct trade: I would rather run degraded than stop trading.

If the second one also falls out, writes fail. At that point the shard has no quorum and I have to make a product decision. My choice: fail closed for new orders on that symbol, keep serving quotes, and surface a halt. I will not acknowledge an order I cannot durably record, because an unrecorded order that I later reconstruct differently is the exact over-fill and audit incident we are trying to prevent.

When a replica returns, it catches up from the leader. That catch-up is the one place lag actually hurts, because failover to a lagging replica is how you lose or duplicate an event. So I fence: the recovered replica cannot become leader until it has caught up to the last committed sequence, and leadership is held by an epoch term so a zombie leader from before the partition cannot come back and accept writes. That is the [[split-brain|Split Brain]] scenario and it is prevented by epoch fencing, not by hope.

**Interviewer:** Last technical one. The exchange feed drops for ninety seconds during a halt and reopens. What now?

**Candidate:** I have a gap, and the correct response is not to guess.

The normalizer tracks a sequence number per feed. On a gap, it marks the symbol's quote as `STALE` and stops publishing to the fanout for it, so the client sees an explicit stale flag rather than a frozen price that looks live. That distinction is a real product requirement and it is the kind of thing that is invisible in the design doc and obvious in production.

For quotes, the right answer is to resubscribe and take a fresh snapshot from the exchange, then resume, and mark the transition. For anything involving my own book, nothing is lost, because my book is in my log and independent of the feed. The dangerous case is if I were deriving display prices from my book, which is another reason I decided not to.

And I would have a trading halt path that is a deliberate, audited operation: a symbol-level halt, a broker-level halt, and a market-wide circuit breaker, with automatic halt on a detected sequence gap and a manual resume that requires a human. Auto-resume on a data-integrity failure is how you turn a 90-second feed gap into a 90-minute incident.

---

## Phase 5: Availability, Trade-offs, and Final Summary (30:00 - 45:00)

**Interviewer:** Give me the availability model. Not the percentage, the model.

**Candidate:** Availability of individual symbols, decomposed.

Matching for a symbol is available if any of its 3 replicas is alive and in-sync. That is the unit. Order intake for an account is available if its account shard has a primary. Quotes for a symbol are available if the quote plane is healthy, which is independent of both. Those are three separate availability domains and I deliberately kept them separate.

The failure modes, and what survives each.

A single node dies: nothing visible. Traffic and leadership shift, orders and quotes continue.

An availability zone dies: a third of replicas and a third of gateways go. Symbols whose surviving in-sync replicas are in the other two zones keep trading. 30 percent of the 1.2 million connections drop and reconnect, so I accept a visible reconnect burst.

The fanout fleet loses capacity: quotes degrade to a throttled, lower update rate, and I add a per-symbol egress rate limit so degradation is fair across symbols rather than first-come-first-served. Order entry is untouched, because it does not go through fanout, it only uses it for notifications, and a user whose notification is delayed can still query status by REST.

The order plane loses a shard: new orders rejected for that account or symbol, quotes unaffected, and an explicit halt. Trading on other symbols is unaffected.

The log loses a partition: this is the scariest one, because the log is the source of truth. RF=3 plus `min.insync.replicas=2` means a single partition loss is not data loss. A multi-partition loss is a page, and my answer is that the order event log is the one component I would replicate across regions despite the cost, because losing it is losing the business.

**Interviewer:** Trade-offs. Be honest about what you gave up.

**Candidate:** Six.

One, per-symbol partitioning means I cannot migrate a symbol's book between shards without a coordinated cutover. I pay with a symbol migration protocol using a transfer-then-activate sequence with a pause, and I only need it when rebalancing or upgrading. The alternative, re-hashing by order id, is not available to me at all.

Two, at-least-once delivery to consumers means every consumer must be idempotent. I dedupe on `symbol` plus `sequence`, and every consumer is idempotent, which is real work and it is the correct price of not running a distributed transaction.

Three, the buying power check is eventually consistent, up to a few milliseconds behind. I fail closed on lag, which costs availability in exchange for safety. A regulatory margin requirement would force a synchronous path.

Four, quote fanout at 20 updates per second per subscription is a deliberate throttle. A user doing something latency-sensitive about the display quote would be misled, which is why the display is labeled as delayed per your requirement.

Five, 11 nines on quote data, but 4 nines on the account ledger, because the ledger uses a 3-replica synchronous Postgres and I did not build consensus. A ledger is small and slow-changing relative to quotes, so the simpler durability model is the right one there.

Six, and this is the one I would lose sleep over: the single-threaded engine means one bad symbol, like an exchange-injected sequence of 60,000 orders a second on one name, occupies one core and can create backpressure on that shard's queue. I have headroom, and I would add a per-symbol admission cap plus a priority lane for cancels over new orders, so a flood of new orders cannot delay a user cancelling existing risk.

**Interviewer:** Why does a cancel need priority over a new order?

**Candidate:** Because a cancel is risk-reducing and a new order is risk-increasing. Under overload, I want to be preferentially shedding the operation that creates exposure. That is a small design decision with a disproportionate effect on how the system behaves in its worst moment, and it is the same principle as failing closed on risk checks and load shedding the read path before the write path.

**Interviewer:** Ninety seconds. Final architecture.

**Candidate:** Two planes.

The quote plane: exchange feed handlers and a normalizer publish canonical quotes to a partitioned log keyed by symbol. A quote engine consumes and maintains per-symbol state, and a periodic snapshotter writes book snapshots to a compaction topic so any consumer can rebuild in seconds instead of replaying to retention. A fanout service holds the subscriber sets in memory, coalesces ticks per connection to at most 20 per second, and packs 14-byte fixed-point quotes into binary frames. 40 WebSocket gateways, 30,000 connections each. Net 24 million messages per second, about 15 Gbps. No dependency on the order plane, ever.

The order plane: the client sends an order to any stateless Order Service pod with an idempotency key. The first thing that happens is a unique-constrained insert, so a retry returns the stored response and does nothing else. Then synchronous pre-trade risk, buying power from an in-memory shadow fed by the fill stream, fail closed if the stream is lagging. Then routing by symbol to a matching engine shard, single-threaded, in-memory, price-time priority, fill caps enforced in the same critical section as the mutation, self-trade check inside that critical section. The engine appends to a replicated log with `acks=all` and only then acknowledges. Events fan out to a client notifier and to persistence, which writes Parquet to object storage for permanent history.

The log is the source of truth. The book and the position are derived views, rebuildable from snapshot plus delta, and any replica can rebuild any of them.

Three trade-offs I accepted. Single-threaded per symbol, so a symbol cannot be split across machines, and I buy determinism and zero locks with the inability to migrate a book cheaply. At-least-once with idempotent consumers rather than exactly-once, so every consumer carries dedupe logic. And a 99.99 percent target expressed as per-symbol availability with an explicit halt, rather than as a platform percentage, because that is the thing I can actually control.

---

## Concepts to Study Separately

- [[kafka-ordering|Kafka Ordering]] - partition key, per-key ordering, and what breaks across partitions
- [[exactly-once-effect|Exactly-Once Effect]] - idempotent consumers versus true exactly-once, and where the boundary is
- [[split-brain|Split Brain]] - epoch fencing and leader epochs in depth
- [[tail-latency|Tail Latency]] - why p99 is the number that matters and how quorum reads make it worse
- [[hotspot-handling|Hotspot Handling]] - one symbol, one thread, and the skew problem
- [[load-shedding|Load Shedding]] - fair degradation across symbols instead of first-come-first-served
- [[outbox-pattern|Outbox Pattern]] - idempotency keys and the transactional boundary around them
- [[websockets|WebSockets]] - backpressure, heartbeats, and connection lifecycle at a million connections
- [[distributed-locks|Distributed Locks]] - why I refused to use a lock in the order path
- [[kafka-replication|Kafka Replication]] - ISR, `min.insync.replicas`, and catch-up behavior
