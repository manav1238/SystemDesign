---
title: Ticket Booking System Design - Problem Statement
status: active
tags: [hld, mock, ticket-booking]
---

# Ticket Booking System Design

## Problem Statement

> Design the ticketing platform behind a large live-events operator — concerts, theatre, sports, conferences. We sell directly to the public and we also act as a reseller for partner venues. The defining characteristic of this business is not average traffic, it is the spike: a popular on-sale at a fixed hour, where demand for a few thousand tickets exceeds supply by one to two orders of magnitude within minutes, and the entire system must behave sanely while everyone else in the world is refreshing the event page.
>
> In scope:
> - An event catalogue with events, venues, seat maps, price tiers, and per-seat or per-section pricing
> - A seat map: numbered seats in sections for reserved-seating events, and a simple capacity pool for general-admission events
> - Searching and browsing events by date, city, artist, and genre, with availability shown to the user
> - Creating a **seat hold** that reserves inventory for a bounded period, displayed to the user as a countdown timer
> - Completing payment during the hold window and converting the hold into a confirmed ticket
> - Releasing expired holds automatically and making the released inventory purchasable again
> - Cancellations and refunds, subject to a per-tier refund policy that is enforced at the time of sale
> - An idempotent booking API, because clients on mobile networks will retry
> - A waiting room or queue for the on-sale, so the spike is absorbed at the edge instead of collapsing the seat inventory service
> - A "sold out" state that appears quickly and correctly, so a sold-out event stops attracting traffic
>
> Out of scope:
> - The payment gateway itself beyond its interface: we assume a third-party processor with a synchronous authorize, an async capture, and a webhook
> - Ticket rendering, PDFs, wallet passes, and barcode generation
> - Venue seating-plan authoring tools; the seat map is given to us as data
> - Resale and secondary-market listings
> - Marketing, recommendations, and personalised discovery
>
> Hard requirements you must respect:
> - **Never oversell.** One seat, one ticket, one holder. A duplicate ticket for a reserved seat is a legal and reputational problem, and an oversold stadium is a disaster.
> - **Availability may be slightly stale; seat assignment may not.** Showing a user "8 seats left" when 3 remain is a bad experience. Selling the same seat twice is a catastrophe.
> - **A hold is not a sale.** Inventory that is held and then abandoned must return to the pool automatically, without operator intervention, and without a leak that permanently shrinks capacity.
> - **The on-sale is a denial-of-service problem wearing a business hat.** You must be able to shed load deliberately and survive it.
> - The system must be correct under concurrency, not just under a clean load test.

---

## Clarifying Questions You Should Ask

Strong candidates ask these before drawing anything. Pick the ones you genuinely need answered.

**On inventory semantics**
1. Do all events have a seat map, or are some general-admission with a single capacity number? This decides whether inventory is a set of seat rows or a counter.
2. Does a price belong to a section, to a tier, to a seat, or can it vary by quantity? Is there ever a bundle or a "best available" allocation?
3. Can seats be blocked for production holds, VIP allocations, accessibility, or staff comps, and when are those blocks applied?
4. Is there a maximum per purchase, and is it per event, per customer, or per order? This is a scalping control and it is a real part of the design.
5. What is the hold duration, and is it configurable per event or per price tier? Ten minutes is normal; VIP drops sometimes need twenty.
6. Can an order contain seats from more than one event, and can one seat be in a cart across two devices?

**On the on-sale**
7. What time does the sale start, and is it announced in advance? A published start time means a predictable stampede and a legitimate reason for a waiting room.
8. How many people typically arrive in the first thirty seconds, and how many tickets exist? The ratio, not the absolute number, is the design driver.
9. Are the pages served from a CDN ahead of time, and is the seat map pre-rendered? If the page itself is dynamic and hits your origin, you have already lost.
10. Is there bot traffic and ticket-resale scraping, and what is the current abuse rate? This changes the identity and rate-limiting design.
11. When sold out, what does the user see, and how fast? A sold-out event must stop generating read traffic, not keep serving a page that queries inventory.

**On correctness and money**
12. Is payment captured at hold time or at confirmation? If capture is async, what is the authoritative state between "hold active" and "ticket issued"?
13. If the payment fails after the hold, do we release the seat immediately or keep it for a retry window?
14. What is the cancellation policy per tier, and must the deadline be re-validated at cancellation time, not just at sale time?
15. Do we ever need a partial refund when an event is rescheduled or a seat is downgraded? This forces a first-class concept of ticket credit.

**On scale and operations**
16. How many events are on sale at once, and how many concurrently "hot"? Hot events are the unit of sharding and they are not evenly distributed in time.
17. What is the read-to-write ratio during a sale? I would guess one hundred reads per write, and I would like it confirmed.
18. Is multi-region needed, or is a single region with a strong on-sale story acceptable? Cross-region seat locking is the hardest problem in this design and I want to know if it is in scope.
19. What is the acceptable oversell tolerance in a crisis? If the database is on fire and we must choose between selling zero extra tickets and selling 0.1% extra, which does the business pick?
20. What is the SLO for the booking API, and what is the SLO for the event page during the first sixty seconds of a sale?

---

## What You Are Evaluated On

### Phase 1: Requirements
Separate functional from non-functional and, critically, separate the *browsing* requirements from the *buying* requirements. They have opposite scale characteristics and opposite consistency requirements. State the oversell decision explicitly, because the business has to own it.

### Phase 2: Estimate
Show the arithmetic for a flash sale, not for a day: requests in the first sixty seconds, read QPS, the demand-to-supply ratio, the number of concurrent seat-lock transactions, and the resulting storage for seat maps and inventory. Also show what the *average* day looks like, so the contrast is visible.

### Phase 3: High-level design
Draw the architecture. Identify the catalogue and search path, the seat map and availability path, the booking and hold service, the payment integration, the hold-expiry reaper, the waiting room, and the ticketing path. Name the shard key and say why — and be prepared to defend it, because "by event" is obvious but the hot-event concentration makes it non-trivial.

### Phase 4: Deep dive
The expected core is concurrent seat reservation: pessimistic locking with `SELECT ... FOR UPDATE` versus optimistic concurrency with a version column, the hold and timeout mechanism, the exact-overlap prevention strategy, and why availability can be cached while seat assignment cannot. Be ready to talk through the flash-crowd control plane and the exactly-once booking effect.

### Phase 5: Trade-offs and follow-ups
This problem is fundamentally about an availability-versus-consistency trade. Defend the choice: why not a distributed lock per seat, why not Redis-only inventory, why not let the payment step decide the seat. Know exactly what you gave up to get no oversell.
