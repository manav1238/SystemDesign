---
title: "Design Uber — Interview Study Transcript"
status: active
tags: [hld, mock, uber]
---

# Design Uber — Interview Study Transcript

A full 45-minute interview, transcribed. Read it once for flow, then use the section headings to drill individual phases.

## Phase 1 — Requirements Clarification (minutes 0-8)

Interviewer: Thanks for joining. I'd like you to design a ride-hailing platform, Uber-style. Take a few minutes to make sure you understand the problem before you estimate anything. What do you want to ask?

Candidate: Let me start by bounding the product, because Uber-shaped systems are usually several products sharing one platform. Is this ride-hailing only, or do we also need food delivery and courier on the same location and dispatch infrastructure?

Interviewer: Ride-hailing only for this exercise. But assume the platform is built so that a second product could be added later without rewriting the location pipeline.

Candidate: Good, that tells me the location and dispatch core must be product-agnostic, and the ride-specific logic sits above it. I'll design for that.

Interviewer: Next.

Candidate: The device edge. Is the driver app ours, or a third-party SDK? Specifically, can we control the reporting interval, the batching, and the retry behavior of location reporting?

Interviewer: It's our app, and we control the SDK. Assume we can adapt the reporting rate per driver per second.

Candidate: That is the single most important answer so far, because the location ingest bill is entirely a function of reporting rate, and reporting rate is a business-and-cost decision, not a physics constraint. If I can adapt the rate, I can cut ingest by 60 percent with no user-visible degradation, and I will show that arithmetic later.

Interviewer: Good. Now freshness. How fresh does a driver's location need to be? One number or more?

Candidate: More than one, and I want to separate them before I estimate anything, because conflating them is the most common error in this problem.

Candidate: Requirement one, the rider's moving map. A rider watching the car approach needs the marker to move smoothly. Human perception tolerates about 200 milliseconds of staleness before the map feels broken, and about one update per second is what looks like motion rather than teleportation. So sub-second, roughly 400 milliseconds, end to end.

Candidate: Requirement two, the matching index. A driver's true position versus the position in the index can differ by 300 to 500 milliseconds and the matcher will still assign the right driver, because the search radius is 300 meters and the ETA model is fuzzy. So the index tolerates a few hundred milliseconds of lag.

Candidate: Requirement three, trip history and replay. That can be minutes stale. Nobody watches it live.

Candidate: So I have three freshness tiers, roughly 400 milliseconds, roughly 2 seconds, and minutes. The design consequence is that only the first one is on a hard latency path, and I will not pay strong-consistency costs for the other two.

Interviewer: Now the matching question that decides the whole engine. When a driver receives a ride offer, is that offer exclusive? Can a driver hold two pending offers at once and pick one, or is the offer reserved for that driver the instant it is sent?

Candidate: This is the question. I want to ask two follow-ups: does the offer expire, and how long is the window?

Interviewer: Exclusive. Once an offer goes to a driver, that driver is reserved for that rider until the offer expires. The window is 15 seconds, and a driver has about 4 seconds to tap accept. If nobody accepts in 15 seconds, the rider is re-offered to the next candidate.

Candidate: Then I have the invariant stated precisely, and it is the hardest correctness property in the system: at most one rider is assigned to a driver at any instant, and at most one driver is assigned to a rider at any instant. That is mutual exclusion over two entity sets, and it has to hold under retries, under clock skew, and under a driver whose phone is offline.

Candidate: And it means matching is not a search problem, it is a compare-and-set problem wearing a search costume. The search is easy. The atomic claim on a driver is the whole problem.

Interviewer: Right. Is matching first-come-first-served, or does the system group riders and solve an assignment problem?

Candidate: I would expect the second, at least in dense zones, and I want to argue for it rather than assume it. First-come-first-served has a failure mode I can name: if two riders request in the same cell in the same second, FCFS gives the closer car to the earlier requester and makes the later requester wait even though a car 200 meters further away would reach them 90 seconds sooner. Batching those two riders and solving a small assignment problem reduces total rider wait. The cost is added latency for the first requester, which is why you would only batch inside a short window, on the order of a few seconds, and only where density justifies it.

Interviewer: Fine. Now money. Is there a real ledger here, or is the fare a number on the trip record?

Candidate: Is there double-entry accounting, refunds, promotional credits, and a nightly reconciliation against the payment processor?

Interviewer: Yes, and it must balance to the cent.

Candidate: Then the trip service and the money service are strongly consistent stores with synchronous replication, and they are not the same store, because a ledger and a trip have different lifecycles and different retention. And I will say now, before estimating, that this gives me a split I will defend all session: strong consistency for assignment and money, eventual consistency for location and history, and never anything in between.

Interviewer: Two more product questions. Cancellation after the driver has arrived, and driver cancellation.

Candidate: For cancellation, the question I need answered is: is the cancellation fee a function of driver arrival, or of elapsed time? Because arrival is a system-observed event with a timestamp we control, while elapsed time is inferable. I'd prefer "driver arrived" as the trigger, because it is a fact in our trip event log rather than a threshold we argue about.

Interviewer: Driver arrival. And drivers can cancel too, though they take a hit.

Candidate: Then driver cancellation is the interesting failure, because the offer is gone, the rider is stranded, and a payout may already have been reserved. The right shape is: driver cancels is a legal transition that emits an event, the matching engine immediately re-enters the batch for that rider with a penalty-weighted scoring function that avoids re-offering to the same driver, and the rider sees an honest message. I'll come back to it.

Interviewer: Non-functional targets. Give me numbers.

Candidate: Match latency: p50 under 250 milliseconds, p99 under 1 second, measured from the ride request being accepted at the edge to the driver assignment being durably recorded.

Candidate: Location freshness on the rider's map: p99 under 400 milliseconds from phone to screen.

Candidate: Availability: 99.95 percent for the match path, because a failed match is lost revenue and it is the only product interaction. Lower for trip history, maybe 99.9 percent, because it is read monthly. And I would put a third number on the page, driver-request push notification delivery within 2 seconds, because a driver who does not see the request in 2 seconds is a lost trip.

Candidate: And the number that does not fit in an SLI: freshness of the location index. I'll manage that with an explicit staleness SLI, p99 under 2 seconds, and treat a breach as a paging alert rather than a dashboard line, because a silently stale index produces wrong matches rather than errors.

Candidate: Consistency, precisely. Trip assignment and the money ledger are strongly consistent, synchronously replicated, single-writer-region. The location index is eventually consistent within 2 seconds. Rider trip history is eventually consistent within 30 seconds. Driver earnings and statements can be 5 minutes stale. That is four different answers for four different data types, and I will justify each one by the cost of being wrong, not by a slogan.

