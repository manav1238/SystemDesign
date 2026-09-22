---
title: Failover
category: Database
priority: must-know
status: learning
difficulty: medium
interview_ready: false
tags:
  - hld
  - database
  - failover
  - availability
---

# Failover

## 1. One-Line Definition
Failover is the automated process of detecting a primary/leader failure and switching service to a standby node so reads and writes continue without an outage.

## 2. Why Do We Need It?
Nodes die — it's a statistical certainty, not a hypothetical. Unless a standby automatically takes over, every node loss is an outage and the RTO is "however long until a human acts." Failover turns minutes-to-hours of human downtime into seconds of automated switchover.

## 3. Simple Intuition
Two generators in a hospital: if the main generator dies, a transfer switch spins up the backup automatically within seconds — the lights barely flicker. Without the switch, a human must walk outside, diagnose, and start the second generator; the ER goes dark meanwhile.

## 4. What Happens Without It?
Primary DB dies → writers fail → users see errors → a pager fires → an engineer SSHes in at 3am, checks health, promotes a replica, re-points the fleet, waits for DNS/config propagation — 20-60+ minutes of downtime, anxiety, and manual-file-error risk in every step.

## 5. Core Idea
The failover loop:
1. **Detection:** health checks / heartbeats / leader-election signals. Caveat: false positives cause spurious failover; tune timeouts.
2. **Decision:** who is the new primary — via quorum/fencing so **split-brain is prevented** (only one leader ever).
3. **Promotion + fencing:** quiesce the old node (fence it — stop it from writing), promote the freshest follower.
4. **Repoint:** app connections / service discovery swerve to the new primary.
5. **Recovery:** old node rebuilds as a follower of the new primary (resync), fleet settings verified.

Key design factors:
- **Detection timeout** → trade: fast failover (more false positives) vs slow (longer outage).
- **Quorum:** majority agreement needed to pick a leader; without quorum the system refuses to act (no split brain) → availability during partition.
- **RPO:** with async replication, promotion can lose the unsynced tail — set expectations (capture point).
- **App re-point:** DNS (slow, TTL-bound), VIP/floating IP, or service-discovery (fast) — plan the "how apps find the primary" story.

## 6. Important Terminology

| Term | Simple Meaning |
|------|----------------|
| Primary / leader | Current writer |
| Standby / replica | Waiting to become primary |
| Promotion | Upgrading a standby to primary |
| Fencing | Preventing the old primary from causing havoc (stops writes) |
| Quorum | Majority of nodes agreeing on the leader |
| Split brain | Two leaders at once (must never happen) |
| RTO | Time to restore service |
| RPO | Data loss window on failover |
| VIP / service discovery | How apps learn the new primary |

## 7. Basic Architecture

```mermaid
flowchart LR
    App --> VIP[Primary/Standby pair (VIP)]
    VIP --> P[(Primary)]
    P -. repl .-> S[(Standby)]
    Monitor[Hearbeat/quorum] -. promote on failure .-> S
```

## 8. Request or Data Flow
1. Heartbeat/quorum pings every node.
2. Primary dies (or loses quorum) → monitor decides "leadership lost."
3. Fence the old primary (isolate/stonith); promote the freshest standby.
4. VIP/re-discovery re-points apps → writes flow to new primary.
5. Old node returns → rebuilt as follower, without a leading role.

## 9. Practical Example
**Primary DB for a payments API (assumptions):** RTO < 60s, RPO 0 (semi-sync to one standby; async elsewhere).
- 1 primary + 1 synchronous standby + 2 async replicas, all in a quorum cluster.
- Primary dies → standby has every commit (semi-sync) → promoted with RPO 0 → VIP moves (seconds) → apps reconnect.
- Announced failover tests: quarterly DR drill validates the loop end-to-end (see disaster-recovery).

## 10. Scaling
- Failover is orthogonal to capacity but interacts: a cluster keeps N nodes; promotion works for any N ≥ 2 (same or larger size). Don't scale *availability* by scaling *capacity* — a bigger primary dies just as easily.
- Multi-AZ + multi-region: failover across AZ (fast, sub-second) vs across region (seconds/minutes + RPO story) — see rpo-rto, multi-region.

## 11. Reliability and Failure Scenarios

| Failure | What Happens | Detection | Recovery | Trade-off |
|---------|--------------|-----------|----------|-----------|
| Primary crash | Writes stop | Heartbeat timeout | Promote standby | Timeout vs false positive |
| Network partition | Split-brain risk | Lost quorum | Only quorum side proceeds | Availability down during split |
| False positive | Unnecessary failover | Spurious promotion | Fail back carefully | Detection tuning |
| Async loss | Post-ack data gone | RPO gap detected | Reconcile/backfill if possible | strict async window |
| Both fail | No standby | Cluster degraded | Backup + manual restore | multi-replica redundancy |

