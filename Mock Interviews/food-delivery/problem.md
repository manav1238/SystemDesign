---
title: "Design a Food Delivery Platform (study transcript)"
status: active
tags: [hld, mock, food-delivery]
---

# Design a Food Delivery Platform — Problem Statement

## Problem Statement

Design a food delivery marketplace similar to DoorDash or Uber Eats, connecting customers, restaurants, and independent delivery partners.

**Core product surface:**

- Three separate applications: a customer app for browsing, carting, and tracking; a restaurant app for managing the menu and the incoming order queue; and a delivery-partner app for accepting jobs and navigating.
- A customer browses restaurants by cuisine, distance, rating, price, and delivery time, opens a restaurant menu, adds items with modifiers, and checks out.
- An order flows: cart, payment, restaurant acceptance, kitchen preparation, ready for pickup, partner assignment, pickup, delivery to the customer, and delivery confirmation.
- Restaurants define a menu with items, modifiers, add-ons, and per-item availability, and they can be busy or sold out on individual items without taking the whole menu down.
- Restaurants set a delivery radius, usually 3 to 8 kilometers, and a customer outside that radius cannot order.
- The platform assigns a delivery partner to each ready order, weighing distance to the restaurant, current partner load, on-time rate, and vehicle type.
- Delivery fees vary with cart size, distance, demand, and time of day. Small orders carry a small-order fee. High-demand periods and popular restaurants attract surge.
- Notifications are multi-channel: push, SMS, in-app, and sometimes email, each with its own fallback chain and rate limits.
- Order history, receipts, and payouts exist for all three parties, and the financial ledger must balance.

**The interviewer says:**

> "Assume 80 million monthly active customers, 800,000 active restaurants, 1.5 million delivery partners online at peak, and 20 million orders per day. Design the system so that the customer sees a status change within two seconds, a restaurant sees a new order within three seconds, a restaurant that ignores an order for seven minutes is auto-rejected and refunded without a human, and no customer is ever charged twice for one order. Show me the order flow, the state machine, the dispatch, the inventory, the payment correctness story, and the sharding keys. Then tell me what breaks."

## Clarifying Questions You Should Ask

Ask these before you draw a single box. The payment-capture-timing and the partner-exclusivity questions are the ones that separate a prepared candidate.

### Scope and the three apps

1. Do all three clients hit the same backend, or separate backends per app? The customer, restaurant, and partner apps have very different screen shapes and very different read volumes, which argues for separate backend-for-frontend layers.
2. Is the restaurant menu a simple list, or does it support modifiers, add-on groups, size variants, and time-limited items? Modifier modeling is where menu schemas get complicated.
3. Are delivery partners independent contractors on their own accounts, or employees? This decides whether partner earnings are a payout statement or a payroll record, and it changes the regulatory shape.
4. Is customer-to-restaurant ordering only, or do we also need customer-to-customer, scheduled orders, group orders, and catering pre-orders?
5. Is a customer able to reorder a past order, and if so, does that go through a cart and reprice, or is it a one-tap clone? It affects the cart service design.
6. Are tips in scope? Tips are the most common source of ledger reconciliation bugs because they arrive late, are customer-initiated, and are split between restaurant and platform.

### Payments and money

7. When is the customer's card charged: at checkout, at restaurant acceptance, or at delivery? And is it authorized and captured, or captured outright? This single question determines the entire refund and failure story, and I would push hard on it before anything else.
8. Does the platform hold funds between restaurant acceptance and delivery, and if so, when are funds released to the restaurant and to the partner?
9. What happens if payment succeeds but the restaurant never accepts? Auto-reject, refund, and how long the hold stays.
10. Do we need split settlement, where one order pays the restaurant 70 percent, the partner 20 percent, and the platform 10 percent, with each party seeing their own statement derived from the same ledger?
11. Is there promo code, wallet credit, and gift card balance, and do those need to be reserved atomically with the card authorization?
12. Are we the merchant of record, or is that the restaurant? It decides who absorbs a chargeback.

### Inventory and menu

13. Is inventory per item, per modifier, or just a boolean sold-out flag? And do restaurants have a real stock count, or only a toggle?
14. Can a restaurant accept an order for an item it has run out of, or must checkout fail? The first is a real operational need and it means oversell is expected and must be compensated.
15. Does the menu change mid-order? If a price changes while a customer is checking out, which price wins, and does that need to be locked at cart-add or at checkout?
16. Is there scheduled availability, such as breakfast items only from 6am to 11am, and seasonal menus?
17. Do we need ingredient-level stock, where a sold-out item should auto-hide items that share an ingredient? Most platforms do not, and it is worth saying why.