Interviewer: Good. Estimate.

## Phase 2 — Scale Estimation (minutes 8-16)

Interviewer: Back-of-the-envelope. Start where you want.

Candidate: Let me start with money, because it bounds everything. 25 million trips per day at a 20 dollar average fare is 500 million dollars a day of gross bookings, and 20 percent take rate is 100 million a day of revenue. That tells me the revenue-at-risk per failed match, which is why I am going to be strict about the match path availability number. It also tells me the trip database is not a big-revenue database in a storage sense, which surprises people.

Candidate: Trips per day: 25 million. Peak hour is not 1/24, it is about 10 percent of the daily volume on a Friday evening in a big city, so 2.5 million trips in the peak hour, which is 694 trips per second. I'll carry 700 per second.

Candidate: Match attempts. Not every trip matches on the first try. City density, time of day, and the 15-second offer expiry all create retries. I'll assume 2.5 search-and-offer attempts per successful match on average, and then a separate retry stream for the riders who never match. That gives about 1,700 offer-generating searches per second from matched trips, plus 30 percent extra from no-match retries, so call it 2,200. Round to 2,500 match searches per second.

Candidate: Now the number I think this problem is actually about. Location ingest.

Candidate: 5 million drivers online at peak. I will not divide by one interval, because drivers are in different states and the states have different reporting rates.

Candidate: On-trip drivers: roughly 1.5 million at peak, because concurrent trips at peak is about 1.2 million, and drivers spend some of the trip looking for the next rider. I will report every 4 seconds while moving. 1.5 million divided by 4 is 375,000 pings per second.

Candidate: Idle-but-online drivers: 3.5 million. An idle driver far from demand does not need a 4-second heartbeat; the rider's map shows a dot, and a dot that lags 30 seconds is still a dot. I will report every 30 seconds. 3.5 million divided by 30 is about 117,000 pings per second.

Candidate: Total: about 492,000 pings per second. Call it 500,000 location messages per second, 24 hours a day, and roughly 45 percent lower at 4am.

Candidate: Let me check that with an independent method. 500,000 per second times 86,400 seconds is 43.2 billion pings per day. Times 80 bytes of payload, which is driver id, lat, lon, heading, speed, accuracy, and timestamp, is 3.45 terabytes per day of raw ingest. For a company that serves 150 million accounts, that is a small number, and I think that is genuinely true. Location ingest is not expensive in storage. It is expensive in messages per second and in the fan-out I have not counted yet.

Candidate: The fan-out is the real cost centre, and this is the part most answers miss. Concurrent trips at peak: 25 million trips a day times an 18 minute average ride is 450 million ride-minutes per day, divided by 1,440 minutes is 312,500 concurrent trips on average, and about 4 times that at peak, so 1.25 million riders are watching a live map mid-trip.

Candidate: Each of those 1.25 million riders receives a position update at 1 hertz. That is 1.25 million outbound messages per second to riders, plus the pre-request map which is 1 to 2 million connected riders at 0.2 hertz, which is about 300,000 per second. Total outbound fan-out about 1.5 million messages per second.

Candidate: Now the comparison I want to land. Ingest is 500,000 messages per second inbound. Fan-out is 1.5 million messages per second outbound. The system spends three times more effort telling riders where drivers are than it does collecting driver positions. And the fan-out is also the larger bandwidth bill, because outbound frames carry the route progress and ETA as well as a point: 1.5 million times 60 bytes is 90 megabytes per second, about 7.8 terabytes per day, more than double the 3.45 terabytes of ingest.

Candidate: Design conclusion before I draw anything: the location index and the realtime fan-out are the two scaling problems, and they are different systems with different shapes, and the trip database is by comparison a small, high-value, strongly consistent transactional store that deserves most of the correctness budget and almost none of the scaling budget.

Interviewer: Storage.

Candidate: Storage, and I will separate latest-only from history because that is a factor of a hundred.

Candidate: Latest position, which is what matching reads. 5 million drivers times about 100 bytes, including the driver id and a small scoring digest, is 500 megabytes per copy. Three copies is 1.5 gigabytes. That fits entirely in memory, and I intend for it to.

Candidate: The cell index. At a resolution with roughly 170-meter edges, there are on the order of 1.6 million populated cells globally, and each cell holds a compact serialized entry. Call it 200 bytes per cell for the index plus the driver id list, and that is 320 megabytes. Again, in memory, everywhere.

Candidate: Location history. At 4-second resolution for the last 60 minutes, that is 15 pings per driver per hour times 5 million drivers times 80 bytes is 6 gigabytes per hour, so 144 gigabytes per day if I never downsample. I will not store it that way. I downsample to one point per minute for trip replay and dispute resolution and keep 90 days, which is 5 million drivers times 1,440 minutes times 90 days times 80 bytes, about 52 terabytes. That goes to object storage in a columnar format, and the analytical copy is a further 100x downsample in a lake.

Candidate: Trips. 25 million a day at 2 kilobytes per trip record including addresses, route summary, fare breakdown, and surge is 50 gigabytes a day, about 18 terabytes a year. Small.

Candidate: Accounts and drivers. 150 million accounts at 1.5 kilobytes is 225 gigabytes. Driver compliance documents, license, vehicle, insurance, background check, 7 million drivers at 6 kilobytes is 42 gigabytes.

Candidate: Ledger entries. 25 million trips at roughly 500 bytes of entries, and there are about 4 entries per trip, is 50 gigabytes a day. Same order as trips, and it is append-only so it compresses hard.

Candidate: The total steady-state store is roughly 100 to 200 terabytes, of which 52 is downsampled history. Compare that to 7.8 terabytes per day of fan-out traffic. Storage is not where the difficulty is in this system, and saying so out loud is what tells you where to spend the design effort.

Interviewer: Traffic. Requests per second, classified.

Candidate: I will separate them into four classes, because they have four different designs and one of them is twenty times bigger than the others.

```
 Class                       Peak rate    Handler                Budget
 --------------------------  -----------  ---------------------  --------------------------
 Location ingest             ~500K/s      ingest svc -> Kafka     100ms, fire-and-forget
 Rider map fan-out           ~1.5M/s      realtime gateway       400ms, persistent conn
 Match searches              ~2.5K/s      matching engine        250ms p50, 1s p99
 Trip lifecycle writes       ~7K/s        trip svc -> Postgres   50ms, strong
 Driver state transitions    ~6K/s        driver svc -> Postgres 50ms, CAS
 Notification sends          ~2K/s        notification svc       2s delivery, async
 UI, fares, receipts         ~20K/s       BFF                    150ms
```

