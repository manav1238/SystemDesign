---
title: Reliability Decisions
status: active
tags:
  - hld
  - revision
  - reliability
---

# Reliability Decisions

How to harden a system and know whether the hardening is working.

## 1. A dependency is failing — what do I do?

- Slow, flaky, might recover → bounded retries + backoff + jitter → [[retry-and-timeout|Retry and Timeout]]
- Sick for a while; retries will pile up → stop trying, fail fast → [[circuit-breaker|Circuit Breaker]]
- Too many requests overall → pop the limiter → [[rate-limiter|Rate Limiter]]

## 2. The whole region is down — what's my target?

- Decide how much data you can lose and how fast you must recover first → [[rpo-rto|RPO and RTO]]
- Choose the operational model → [[standby-models|Standby Models]] (active-passive vs active-active)
- Then build the actual recovery plan → [[disaster-recovery|Disaster Recovery]]

## 3. Is the system actually healthy?

- Universal gauge → [[golden-signals|Golden Signals]] (latency, traffic, errors, saturation)
- Which hop is slow? → [[distributed-tracing|Distributed Tracing]]
- Logs + metrics + traces together → [[observability|Observability]]
- Quantify an SLO and make the product owner sign it → [[sli-slo-sla|SLI / SLO / SLA]]

## 4. Security basics that must be in every design

- Identity: prove identity then authorize action → [[authentication-vs-authorization|Authentication vs Authorization]]
- Modern delegated auth → [[oauth-oidc-jwt|OAuth 2.0 / OIDC / JWT]]
- Protect data at rest and in transit → [[encryption-and-keys|Encryption and Keys]]
- Trust boundaries: validate input → [[web-vulnerabilities|Web Vulnerabilities]]

## Decision tree

```
Dependency failing?
  transient → [[retry-and-timeout|Retry and Timeout]]
  sustained→ [[circuit-breaker|Circuit Breaker]]
  torrent  → [[rate-limiter|Rate Limiter]]

Region down?
  set budget → [[rpo-rto|RPO and RTO]] → [[standby-models|Standby Models]] → [[disaster-recovery|Disaster Recovery]]

Healthy?
  watch → [[golden-signals|Golden Signals]] + [[observability|Observability]]
  find slow hop → [[distributed-tracing|Distributed Tracing]]
  contract → [[sli-slo-sla|SLI / SLO / SLA]]
```