### Dispatch and delivery

18. When a partner is offered a job, is the offer exclusive, and can a partner hold two pending offers at once? This decides whether dispatch is a queue or a compare-and-set.
19. How long does a partner have to accept, and what is the offer expiry?
20. Can one partner carry multiple orders if they are going to the same restaurant, and does batching happen at the restaurant or in the algorithm?
21. What happens when a partner's app crashes mid-delivery? Is the order reassigned, and is the customer told?
22. Can a customer choose a delivery window, such as "arrive between 6pm and 7pm", and does that change dispatch to a reservation model?
23. Does the platform do its own routing and ETAs, or call an external maps provider? Both are realistic and the answer changes the ETA service design.
24. Is delivery radius a fixed radius or a real polygon, and who draws it?

### Non-functional and scale

25. What is the target latency from order placed to visible in the restaurant app, and separately to visible in the customer app? These are different numbers with different budgets.
26. What is the auto-reject window, and is it a hard requirement that the customer is refunded without human action?
27. Is peak traffic concentrated, like a dinner rush where 15 percent of orders land in one hour, and are there flash events such as a televised sports final where volume goes up 5x in two minutes?
28. Is there a hard availability target on the order-creation path, and a different one on menu browsing?
29. Is the platform expected to run multi-region, and does a restaurant's location determine its home region?
30. Is there a cost ceiling per order, and how much of it is the delivery subsidy?

## What You Are Evaluated On

The interview is scored on five phases, matching [[06-hld-interview-checklist|HLD Interview Checklist]].

### Phase 1 — Requirements Clarification

Did you ask when the card is charged, and did you use the answer to build the failure story rather than just noting it? Did you ask about partner offer exclusivity, which decides whether dispatch is a queue or a compare-and-set? Did you establish that money is a double-entry ledger and not a field on the order, which sets the consistency bar for everything? See [[functional-vs-non-functional-requirements|Functional vs Non-Functional Requirements]] and [[idempotency|Idempotency]].

### Phase 2 — Estimation

Did you estimate orders per second at peak, not per day, and split the traffic into four classes, catalog reads, order writes, partner location pings, and customer tracking fan-out, because they differ by two orders of magnitude? Did you size the concurrent tracking socket count, which turns out to be the same order of magnitude as the number of in-flight deliveries? See [[capacity-estimation|Capacity Estimation]] and [[latency-budget|Latency Budget]].

### Phase 3 — High-Level Architecture

Did you draw three distinct backend-for-frontend layers for the three apps, and justify that split by read volume and by client release cadence rather than by saying "microservices"? Did you keep the order, payment, inventory, catalog, dispatch, location, and notification concerns as separate scale units? Did you place the notification and inventory updates off the order's synchronous path entirely? See [[backend-for-frontend|Backend for Frontend]], [[microservices|Microservices]], and [[service-oriented-architecture|Service Oriented Architecture]].

### Phase 4 — Deep Dive

Did you go deep on at least five of: the order state machine and its legal transitions, the dispatch algorithm and its hot-restaurant behavior, inventory as a reservation ledger rather than a decrementing counter, the payment correctness story with authorize/capture/refund and idempotency, the dual-write problem and the outbox that solves it, the multi-channel notification service, delivery-radius geofencing, and ETA prediction? See [[outbox-pattern|Outbox Pattern]], [[event-driven-architecture|Event Driven Architecture]], and [[saga-and-strangler|Saga and Strangler]].

### Phase 5 — Trade-offs and Failure Scenarios

Did you name the cost of every decision, especially the reserve-then-confirm inventory model versus a simple counter, and the batch-versus-greedy dispatch trade-off? Did you walk through concrete failures: a 5x dinner-rush spike, primary order database death, a sold-out bestseller hot key, replica lag on partner state causing a double assignment, a cache stampede on a promoted restaurant's menu, a restaurant app that is down during service, a duplicate order from a double tap, and a payment webhook arriving ten minutes late? See [[trade-off-analysis|Trade-Off Analysis]], [[replication-lag|Replication Lag]], and [[graceful-degradation|Graceful Degradation]].

## Related Reading

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the skeleton this problem is scored against
- [[01-rapid-revision|Rapid Revision]] — one-liners per concept for the day before
- [[outbox-pattern|Outbox Pattern]] — the dual-write fix
- [[idempotency|Idempotency]] — order and payment correctness
- [[event-driven-architecture|Event Driven Architecture]] — order state as events