Candidate: Trip lifecycle writes at 7,000 per second is worth unpacking, because people assume the trip database is the hot store. A trip generates about 10 state changes, so 700 trip creations per second times 10 is 7,000 writes per second. Spread across 256 shards that is 27 writes per second per shard. That is a rounding error for a database. Meanwhile 500,000 location messages per second spread across a topic is 40 messages per second per partition, which is fine, but there are 2,500 of them.

Candidate: So the honest summary is: the transactional side of Uber is a comfortable load, and the streaming side is the whole problem. I would rather say that clearly than pretend the database is the hard part.

Interviewer: Design the APIs.

## Phase 3 — API and Data Design (minutes 16-22)

Interviewer: Public and internal contracts. Start with the ones a client calls.

Candidate: Let me group by direction. Rider-facing, driver-facing, and internal.

Candidate: Rider-facing, HTTP through a BFF that aggregates and hides internal topology.

```
 POST   /v1/rides                      create a ride request
 GET    /v1/rides/{trip_id}            trip state, driver summary, ETA
 POST   /v1/rides/{trip_id}/cancel     cancel with reason
 GET    /v1/geo/nearby                 drivers visible on the pre-request map
 GET    /v1/riders/me/trips            ride history, cursor paginated
 GET    /v1/riders/me/receipts/{id}    immutable receipt
 POST   /v1/riders/me/trips/{id}/rate  rate the driver
```

Candidate: Driver-facing, same BFF, different auth subject.

```
 POST   /v1/drivers/me/availability    { online: true|false }
 POST   /v1/drivers/me/location        batched position report
 POST   /v1/offers/{offer_id}/accept
 POST   /v1/offers/{offer_id}/decline
 GET    /v1/drivers/me/trips/current   the in-flight trip
 GET    /v1/drivers/me/earnings        daily and weekly statements
```

Candidate: The realtime channel, which is a WebSocket per [[websockets]], not polling, because polling a 1 hertz position update with 1.25 million riders means 1.25 million long-poll requests held open, which is the same socket count by a different name and a much worse failure mode.

```
 wss://rt.example.com/v1/stream        ?token=<jwt>
 subscribe { channel: "trip:{id}", rate_hz: 1 }
 subscribe { channel: "map:{geohash}", rate_hz: 0.2 }
 msg     { type:"driver_moved", trip_id, lat, lon, heading, eta_s, seq }
```

Candidate: Payload details matter here. I will send a position delta, not a full position, and I will send the sequence number so the client can drop out-of-order frames and detect its own gap. And the client interpolates between the 4-second pings, so a 1 hertz outbound stream is smooth without the ingest ever running at 1 hertz.

Interviewer: Internal contracts.

Candidate: Internal, and this is where the interesting ones are.

```
 POST /internal/v1/matches              run a match batch for a zone
 POST /internal/v1/drivers/{id}/state   CAS transition, returns 409 on conflict
 GET  /internal/v1/geo/query            cell ring search, returns candidate digests
 POST /internal/v1/trips/{id}/events    append trip event, idempotency key required
 GET  /internal/v1/zones/{id}/surge     current multiplier
 POST /internal/v1/notifications        send, idempotency key required
```

Candidate: Two of those deserve explanation. The state transition endpoint is a compare-and-set, and the 409 is a normal outcome, not an error, because it means another requester got the driver first. And the trip event append requires an idempotency key because the transition will be retried on timeout and I cannot distinguish "the write failed" from "the response failed".

Candidate: Sagas. Creating a ride is not a single transaction. Let me name them.

```
 RideRequested        -> validate rider, compute fare+surge snapshot, hold payment
 PaymentAuthorized    -> persist trip in CREATED, publish RideCreated
 RideCreated          -> enter match batch, enqueue map pin
 DriverAssigned       -> CAS driver state, write assignment, publish DriverAssigned
 DriverAssigned       -> push offer to driver, push trip to rider
 OfferAccepted        -> transition trip to ASSIGNED, start billing meter
 TripStarted          -> begin route recording, begin realtime channel
 TripCompleted        -> stop meter, compute surge-adjusted fare, close payment
```

Candidate: And the failure branches: no match within the window is a legal terminal state, and driver cancellation re-enters the match step. Every forward step has a compensating or a retry path, and every one of them is driven off the event log rather than off synchronous call chains, which is what keeps the ride path under a second. Study separately: [[saga-and-strangler|Saga and Strangler]] and [[event-driven-architecture|Event Driven Architecture]].

Interviewer: Schema.

Candidate: I will do this as a set of tables rather than one big schema, and I will lead with the one that carries the money.

```
CREATE TABLE trips (
  trip_id            BIGINT PRIMARY KEY,
  rider_id           BIGINT NOT NULL,
  driver_id          BIGINT,
  status             TEXT NOT NULL,
  requested_at       TIMESTAMPTZ NOT NULL,
  assigned_at        TIMESTAMPTZ,
  arrived_at         TIMESTAMPTZ,
  started_at         TIMESTAMPTZ,
  completed_at       TIMESTAMPTZ,
  cancelled_at       TIMESTAMPTZ,
  cancel_reason      TEXT,
  cancel_actor       TEXT,
  pickup_lat         DOUBLE PRECISION NOT NULL,
  pickup_lon         DOUBLE PRECISION NOT NULL,
  dropoff_lat        DOUBLE PRECISION NOT NULL,
  dropoff_lon        DOUBLE PRECISION NOT NULL,
  surge_multiplier   NUMERIC(5,2) NOT NULL,
  fare_base_cents    BIGINT NOT NULL,
  fare_surge_cents   BIGINT NOT NULL,
  fare_toll_cents    BIGINT NOT NULL,
  fare_cancel_cents  BIGINT NOT NULL,
  fare_total_cents   BIGINT NOT NULL,
  currency           CHAR(3) NOT NULL,
  promo_id           BIGINT,
  version            BIGINT NOT NULL,
  created_at         TIMESTAMPTZ NOT NULL,
  updated_at         TIMESTAMPTZ NOT NULL
);
```

Candidate: Two design choices in that table. `surge_multiplier` is snapshotted onto the trip at request time, so if surge changes mid-ride the price the rider agreed to does not move. And `version` is an optimistic-concurrency counter, because the trip has concurrent writers, the rider cancelling while the driver arrives.

Candidate: The event log, append-only, is the audit trail and the recovery mechanism.

```
CREATE TABLE trip_events (
  event_id      BIGINT PRIMARY KEY,
  trip_id       BIGINT NOT NULL,
  seq           INT NOT NULL,
  type          TEXT NOT NULL,
  actor_type    TEXT NOT NULL,
  actor_id      BIGINT,
  payload       JSONB NOT NULL,
  idempotency_key TEXT,
  occurred_at   TIMESTAMPTZ NOT NULL,
  UNIQUE (trip_id, seq),
  UNIQUE (trip_id, idempotency_key)
);
```

