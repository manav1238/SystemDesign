---
title: "Design Uber (study transcript)"
status: active
tags: [hld, mock, uber]
---

# Design Uber — Problem Statement

## Problem Statement

Design a ride-hailing platform similar to Uber: riders request a ride, the system finds a nearby driver, the driver accepts, the trip runs, and both sides see each other move in real time.

**Core product surface:**

- Riders open the app, see nearby drivers on a map as live dots, request a ride, watch the driver approach, share the trip with a friend, and see fare, ETA, and route.
- Drivers have a driver app: go online and off line, see incoming ride requests, accept or decline within a time window, navigate to pickup, and see earnings, including surge incentives and weekly payout statements.
- The platform continuously ingests GPS location from millions of devices and serves "find nearby drivers" queries at very low latency, in a fixed radius, over a moving map.
- Matching must score candidate drivers on multiple dimensions: pickup ETA, straight-line distance, driver rating, vehicle class, and surge multiplier eligibility.
- Surge pricing adjusts the fare multiplier by zone and time based on real-time supply and demand.
- Every trip is a state machine with money attached: fare, surge, tolls, cancellation fees, driver payout, and platform commission. Money must balance.
- Riders and drivers both have trip history: past trips, ratings given and received, receipts, and tax documents.
- Notifications are push, plus SMS for driver requests and critical rider events.
- The system is city-scoped but global, with wildly uneven density: Manhattan at 6pm on a Friday versus a suburb at 4am.

**The interviewer says:**

> "Assume 100 million monthly active users, 5 million drivers online at peak, and 25 million trips per day. Design the system so that a location update travels from a driver's phone to a rider's map in under a second, a ride request is matched in under a second at the 99th percentile, the system survives a burst of 2x peak traffic, and no driver is ever assigned to two riders at the same time. Show me the ingest path, the geospatial index, the matching engine, the trip state machine, and the sharding keys. Then tell me what breaks."

## Clarifying Questions You Should Ask

Ask these before you draw a single box. The geospatial and heartbeat questions are the ones that separate a prepared candidate.

### Scope and product surface

1. Is this ride-hailing only, or do we also need food delivery, courier, and transit products on the same location infrastructure? Uber's real complexity comes from several products sharing one location and dispatch platform.
2. Is the driver app ours, or a third-party SDK? Do we control the sampling rate, the batching, and the retry behavior of location reporting? This decides whether we can adapt the update frequency ourselves.
3. Is the rider-side live map in scope, including the animated driver marker and the ETA recalculation, or only the trip state?
4. Are scheduled and recurring rides in scope, such as a 7am airport run every weekday? That is a scheduler, not a request.
5. Is driver earnings, payouts, and the financial ledger in scope, or only the trip record? If money is in scope, the consistency bar is completely different.
6. Are pool rides, where multiple riders share a car, in scope? Pools change matching from one-to-one to one-to-many and change the state machine.
7. Is surge pricing computation in scope, or is the multiplier given to us?

### Location and the device edge

8. How fresh does a driver's location need to be on the rider's map, and how fresh in the matching index? These are two different freshness requirements and conflating them is a common mistake.
9. Does the driver report a fixed interval, or can we make the interval adaptive based on speed and trip state? Adaptive rate changes the entire cost model.
10. What is the accuracy requirement? 5-meter accuracy at 1Hz across 5 million drivers is a fundamentally different bill from 20-meter accuracy at 0.2Hz.
11. Do we need location history for trip replay and route reconstruction, or only the latest position? History is 100x the storage of latest-only.
12. Do we ingest raw device pings, or a derived "speed and heading" signal? Do we compute the map matching, the snapping to road, on device or server?

### Matching

14. Is matching first-come-first-served, or is it an optimization that batches riders in a zone and solves for a global assignment? Uber's answer is batch-and-optimize, and the interviewer wants to hear you find it.
15. Does a rejected or timed-out driver request roll to the next driver immediately, and how many candidates do we try before declaring no match?
16. Must a driver be offered at most one pending request at a time, or can a driver hold several and choose? The first is a mutual-exclusion problem; the second is a much easier system.
17. How long does a driver have to accept before the offer expires? That number sets the retry budget.
18. Do driver preferences matter, such as a driver who does not want airport runs, or a rider who only accepts a specific vehicle class?
19. Is the surge multiplier part of the matching score, such that a driver in a high-surge zone is more attractive to the algorithm?