## 12. Consistency and Correctness
Failover *itself* must be consistent: never two leaders (split-brain double-writes corrupt data). Use quorum + fencing. Understand the **ack-vs-durable** boundary: sync ack means acked = present on the standby; async ack may be lost. Set RPO honestly and never "assume zero" without the sync/echo proof.

## 13. Performance
- Detection timeout adds to RTO (tune against false-positive risk).
- Sync replication per write adds follower-ack latency (the price of RPO 0).
- Re-point latency (VIP vs DNS) adds to RTO; DNS with long TTL turns "60s failover" into minutes.

## 14. Security
- Fencing must be bulletproof (the old primary must not be able to write after replacement — stonith/power-off, credentials revocation).
- Replication links use TLS + mTLS; readiness checks must not be spoofable externally.

## 15. Trade-Offs

| Choice | Advantages | Disadvantages | When to Use |
|--------|------------|---------------|-------------|
| Fast failover (short timeout) | Low RTO | False-positive flips | Sensitive latency service |
| Slow failover (long timeout) | Fewer spurious | Long RTO | Stable infra, tolerance |
| Quorum required | No split brain | Down during partition | Correctness-critical |
| Majority-only | prevents split-brain | Can't write N/2 dead | Standard |
| Async promotion | Fast, any follower | RPO > 0 | Read-tolerant data |

## 16. Common Mistakes
- "We have replicas so we have failover" — replicas without promotion automation are just capacity.
- Split-brain-enabled automatic failover (no quorum/fencing) → silent data corruption; often worse than downtime.
- DNS-based re-pointing with long TTL → failover of minutes.
- Never testing failover; the first rehearsal is the outage.

## 17. HLD vs LLD Boundary
HLD: failover topology/mode, quorum/fencing policy, RPO/RTO, detection strategy, re-pointing mechanism. LLD: the specific health-check handler, client reconnect behavior in a service.

## 18. Interview Questions

### Beginner
- What is failover and why is detection important?
- What is split-brain and how is it prevented?

### Intermediate
- A DB dies; what exactly happens between "die" and "writes flowing again"?
- Async replication + failover: what's the RPO? How do you minimize it?

### Advanced
- Design a cluster that survives the primary, the whole AZ, and a region failing one after another, keeping RPO close to zero.
- How do you fail *back* safely after the old primary returns?

## 19. Interview Memory Summary

> [!abstract]- Interview Memory Summary

### Remember

- Failover loop: detect → promote → repoint → recover.
- Quorum + fencing = no split brain (never two leaders).
- RPO comes from async replication; RTO from detection + repoint speed.
- Test it — quarterly drills, not first-rehearsal-in-an-outage.
- DNS fails slowly; use VIP/floating-IP/service-discovery.
- Detection timeout trades false positives against longer outages.

### 30-Second Explanation

Heartbeat/quorum detects → fence old → promote standby → repoint via VIP/discovery; tune detection timeout, define RPO/RTO, rehearse.

### Interview Traps

- "We have replicas so we have failover" — replicas without promotion automation are just capacity.
- Failover without quorum + fencing → silent split-brain corruption (worse than downtime).
- DNS-based re-pointing with long TTL → failover of minutes.
- Never testing failover; the first rehearsal is the outage.

### Key Trade-Off

Automated failover trades availability (seconds of switch-over) against complexity and risk: fast detection cuts RTO but raises false-positive flips, and skipping quorum + fencing buys apparent simplicity at the price of potential split-brain corruption.

## 20. Related Concepts

### Prerequisites

- [[database-replication|Database Replication]]
- [[availability|Availability]]

### Commonly Used Together

- [[rpo-rto|RPO and RTO]]
- [[standby-models|Standby Models]]
- [[disaster-recovery|Disaster Recovery]]

### Advanced Concepts

- [[cap-theorem|CAP Theorem]]

Related planned topics (not authored yet): split-brain.

## 21. References
Postgres/MySQL HA docs (Patroni, orchestrator), cloud managed-DB HA documentation. Verify current sem-sync/quorum behavior with provider docs.

## 22. Active Recall

Test yourself before revealing each answer.

> [!question]- What is failover and why is detection important?
> Failover is the automated process of detecting a primary failure and switching service to a standby so reads/writes continue. Detection matters because its timeout sets the trade: fast detection = low RTO but false-positive flips; slow detection = fewer spurious fails but a longer outage. It's minutes-to-hours of human downtime vs seconds of switchover.