Candidate: The unique constraint on `idempotency_key` inside a trip is the whole trick. A retried transition is a no-op, not a second state change, and I do not need a distributed transaction to guarantee that.

Candidate: The ledger, double-entry, never updated in place.

```
CREATE TABLE ledger_entries (
  entry_id     BIGINT PRIMARY KEY,
  txn_id       UUID NOT NULL,
  account      TEXT NOT NULL,
  direction    SMALLINT NOT NULL,
  amount_cents BIGINT NOT NULL,
  currency     CHAR(3) NOT NULL,
  trip_id      BIGINT,
  created_at   TIMESTAMPTZ NOT NULL
);
```

Candidate: Rider, driver, platform, and processor clearing are accounts. Every trip produces a balanced set of rows and the invariant is that the sum of an entire `txn_id` is exactly zero. Nothing is ever updated, so there is no lost-update problem and the nightly reconciliation is a single aggregate query. Study separately: [[transactions-and-acid|Transactions and ACID]] and [[soft-delete-audit-tables|Soft Delete and Audit Tables]].

Candidate: Driver state, and this is the hot row.

```
CREATE TABLE driver_state (
  driver_id      BIGINT PRIMARY KEY,
  availability   SMALLINT NOT NULL,
  current_trip_id BIGINT,
  zone_id        BIGINT,
  version        BIGINT NOT NULL,
  last_ping_at   TIMESTAMPTZ,
  updated_at     TIMESTAMPTZ NOT NULL
);
```

Candidate: The claim is a single statement, and I want you to see that it is one statement and not a read-then-write.

```
UPDATE driver_state
SET availability = 1, current_trip_id = :trip, version = version + 1
WHERE driver_id = :driver
  AND availability = 0
  AND current_trip_id IS NULL;
```

Candidate: If `rows_affected` is 0, someone else won. That is the entire mutual-exclusion mechanism, and it is a database-enforced invariant rather than an application lock, so it survives a client crash, a retry, and a deploy at the same time. Study separately: [[distributed-locks|Distributed Locks]] and [[database-locking|Database Locking]].

Interviewer: The driver state table is one row per driver, 5 million rows, and you just told me matching touches 2,500 times a second. How is that row distributed, and is that not a hotspot?

Candidate: 5 million rows sharded by `driver_id` modulo 4,096 shards is about 1,200 drivers per shard and 2,500 writes per second spread over 4,096 shards, which is 0.6 writes per second per shard. It is not a hotspot at all, and I want to be honest that the matching engine's fear of contention is misplaced. The contention that does exist is not on the row, it is in the candidate selection in a dense cell, and that is a different problem I will come back to.

Candidate: But I would still keep the state in a sharded store rather than one big Postgres, because a single database holding 5 million rows that are mutated on the critical path of every match is a shared-fate dependency I do not want. The rule I am applying: a strongly consistent store is fine at this size, but it should be sharded so that losing one shard is a localized incident.

Interviewer: Now the parts that are not relational.

Candidate: Three non-relational stores, and I will be explicit that they are key-value, not a general-purpose NoSQL substitute.

Candidate: One, the latest position. Key is `driver_id`, value is `{lat, lon, heading, speed, accuracy, ts, zone_id}`. About 500 megabytes total, held in memory in a sharded in-process map replicated three ways. The access pattern is a point read by driver id and a bulk read by cell, and that second one is served by the next store, not this one.

Candidate: Two, the cell index. Key is `cell:{res}:{cell_id}`, value is a compact list of driver ids plus a scoring digest per driver. About 320 megabytes. The access pattern is a ring read, so the structure that matters is the key: cell id, not driver id, because the query is a spatial one and a spatial query cannot be served by a key you do not have.

Candidate: Three, the offer and notification queue state. Key is `offer_id`, short TTL, and a per-driver inbox of pending offers, which is the thing that makes an exclusive offer cheap to represent.

Candidate: And the fourth, which is a log not a store: the location stream. Key is `driver_id` so that all pings for a driver are ordered, which I need for delta encoding and for reconstructing a driver's path during a trip. Study separately: [[kafka-ordering|Kafka Ordering]] and [[shard-routing|Shard Routing]].

## Phase 4 — High-Level Architecture and Data Flows (minutes 22-30)

Interviewer: Draw it.

Candidate: Here is the whole system, grouped by scale unit, because the boundaries between these groups are the design.

```
                                 RIDER APP            DRIVER APP
                                    |                     |
                              HTTPS / WSS              HTTPS / WSS
                                    |                     |
                    +---------------+---------------------+-------------+
                    |                GLOBAL EDGE (anycast, TLS, WAF)      |
                    +-------------------+--------------------------------+
                                        |
                         Geo-DNS / global LB -> nearest region
                                        |
     +------------------+---------------+----------------+---------------+
     |                  |                                |               |
+----------+     +--------------+                 +---------------+  +------------+
| Rider    |     | Driver BFF   |                 | Location      |  | Realtime   |
| BFF      |     |              |                 | Ingest        |  | Gateway    |
|          |     | avail, offers|                 |               |  |            |
| +-------+ |     +------+-------+                 +-------+-------+  +-----+------+
|  |ride   |            |                                 |              |  1.5M msg/s |
|  |svc    |            |                          Kafka  driver-   |  fan-out    |
|  +-------+ |            |                          location          |             |
+-----+-----+            |                          (500K msg/s)       |             |
      |                  |                                |             |             |
      |           +------v-------+                        |             |             |
      |           | Driver Svc   |                        |             |             |
      |           | state + CAS  |                        |             |             |
      |           +------+-------+                        |             |             |
      |                  |                                |             |             |
+-----v------------------v-----+      +-------------------v-----------v-------------+
|            MATCHING ENGINE         |        GEOSPATIAL INDEX                    |
|  batch per zone, score, CAS claim  |  cell -> driver ids + scoring digest       |
|  2.5K searches/s, p99 < 1s         |  in-memory, 320MB, 3 copies             |
+-----+--------------------------------+------------------------+--------------------+
      |                                |                        |
      |  +-------------+                |  +-------------+      |
      +->| Trip Svc    |                +->| Location    |------+
         | state machine|                   | Query Svc   |  zone -> surge
         +------+------+                   +-------------+  batch
                |
       +--------+---------+---------------------------+
       |                  |                           |
+-------------+  +---------------+          +------------------+
| Trip DB     |  | Money /       |          | NOTIFICATION SVC |
| Postgres,   |  | Ledger        |          | push, SMS, FCM   |
| strong,     |  | double-entry  |          | idempotent       |
| 256 shards  |  +-------+-------+          +------------------+
+-------------+          |
                  +-------v--------+
                  | Trip History   |
                  | + object store |
                  | read models    |
                  +----------------+
```

