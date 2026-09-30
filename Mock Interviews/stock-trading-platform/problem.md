---
title: Stock Trading Platform System Design - Problem Statement
status: active
tags: [hld, mock, stock-trading-platform]
---

# Stock Trading Platform System Design

## Problem Statement

> Design the backend of a retail brokerage that lets users stream live quotes and place, modify, and cancel stock orders.
>
> A user opens the app and sees a list of roughly 5,000 tradable symbols (US equities and ETFs). For any symbol they are watching, last price, bid/ask, and volume update many times per second and must appear on the screen within a fraction of a second of the exchange tick. The user can also place a market, limit, or stop order, and must be told definitively whether it was accepted, rejected, partially filled, or filled.
>
> In scope:
> - Ingesting a live market data feed from one or more exchanges and normalizing it
> - Fanning out live quotes to many thousands of concurrent users watching the same symbol
> - Maintaining an order book per symbol and matching incoming orders against resting orders on price-time priority
> - Order lifecycle: place, modify, cancel, and the resulting state transitions
> - Pre-trade risk and validation: sufficient buying power, price collars against fat-finger errors, max order size, self-trade prevention
> - Durable, replayable audit of every order and every trade, queryable for statements and regulatory review
> - Order placement retries that are safe, meaning a network timeout can never result in a duplicate order or a double execution
> - Surviving the loss of a node, and a broker reconnect, without losing or duplicating a trade
>
> Out of scope:
> - Options, futures, margin calculations beyond a simple buying-power check, and short selling mechanics
> - Level 2 depth-of-market data to retail clients (a single consolidated top-of-book is enough)
> - Regulatory reporting to exchanges, 1099 generation, and tax lot accounting
> - Charting, screeners, and news
> - The brokerage's internal risk and capital model
>
> Design for 3 million daily active traders across 5,000 symbols, an aggregate order rate that must survive a market-open burst, and quote fanout to hundreds of thousands of concurrent WebSocket connections. The order acknowledgement path must return in single-digit milliseconds at the 99th percentile, and the platform is expected to be available 99.99% of the time. Being down during market hours is the worst possible failure, so failure behavior matters as much as throughput.

---

## Clarifying Questions You Should Ask

Strong candidates ask these before drawing anything. Pick the ones you genuinely need answered.

**On market structure and semantics**
1. Do I need Level 1 (last trade, bid, ask) or Level 2 (full depth of book)? Level 2 changes the fanout cost by orders of magnitude.
2. Am I the exchange, or a broker routing to exchanges? This decides whether I own the order book or consume one.
3. Is the order book for a symbol owned by exactly one system at all times? How quickly must it fail over if that system dies?
4. What is the actual latency target, and is it measured to the client, to my matching engine, or to the exchange fill? Each is a different number by two orders of magnitude.
5. Is the price-time priority matching rule strict FIFO at a price level, or is there pro-rata allocation for market makers?
6. Do I need to support self-trade prevention and price collars, and are those configurable per account or hard-coded?

**On scale and shape of the load**
7. What is the distribution of order rate across symbols? I strongly suspect a handful of names dominate and I want to know the peak, not the mean.
8. What does the load look like at the open and the close? Is the burst 5x or 100x the intraday average?
9. How many symbols does a single user watch simultaneously, and how often do they add or drop a subscription?
10. Do users see quotes outside market hours, and do I need to run a continuous pre/post-market book?
11. Is the order rate dominated by retail bursts, or by algorithmic clients that send steady machine-speed flow?

**On correctness and money**
12. What is the exact definition of "no double execution" here — no duplicate order at intake, or no over-fill of a single order? They are different problems.
13. If the client times out after sending an order, how does it find out what happened? Is there a query-by-client-order-id contract?
14. What is the recovery point objective if the whole cluster dies mid-session? Can I replay from the exchange, or must the order event log be the source of truth?
15. How long must the full order and trade history be retained and queryable, and does an auditor need to reconstruct any single day's book exactly?

**On infrastructure**
16. Am I allowed to lean on Kafka as the durable log, or should the order book be a purpose-built in-memory engine with its own write-ahead log?
17. Is single-region acceptable, and does "99.99%" allow for an 8-minute maintenance window per quarter, or do they need active-active?
18. What is the acceptable cost ceiling per user per month, given that quote fanout dominates it?
19. What happens to orders in flight when the matching engine for a symbol crashes mid-match? I need to know if partial state is acceptable or if I must guarantee atomicity.

---

## What You Are Evaluated On

### Phase 1: Requirements
Separate functional from non-functional. Pin down the semantics that actually drive the architecture: which side of the trade you own, the strict ordering guarantee per symbol, the order state machine, and the definition of double execution. State assumptions explicitly rather than silently absorbing them.

### Phase 2: Estimate
Do the back-of-envelope math out loud with real numbers: daily to peak order rate, concurrent connections, outbound quote messages per second, event log volume and retention, per-shard throughput ceiling. Show the arithmetic, not just the conclusion. The per-symbol skew question is what separates strong answers from average ones.

### Phase 3: High-level design
Draw the architecture. Identify the market data ingest path, the quote fanout path, the order intake path with risk checks, the matching engine partitioned by symbol, the order event log, the position and account store, and the WebSocket gateway layer. Name the partition key for the durable log and say why it is the symbol.

### Phase 4: Deep dive
Pick two or three areas and go genuinely deep. The expected core is the single-writer-per-symbol matching engine and how it scales horizontally, the idempotency and state machine on the order path, and the replay and snapshot model that makes the book recoverable. Be ready to talk through quote fanout cost, hot symbol skew, and the exact failure behavior when one shard is lost.

### Phase 5: Trade-offs and follow-ups
Defend the choices against alternatives. "Why not Kafka Streams or Flink instead of a custom engine", "why not Redis for the order book", "why not just make the matching engine stateless and horizontal", "what happens when the primary for a symbol dies", "how do you handle one symbol taking 100x the average volume", "how do you survive replica lag and a quote cache stampede after a crash". Know exactly which trade-off you accepted and what it cost.