### Money and correctness

20. Is there a real payment ledger with double-entry accounting, refunds, and reconciliation against the processor, or is the trip fare just a number we report?
21. When a rider cancels after the driver has arrived, who is owed what, and when is the cancellation fee assessed? This is the hardest state transition in the product.
22. Do we need to handle the case where the trip record is written but the driver's payout is not? Which is authoritative?

### Non-functional and scale

23. What is the target match latency at p99, and does the requirement differ between a dense city center and a suburb?
24. What availability do we demand for matching, given that every failed match is lost revenue, and what availability do we demand for trip history, given that it is read once a month?
25. Is the system single-region-per-city, or must a trip be resilient to a whole-region failure with active trips in flight?
26. Are we allowed a hard cap on how far we will search for a driver, and what happens if no driver is found inside the cap?
27. What is the cost ceiling per location update? Location ingest at 500,000 messages per second is a line item, and a candidate who treats it as free has not done the arithmetic.
28. Is there data residency to satisfy, such as India, Russia, or the EU, that constrains where location data physically lives?

## What You Are Evaluated On

The interview is scored on five phases, matching [[06-hld-interview-checklist|HLD Interview Checklist]].

### Phase 1 — Requirements Clarification

Did you separate the two freshness requirements, rider-map latency versus matching-index latency, before estimating? Did you ask whether a driver can hold multiple pending offers, because that single answer changes the matching engine from a mutual-exclusion problem into a queue? Did you establish that money is a ledger and not a number, which sets the consistency bar for the trip service? See [[functional-vs-non-functional-requirements|Functional vs Non-Functional Requirements]] and [[latency-vs-throughput|Latency vs Throughput]].

### Phase 2 — Estimation

Did you derive location ingest rate from online drivers and reporting interval, and split on-trip versus idle drivers, rather than picking a number? Did you find the 5 million concurrent rider sockets and the resulting 1 million messages per second outbound, which is larger than the ingest and is the real cost centre? Did you size the geospatial index cells and the match-search fan-out? See [[capacity-estimation|Capacity Estimation]], [[server-capacity|Server Capacity]], and [[latency-budget|Latency Budget]].

### Phase 3 — High-Level Architecture

Did you draw a distinct location ingest tier, a geospatial index tier, a matching service, a trip service, and a realtime gateway for rider sockets, as separate scale units? Did you make the realtime fan-out to riders visible rather than hiding it inside a "WebSocket service" box? Did you place the trip database as a strongly consistent store separate from the eventually consistent location store? See [[websockets]], [[microservices|Microservices]], and [[control-plane-vs-data-plane|Control Plane vs Data Plane]].

### Phase 4 — Deep Dive

Did you go deep on at least five of: the location heartbeat and ingest pipeline with backpressure, the geospatial index and the ring search, the matching engine and the per-driver mutual exclusion, the trip and driver state machines, hot-region handling, rider-side live tracking with interpolation and delta encoding, surge computation, and trip history for both parties? Did you reason about consistency per operation, strongly consistent for assignment and money, eventually consistent for location and history? See [[strong-vs-eventual-consistency|Strong vs Eventual Consistency]], [[hotspot-handling|Hotspot Handling]], and [[time-series-at-scale|Time Series at Scale]].

### Phase 5 — Trade-offs and Failure Scenarios

Did you name the cost of every decision, especially the GPS-accuracy-versus-update-rate trade-off with real arithmetic? Did you walk through concrete failures: 2x traffic spike, primary trip database death, a single hot cell at Times Square, replica lag on driver state causing a double assignment, a cache stampede on driver profiles, a driver's phone going offline mid-offer, and a location ingest backlog? Did you close by naming the two or three things that would actually break first, which are not the things a naive design would predict? See [[trade-off-analysis|Trade-Off Analysis]], [[replication-lag|Replication Lag]], and [[graceful-degradation|Graceful Degradation]].

## Related Reading

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the skeleton this problem is scored against
- [[01-rapid-revision|Rapid Revision]] — one-liners per concept for the day before
- [[websockets]] — the rider-side realtime connection
- [[shard-routing|Shard Routing]] — location-index sharding by geocell
- [[distributed-locks|Distributed Locks]] — the one-driver-one-rider invariant