Candidate: Read that as five independent scale units that can each be deployed and failed independently: the ingest pipeline at 500,000 messages per second, the realtime fan-out at 1.5 million messages per second, the geospatial index as a small in-memory thing, the matching engine at 2,500 searches per second, and the transactional core at 7,000 writes per second. Study separately: [[microservices|Microservices]] and [[cell-based-architecture|Cell-Based Architecture]].

Interviewer: Walk me through the location update path. Driver taps nothing, the app just reports, and the rider sees a moving dot.

Candidate: Six steps, and I will give the budget for each because the 400 millisecond target is only achievable if I spend it deliberately.

```
 t=0ms      Driver app fixes GPS, builds the ping {id, lat, lon, hdg, spd, acc, seq}
 t=10ms     Batched with other pings, up to 50 per batch, sent over the existing TLS
 t=40ms     Location ingest, anycast LB -> nearest ingest node, auth, rate-limit per driver
 t=55ms     Produce to Kafka topic driver-location, key=driver_id, ack=all
 t=70ms     Index consumer: 3+2+2 = 7 subs consume this topic
 t=110ms    Consumer 1: update latest-position map and cell index, in memory
 t=120ms    Consumer 2: update zone supply/demand counters -> surge
 t=130ms    Consumer 3: if driver is on a trip, resolve trip_id, write to live-map topic
 t=180ms    Consumer 4: trip history downsampler
 t=200ms    Consumer 5: live-map topic -> realtime gateway, batched per gateway node
 t=240ms    Gateway pushes to rider sockets, delta-encoded, seq-stamped
 t=300ms    Rider app interpolates and paints, sub-frame
```

Candidate: The 40 milliseconds from app to broker and the 170 milliseconds of consumer fan-out are where the money goes. The design move that creates headroom is that nothing on this path touches Postgres. The whole path is memory and a log, and if I put a database write in front of the index update, I have spent the entire budget on one row.

Candidate: The second design move is batching. The gateway batches per node and per rider, so 1.25 million riders at 1 hertz becomes about 1,500 gateway nodes each emitting roughly 1,000 messages per second, which it can turn into fewer, larger, more efficient frames.

Interviewer: Now the ride request path.

Candidate: Seven steps for a request to a match.

```
 1  POST /v1/rides at the rider BFF, authenticated, rate-limited per rider
 2  Ride service: idempotency check on (rider_id, client_request_id)
 3  Fare service: base fare + distance + duration + SURGE SNAPSHOT + promos
 4  Payment: authorize (hold), not capture. Idempotency key = trip_id
 5  Write trip row, status = CREATED, in the home region for that city
 6  Outbox row in the same transaction -> RideCreated on the bus
 7  Matching engine: rider is appended to the pending list for cell(res9, pickup)
 8  Within <= 500ms, the next batch tick for that cell runs:
      a  fetch candidate digests from the cell index ring, 1 -> 7 -> 19 cells
      b  score: eta_seconds = f(distance, speed, traffic, heading) * 0.6
                + rating * 0.1 + acceptance_rate * 0.1 + ... -> normalized 0..1
      c  order candidates, attempt CAS claim on the top N=5
      d  on CAS success: write assignment, transition trip to ASSIGNED,
         publish DriverAssigned, send push offer with a 15s expiry
      e  on CAS failure for all N: widen to the next ring, retry until budget
 9  p50 ~250ms, p99 < 1s from step 1
```

Candidate: Step 4 is deliberate. We authorize, we do not capture, because we do not want the rider's money moving if no driver ever appears. Capture happens at trip start. That is a small decision that eliminates an entire class of refund automation.

Candidate: Step 8e is where the whole design is load-bearing. We do not hold a transaction open while we search and while we push to a driver's phone, because pushing to a phone is a network call to a device on a cellular network and could take 3 seconds or fail forever. So: search, claim atomically, then notify, and treat notification as best-effort with the trip as the source of truth. The claim happens before the notification, never after, because the reverse ordering allows a double assignment.

Interviewer: What is the scoring function, and why those weights?

Candidate: I will be honest that a real system learns these weights, and I will describe the shape and the guardrails. Candidates are ranked by a learned score, roughly predicted pickup ETA in seconds as the dominant feature, then driver rating normalized, then acceptance probability, then a small penalty for a driver who is already offered to another rider, then vehicle class eligibility as a filter not a score, and surge zone as a tiebreaker so the platform prefers drivers who can capture a profitable fare.

Candidate: Guardrail one, the score is never allowed to break the hard filters: driver must be AVAILABLE, must not be on a trip, must be inside the radius, must be within the rider's time window. Guardrail two, ETA is a model output, so I keep a pure-distance fallback for a cold model or a model timeout, and the fallback is worse but never absent. Guardrail three, exploration, a small fraction of offers go to non-top-ranked drivers so we do not lock into a policy that stops learning.

Candidate: And the reason to batch at all, restated: with a batch of 5 riders and 20 drivers in the cell, greedy first-come-first-served can produce a total wait of 11 minutes across the batch, while a small assignment solve produces 6. Same drivers, same cars, better outcome. Study separately: [[search-ranking|Search Ranking]] and [[distributed-scheduling|Distributed Scheduling]].

## Phase 5 — Deep Dive (minutes 30-42)

### 5.1 The geospatial index

Interviewer: Explain the index. Why not just put latitude and longitude in Postgres and query it?

Candidate: Because the query is a range query on two floating-point columns and Postgres will not use a B-tree for it, so it does a full scan, and even with a bounding-box prefilter you then need a distance function on every candidate. At 2,500 searches per second each touching 60 candidate cells, that is 150,000 geospatial evaluations per second, and I do not want a database in that loop.

Candidate: The index is a hierarchical hex or quadtree, roughly a 170-meter cell at the default resolution. The key is `cell_id`, and the value is the candidate list. Two properties make it work: the cell is a bounded polygon, so distance is approximated by the cell's center distance with a known error bound, and the key is hierarchical, so I can aggregate drivers to a coarser cell when a region is hot.

Candidate: Why hex over quadtree: hexagons have six neighbors, which makes ring traversal symmetric, and no cell neighbors are ever adjacent, so a ring query is exactly two conditions rather than a distance filter. Why not geohash: geohash is a string prefix, so a radius query is a prefix range on a lexically sorted key, which is workable but makes the variable cell sizes at boundaries awkward for a fixed 300-meter radius.

Candidate: On a miss or a cold cell, a quadtree at two finer resolutions under each parent is the escape hatch for the last 50 meters, where a single coarse cell is too large to rank candidates well. That is a two-level structure, coarse for the ring, fine for the boundary.