> [!question]- What is split-brain and how is it prevented?
> Split-brain is two leaders at once, which double-writes and corrupts data. It's prevented with **quorum** (only a majority can elect a leader, so a minority side refuses to proceed) plus **fencing** (isolate/stonith the old primary so it can't write after replacement).

> [!question]- Design decision: a payments primary with RTO < 60s, RPO 0. What's the topology?
> 1 primary + 1 synchronous standby (RPO 0: the standby has every commit before ack) + 2 async replicas, in a quorum cluster. On primary death: fence the old node, promote the semi-sync standby, move the VIP so apps reconnect in seconds. Quarterly DR drills validate the loop.

> [!question]- Trade-off: short detection timeout vs long timeout for failover.
> Short timeouts catch failures fast (low RTO) but raise false positive flips — an unnecessary failover has its own risk window. Long timeouts avoid spurious switches but stretch the outage. Tune the timeout against your false-positive tolerance and the RTO you're committed to.

> [!question]- Failure scenario: async replication + leader death. What's the RPO and how do you minimize it?
> With async, promotion may lose the unsynced write tail (RPO > 0) — acked-but-unshipped writes are gone. Minimize it: use semi-sync for the promotion path (freshest standby guarantees the tail), set honest RPO expectations, and reconcile/backfill if the old node recovers.

> [!question]- Interview scenario: "We have replicas, so we have automatic failover." Respond.
> Replicas without promotion automation are just capacity — someone must still detect, fence, promote, and re-point. Automatic failover requires quorum + fencing (split-brain protection), a re-point mechanism (VIP/service discovery, not long-TTL DNS), and rehearsal. "We have replicas" ≠ "we have failover."

> [!question]- Interview scenario: how do you fail *back* safely after the old primary returns?
> Rebuild the returned node as a follower of the new primary (resync from snapshot + catch up) before ever letting it lead again. Verify replication health, update the fleet view, and only then consider it for promotion if needed — never re-promote it blindly.

## 23. When Should I Use This?

### Use it when

- Node/primary failure is probabilistically certain and outages are expensive.
- You need RTO in seconds/minutes, not "until a human acts."
- You can automate detection + promotion with quorum + fencing.
- You have a re-point mechanism (VIP/discovery) and can rehearse.

### Avoid it when

- You can't prevent split-brain (no quorum/fencing) — risk of silent corruption.
- Re-pointing depends on long-TTL DNS (failover becomes minutes).
- You won't test it — the first rehearsal can't double as the outage.
- Single-node simplicity matters more than availability (then no replicas needed).

### What problem does it solve?

Problem: nodes die, and without a standby the outage lasts until a human acts. Bottleneck: manual detect/promote/repoint at 3am is minutes-to-hours of downtime and error-prone. Solution: automated failover — heartbeat/quorum detects, fencing isolates the old leader, standby promotes, and VIP/discovery re-points the fleet in seconds.

### What problem does it NOT solve?

It doesn't prevent data loss with async replication (RPO still > 0 unless sync/semi-sync), doesn't cure split-brain if you skip quorum/fencing, and doesn't remove the detection-timeout trade. It also doesn't guarantee the new leader is healthy for the *whole* fleet — re-point must still propagate.

## 24. Decision Connections

Decisions that go together with Failover:

- [[database-replication|Database Replication]] — failover operates on replicas (promotion).
- [[availability|Availability]] — failover is the mechanism that delivers availability.
- [[rpo-rto|RPO and RTO]] — the numbers failover design commits to.
- [[standby-models|Standby Models]] — hot/warm/cold standby choices feeding promotion.
- [[disaster-recovery|Disaster Recovery]] — failover is the short-horizon piece of DR.
- [[database-connection-pooling|Database Connection Pooling]] — the pool re-points to the new primary.
- [[cap-theorem|CAP Theorem]] — the partition behavior that quorum handles.

Decision tree:

```
Primary node dies
    |
    +-- Is a standby ready and freshest?
    |      → promote it ([[database-replication|Database Replication]])
    |
    +-- Could the old node still write?
    |      → fence it (prevent split-brain)
    |
    +-- How fast must service resume?
    |      → VIP/discovery re-point ([[rpo-rto|RPO and RTO]])
    |
    +-- Too many false positives flips?
    |      → tune detection timeout
    |
    +-- Want confidence it works?
    |      → quarterly drills ([[disaster-recovery|Disaster Recovery]])
```