Candidate: Staleness policy, which is the part I would get asked about. The index is updated by the stream, so it is at most 2 seconds behind. I never read the index for anything that needs to be current. It is a candidate generator, and candidates are then verified against the authoritative driver state, which is the thing I actually trust. Study separately: [[database-indexing|Database Indexing]] and [[shard-routing|Shard Routing]].

Interviewer: The cell index is a map of maps in memory. What are you storing in it, and how do you keep it correct?

Candidate: Per driver, in the cell: driver id, availability flag, a compact scoring digest of about 60 bytes, a last-ping timestamp, and a small bloom filter or generation stamp used for cheap removal. No names, no phone numbers, no payment data. If a driver moves between cells, the consumer removes from the old cell and adds to the new, and the generation stamp is how the removal knows it is removing the right version and not a stale duplicate.

Candidate: Correctness has one rule: entries are removed by TTL on `last_ping_at` rather than by trusting the move event. If a removal is lost, the driver expires on their own after 60 seconds, and the search filters on `last_ping_at` freshness anyway, so a stale entry cannot produce a bad match, it can only waste a candidate slot. That is fail-safe rather than fail-noisy, and I prefer it because it does not need a repair job to be correct.

### 5.2 Hot regions

Interviewer: It is 6pm on a Friday in Manhattan. Tell me what happens.

Candidate: The failure I would design against is a cell becoming a contention point, not a capacity point. Twenty thousand drivers in one cell, several hundred ride requests per second in that cell, and my CAS loop is trying to claim the same popular drivers. I will get a high CAS failure rate, and the symptom is that match latency p99 goes from 400 milliseconds to 3 seconds in one zone while the rest of the city is fine.

Candidate: Five mitigations, in the order I would ship them.

```
 1  Adaptive cell subdivision. Hot cells split to a finer resolution (170m -> 60m).
    Hierarchical keys make this a one-time reindex of that cell, not a global rebuild.
 2  Widen instead of retry. After N failed CAS attempts, grow the search ring rather
    than hammering the same drivers. Contention becomes latency in geography, not
    latency in the datastore.
 3  Batch widening. Group the batch's riders and claim drivers in one pass,
    so a contested driver is attempted once per batch, not once per rider.
 4  Candidate cap per attempt. Hard-cap at 5-8 CAS attempts, then widen.
    Bounds the amplification factor at 2.5K searches/s * 8 = 20K CAS/s worst case.
 5  Surge as a supply signal, not just a price. Push the multiplier up in the hot
    cell so drivers are incentivized to route there, which is the only mitigation
    that increases supply rather than just spreading demand.
```

Candidate: And the guard I would insist on: a per-zone circuit that sheds matching load when the CAS failure rate exceeds a threshold, returning a "finding your driver, this is busy" state rather than a spinner. A user who sees honest scarcity churns less than a user who watches a spinner for 8 seconds. Study separately: [[hotspot-handling|Hotspot Handling]] and [[load-shedding|Load Shedding]].

### 5.3 The state machines

Interviewer: Show me the trip state machine, and then the driver one, because I think people conflate them.

Candidate: They are different machines over different entities and they are deliberately not the same machine.

```
 TRIP
   (none) -> CREATED -> OFFERING -> ASSIGNED -> ARRIVING -> IN_PROGRESS -> COMPLETED
                 |           |           |            |             |
                 |           |           |            |             +-> DISPUTED
                 |           |           |            +-> CANCELLED_BY_RIDER
                 |           |           |            +-> CANCELLED_BY_DRIVER
                 |           |           +-> ARRIVED
                 |           +-> OFFER_EXPIRED -> (back to CREATED, retry)
                 +-> PAYMENT_FAILED
                                              +-> CANCELLED_BY_RIDER (pre-pickup, fee applies)
                                              +-> EXPIRED (no driver in window)
```

Candidate: The rules that people get wrong. First, `COMPLETED` is terminal, and a dispute is a separate machine layered on top, not a trip state, because a dispute can apply to a completed trip days later. Second, a rider cancellation before `ARRIVED` costs a fee and a cancellation after `ARRIVED` costs nothing, and the branch point is a system-observed event, not a client assertion. Third, `OFFER_EXPIRED` is not an error, it is the normal retry path, and it must not consume the rider's "no driver found" budget.

```
 DRIVER
   OFFLINE -> ONLINE_IDLE -> OFFERING -> ONLINE_IDLE
                            |  \
                            |   +-> OFFLINE (declined all, or went offline)
                            +-> ON_TRIP -> OFFERING -> ON_TRIP
                                          |
                                          +-> ONLINE_IDLE
   any -> SUSPENDED (compliance, deactivation)   terminal until appeal
```

Candidate: The invariant linking them: a driver has at most one `current_trip_id`, and a trip has at most one `driver_id`. Enforced by the CAS on `driver_state` plus the unique constraint on the assignment, and both are database invariants so they hold through a crash. And `SUSPENDED` is important to name, because a suspended driver with a live `current_trip_id` is a real operational state and the system has to decide what happens to their passenger. Study separately: [[event-types|Event Types]] and [[idempotent-consumer|Idempotent Consumer]].

### 5.4 Rider-side tracking

Interviewer: You said 1.5 million messages per second outbound. That's the biggest number in your design and you spent four minutes on ingest. Defend it.

Candidate: Because it is the biggest number, and because it is the only part of the system that is directly visible to a human. Ingest failing degrades match quality. Fan-out failing means a rider watches a frozen car, and they will rate the driver down and stop using the product.

Candidate: The optimizations, and they are all about bytes and frame rate rather than about cleverness. Delta encoding: send lat and lon as int32 offsets from the last frame the client acknowledged, not as doubles, which takes the payload from about 60 bytes to about 22. Acknowledge-and-window: the client ACKs the last seq, the gateway keeps a 2-second window, and lost frames are never resent because the next frame supersedes them. Route-relative positioning: the rider is on a route, so I send progress along the route polyline, which is a float from 0 to 1 plus a small correction, and the client does the map matching locally. Interpolation: the client renders at 60 frames per second between the 1 hertz samples, so the motion is smooth at a fraction of the bytes.

Candidate: Pre-request map is a different animal and I treat it differently: 0.2 hertz, aggregated to the cell rather than per driver, and pre-rendered on the client from a vector tile. A pre-request map showing 400 individual dots is not a product, it is a heatmap, and rendering it from a tile costs the server nothing. Study separately: [[websockets]], [[tail-latency|Tail Latency]], and [[connection-pooling|Connection Pooling]].

### 5.5 Caching

Interviewer: What do you cache, and where does it go wrong?

Candidate: Three caches, each with a different reason to exist and a different invalidation story.

```
 Cache                    Key                 Value            TTL      Invalidation
 ------------------------ ------------------ --------------- -------  ----------------
 driver profile + digest   driver_id          ~2KB            5 min    stream update
 zone supply/demand        zone_id             counters        1s       stream update
 fare + surge quote        (from_cell,to,tier) fare_cents      30s     on multiplier change
```

Candidate: The driver profile cache exists to take scoring off the database, and it is invalidated by the stream, not by a timer, so it is at most 2 seconds stale, and I do not care because the score is a heuristic. The zone counter cache is 1 second because surge computed on 1-second-old supply is fine and surge computed on 1-minute-old supply is not. The fare quote cache is the one with a real correctness constraint, and it is bounded by the surge multiplier validity window, not by a number I picked, using the rule that cache TTL must never exceed the lifetime of the thing being cached.

Candidate: Now the stampede, and I will answer it as a general pattern since it comes up in every interview.

Candidate: The stampede risk here is a cold zone cell, or a fare TTL expiring across a whole city, or a driver profile cache flush. Four standard defenses. TTL jitter, so a million keys do not expire in the same millisecond. Single-flight, so a million concurrent misses produce one origin read and the rest wait on it. Probabilistic early expiration, so only the first unlucky reader refreshes. And stale-while-revalidate, so a stale value is served instantly while a refresh runs behind, because a 2-second-stale driver profile is a strictly better answer than no answer.

Candidate: The Uber-specific defense is the best one: make the stampede impossible rather than survivable. The whole index is 320 megabytes, so I can hold it in memory on every matching node and there is no cold-miss path to stampede. Scale the dataset up until it no longer fits, and the design is still correct. Study separately: [[caching|Caching]], [[redis|Redis]], and [[cache-warming|Cache Warming]].

### 5.6 Replication, failover, and the ugly failure modes

Interviewer: Traffic just doubled. Peak-hour surge, an event, whatever. Walk me through the first ten minutes.

Candidate: First, where does it hit, in order? Rider and driver app traffic roughly doubles: 1M to 2M location pings per second, and 3M to 6M fan-out messages per second. Match searches go from 2,500 to 5,000 per second. The realtime gateway is the first thing to break, because it holds stateful connections and each connection is a heap and a kernel socket, and 6 million sockets is not a thing you autoscale into.

Candidate: My response, in order. First, the gateway tier scales on live connection count, not on CPU, and the autoscaler uses a custom metric of connections per node with a target of 20,000 per node, so 6 million connections is 300 nodes and that is a scaling decision, not an incident. Second, the fan-out rate degrades: I drop pre-request map updates to 0.1 hertz and hold rider trip updates at 1 hertz, because an on-trip map is worth bytes and a browse map is not. Third, ingest sheds by rate-limit per driver: a hard per-driver cap of one ping per 2 seconds, and idle drivers over the cap are simply skipped, because a dropped idle ping costs nothing for 30 seconds. Fourth, matching gets a shed threshold: if the match queue for a zone exceeds 2 seconds of age, new requests for that zone get a slower, cheaper path using coarser cells and pure-distance ranking instead of the model.

Candidate: And what I would deliberately not do: I would not degrade correctness. No partial trip writes, no best-effort money, no dropping the CAS. Under 2x load the system gets slower and less smooth, and it stays correct. The moment I let it become incorrect is the moment I have lost the ledger reconciliation argument forever.

Interviewer: The primary trip database dies.

Candidate: The trip database is sharded, 256 physical shards, so first question: is one shard down or all of them. A single shard is a contained incident: rides whose trip hashes to that shard cannot be created, which in a 256-way sharded world is 0.4 percent of new rides failing, and I can shed that deliberately with a clear error rather than timeouts.

Candidate: All shards down, or the primary for a shard dying, is the real case. Detection is a failed health check plus a write-failure rate alarm, and I want that under 10 seconds. Then: promote a replica. Because replication is synchronous to one replica in another availability zone and asynchronous to the rest, there may be committed transactions that the promoted replica does not have, so the promotion is a split-brain risk, and the rule is to fence the old primary with a lease expiry before promoting, not after. Fencing, not hoping.

Candidate: The RTO I would commit to is 60 seconds for the match path, and I would get there by not depending on the trip database in the first 200 milliseconds of matching. Let me be explicit about what matching needs: the rider's request needs a trip row and a payment hold, and that is the only trip-database read in the hot path. Everything after that, the cell index, the driver CAS, the surge, the ETA, is in the in-memory plane. So the database outage costs me trip creation, not matching of already-created trips. That is a deliberately narrow blast radius and I built it that way on purpose.

Candidate: RPO on the trip database is approximately zero for committed writes, thanks to synchronous replication to the second replica. RTO is 60 to 90 seconds including fencing. And critically, the money ledger is a separate database with its own failover, because losing the ledger and losing the trip record are different incidents and coupling their recovery couples their risk.

Candidate: During promotion, all writes fail fast, no retries, because a retry storm against a database that is being promoted is how you turn a 60-second outage into a 10-minute one. Study separately: [[failover|Failover]], [[split-brain|Split Brain]], and [[disaster-recovery|Disaster Recovery]].

Interviewer: Replica lag. You said the driver state lives in a store. What if the read you're using for verification is a lagging replica, and it says the driver is AVAILABLE when they're not?

Candidate: Then I have exactly the bug the system exists to prevent, and it is the highest-severity bug in this system because it produces a double assignment, which is a rider with two drivers, which is a support incident and a payout dispute.

Candidate: The defense is to never verify against a replica. Verification of driver availability happens on the same primary that performs the CAS, inside the same statement. The read and the write are one operation. There is no window because there is no round trip.

Candidate: The second defense is the unique constraint. Even if some path somehow proposed a bad assignment, the assignment write itself carries a partial unique index on `driver_id` where `driver_id IS NOT NULL AND status IN ('ASSIGNED','ARRIVING','IN_PROGRESS')`, and that index is enforced by every replica-independent write path. So a double assignment is rejected by the database even if the application logic is wrong, which is the difference between a bug and a data-integrity incident.

Candidate: The third is a reconciliation job that compares the driver's `current_trip_id` against the trips table every minute and pages on any mismatch, because a bug you detect in 60 seconds is an incident and a bug you detect in 30 days is an obituary. Study separately: [[replication-lag|Replication Lag]] and [[exactly-once-effect|Exactly Once Effect]].

Interviewer: The driver profile cache. Five million entries, TTL five minutes, stream-invalidated. Walk me through the stampede.

Candidate: It cannot stampede, and I want to explain why rather than just assert it, because the general pattern matters more than this instance. The dataset is 320 to 800 megabytes. Every matching node holds it in memory in full. There is no miss path, so there is nothing to stampede. The protection is that the dataset is small enough to be a non-problem, and I only reach for TTL jitter and single-flight if the dataset ever grows past a single machine's memory.

Candidate: Where stampedes do bite in this design is the surge multiplier, because it is a global-ish key that changes for a whole city at once. When the multiplier for a zone changes, every fare-quote cache entry for that zone becomes invalid simultaneously, and that is a designed mass expiration. Defenses here: version the surge value, so a quote carries the multiplier version it was priced with, and a quote is valid while that version is current, so a multiplier change simply makes old quotes invalid rather than requiring a sweep. And cap the multiplier change rate, so surge moves in steps at most every 30 seconds, which turns a mass invalidation into a controlled trickle. Study separately: [[cache-warming|Cache Warming]] and [[rate-limiter|Rate Limiter]].

### 5.7 GPS accuracy versus update rate

Interviewer: Last technical pushback. Someone on your team says: double the location update rate to 1 hertz for every driver, always. It's smoother, ETAs are better, riders are happier. Ship it.

Candidate: I would not ship it, and I want to show the arithmetic rather than appeal to taste.

Candidate: Going from the weighted 500,000 pings per second to uniform 1 hertz is 5 million per second. That is 10x ingest: 5 million times 80 bytes is 400 megabytes per second, about 34 terabytes per day, roughly 3 times the entire production database footprint per day of raw event volume. Then the fan-out consequence, which people forget: matching consumes the index, and a 10x index write rate does not buy 10x accuracy, it buys 2x accuracy, because a driver at highway speed at 4 seconds moves 44 meters, and at 1 second moves 11 meters, but the dominant error in a GPS fix indoors or in an urban canyon is 10 to 30 meters regardless of how often you ask. I would be paying 10x the cost to move the error from 44 meters to 11 meters, and the remaining 11 meters is buried under a 10-meter measurement error.

Candidate: So the answer is adaptive rate, not a single number. Moving on a trip: 2 to 4 seconds, because the rider is watching. Moving with an active offer or approaching a pickup: 1 to 2 seconds, because ETAs are being computed from it and the rider is looking at the map with intent. Idle in a dense cell: 10 seconds, because the rider's pre-request map can interpolate. Idle in a sparse area: 30 seconds, because there is no demand to be missed. And a full 1 hertz, but only for a window: when a driver is within 200 meters of a pickup, or when a trip is about to start, or during a dispute replay. I call this the last 200 meters rule, and I have seen it earn more rider satisfaction than uniform 1 hertz ever did.

Candidate: The client-side lever is the same idea: dead reckoning. The app knows the last two fixes, the speed, and the heading, so it can extrapolate the position between fixes and correct on the next one. That makes the map smooth at 0.2 hertz ingest, and it moves the smoothness requirement out of the network and into the client, where the frames are free.

Candidate: And the honest limitation: dead reckoning is wrong at intersections and in tunnels, so it needs to be damped when the GPS accuracy field degrades, and I would never let a dead-reckoned position feed back into the server's index. Extrapolation is a rendering concern; the index wants the last real fix. That is a boundary most real implementations get wrong, and stating it is worth more than the optimization itself. Study separately: [[bottleneck-identification|Bottleneck Identification]] and [[cost-estimation|Cost Estimation]].

## Phase 6 — Closing (minutes 42-45)

Interviewer: Wrap up. If I remember one page of this, what should it be?

Candidate: Four things.

Candidate: One, the shape of the system. Everything high-volume and lossy-tolerable, which is location, the geospatial index, and the rider fan-out, is memory and a log. Everything low-volume and correctness-critical, which is the trip lifecycle and the ledger, is a strongly consistent transactional store. Putting a database write in front of the index update would spend the entire latency budget on one row, and the interviewer's instinct to look for the database hotspot is backwards here, because the database is the easy part.

Candidate: Two, the numbers that size it. 500,000 location pings per second inbound, 1.5 million fan-out messages per second outbound, 2,500 match searches per second, 7,000 transactional writes per second. The fan-out is three times the ingest and that is the thing people forget.

Candidate: Three, the invariants. At most one trip per driver and one driver per trip, enforced by a single compare-and-set on the primary and backed by a partial unique index, so the invariant is a database property rather than an application convention. And that is why the answer to "what if the primary is down" is "matching does not care, because matching does not read the trip database."

Candidate: Four, the failure modes I would actually lose sleep over, and note that none of them are the ones the problem statement suggests. Not the database, which is small and replicated. Not ingest, which is elastic and lossy by design. The three I would page for are: a hot cell turning match latency into a per-zone incident, which needs adaptive subdivision; a stale driver-state read turning into a double assignment, which needs the CAS-plus-unique-index discipline; and the realtime gateway failing under connection-count growth, which needs the tier to scale on connections and not on CPU.

Candidate: And the one trade-off I would most want to defend if I only had a minute: accuracy versus update rate. Adaptive rate with a last-200-meters full-rate window beats uniform 1 hertz by an order of magnitude in cost for a difference in rider-perceived quality that barely registers. Study separately: [[cell-based-architecture|Cell-Based Architecture]], [[tail-latency|Tail Latency]], and [[graceful-degradation|Graceful Degradation]].

Interviewer: That's a good session. You asked about the offer exclusivity, which is the question that determines whether the matching engine is a search or a distributed lock, and almost nobody asked it. Thank you.

## Concepts This Transcript Drills

- Location heartbeat ingest, backpressure, and lossy-by-design streaming: [[backpressure]], [[kafka-architecture|Kafka Architecture]], [[kafka-producers-consumers|Kafka Producers and Consumers]]
- The hierarchical geospatial index and ring search: [[database-indexing|Database Indexing]], [[shard-routing|Shard Routing]]
- Match quality, batching, and assignment: [[search-ranking|Search Ranking]], [[distributed-scheduling|Distributed Scheduling]]
- Mutual exclusion for driver assignment: [[distributed-locks|Distributed Locks]]
- The trip and driver state machines, and the outbox that drives them: [[event-driven-architecture|Event Driven Architecture]], [[outbox-pattern|Outbox Pattern]]
- Hot-region mitigation: [[hotspot-handling|Hotspot Handling]], [[cell-based-architecture|Cell-Based Architecture]]
- Rider-side realtime fan-out: [[websockets]], [[tail-latency|Tail Latency]]
- Primary death, replica lag, and fencing: [[failover|Failover]], [[replication-lag|Replication Lag]], [[split-brain|Split Brain]]
- Location history at scale: [[time-series-at-scale|Time Series at Scale]], [[data-warehouse-lake|Data Warehouse / Lake]]
