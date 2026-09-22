---
title: CI/CD
category: Deployment and Infrastructure
priority: important
status: learning
difficulty: easy
interview_ready: false
tags:
  - hld
  - deployment
  - automation
---

# CI/CD

## 1. One-Line Definition
CI/CD (continuous integration and continuous delivery/deployment) automates the pipeline from code commit to production — building, testing, and packaging every change (CI), then delivering it to staging and deploying to production automatically upon approval or automatically upon passing gates (CD).

## 2. Why Do We Need It?
Manual release rituals are slow, error-prone, and un-reviewable: a hand-run build on a laptop, a manual "deploy bundle" copied to servers, and a "did it go live?" guess. CI/CD compresses the gap between "code written" and "code running" so feedback is fast, every change is testable in minutes, and deployments become boring, repeatable, and reversible. It is the vehicle that makes the deployment toolbox — [[deployment-strategies|Deployment Strategies]], [[feature-flags|Feature Flags]], [[autoscaling|Autoscaling]] — actually executable at speed.

## 3. Simple Intuition
A factory assembly line with inspection stations. Every part (commit) moves down the line automatically: first station checks the part's spec (build + unit tests), the next stress-tests it (integration tests), the next packs it (artifact), the next ships it to the showroom (staging). Only parts that pass every station reach the customer floor (prod), and the line is triggered by the part itself — not by a person walking each piece by hand.

## 4. What Happens Without It?
Merge trains that break the build for days; "works on my machine" stories; a hand-assembled release bundle that no one can reproduce; deploys that happen at 11pm by one person holding their breath. Every change is a gamble, reverting is a nightmare, and the team ships rarely because each release is an event. Fast feedback (see [[functional-vs-non-functional-requirements|Functional vs Non-Functional Requirements]] on dev-speed) is gone, and the cost of a mistake is measured in outages instead of a pipeline rerun.

## 5. Core Idea
- **CI — integration happens continuously:** every commit to the main branch triggers a build + test in a controlled environment; breaking the build is a team emergency. Trunk-based (short-lived feature branches) makes this achievable.
- **Artifacts are the product of CI:** the pipeline produces an immutable, versioned, signed artifact (container image, binary) — deploy and run exactly what was tested, never a rebuild at deploy time.
- **CD — delivery/deployment:** delivery = artifact promoted to staging and made available to prod gates; deployment = the same artifact promoted into production automatically once quality gates pass (green tests, SLO checks, manual approval).
- **Pipeline-as-code:** the build spec itself lives in git (gitHub Actions, GitLab CI, Jenkinsfile), so the pipeline is reviewed, versioned, and branchable like code.
- **Quality gates, not just build:** lint, unit, integration, contract, security scan, SBOM/artifact signing, image digest pinning — each a checkpoint that blocks promotion. Later gates exercise runtime checks: canary health, SLO adherence, rollback triggers.

## 6. Important Terminology

| Term | Simple Meaning |
|------|----------------|
| CI | Continuous integration: auto build + test each commit |
| CD | Continuous delivery/deployment: promote artifact through envs |
| Artifact | Immutable, versioned build output (image, binary) |
| Pipeline | The ordered set of stages a change flows through |
| Quality gate | A checkpoint that blocks promotion on failure |
| Trunk-based dev | Committing to main frequently in small changes |
| Promotion | Moving an artifact from one env to the next |
| Rollback | Reverting to the previous known-good artifact |
| Image digest | Content-addressed, immutable image identifier |
| Service-level gate | SLO/latency checks that gate a deploy |

## 7. Basic Architecture

```mermaid
flowchart LR
    Commit[Commit to main] --> CI[CI: build + test]
    CI --> Artifact[Artifact registry]
    Artifact --> Env1[Staging]
    Env1 -->|quality gate| Env2[Canary in prod]
    Env2 -->|"SLO gate"| Prod["Production 100 percent"]
    Prod -->|rollback trigger| Rolling[Previous artifact]
```

A change races through fast gates, then is promoted environment-by-environment with heavier gates (including live SLO checks) at each step; the artifact never changes, so what was tested is exactly what runs.

## 8. Request or Data Flow
1. A branch merges to main: `git push` fires the pipeline definition from the repo.
2. CI checks out the code, builds, runs unit + lint; failure stops the pipeline and flags the commit.
3. Success → an image is built, signed, pushed to the registry with a content digest, and tagged with the commit SHA.
4. The artifact is deployed to staging; integration + contract tests run against the real stack.
5. On green gates, the artifact is promoted: canary 5% via traffic weighting; SLO gate observes for N minutes.
6. Gate passes → 100% production rollout; later, a git tag or rollback pipeline can flip back to the previous digest.

## 9. Practical Example
**A 200-engineer platform squad sees 300 merges/day through one shared pipeline.** Average CI: 8 minutes (build + parallel test shards), risk gating via `grep`-fast unit layers and slow integration later. Every artifact is a signed image pinned by digest. Deploys run on gate: staging first, then a 5% canary held 10 minutes against SLO error-budget alarms, then 100%. A flaky deploy is auto-rolled-back by the pipeline to the last passing digest in under 2 minutes. The monthly "release day" culture is gone — releases happen continuously and quietly.

## 10. Scaling
- **Build/test parallelism is the growth lever:** shard tests, cache layers for images, and separate fast (unit) vs slow (integration) stages so long suites don't slow every commit.
- **Pipeline fan-out at scale:** 100s of services each with its own pipeline — converge on shared templates/modules so behavior (signing, gating) is uniform, or use a monorepo's affected-path detection to build only changed services.
- **What breaks:** flaky tests that rotate the flakiness gambit, long queued pipelines at peak merge parties, and heavy promotion traffic overloading the artifact registry — cache popular layers, pre-push, and quota the concurrent pipeline load.

## 11. Reliability and Failure Scenarios

| Failure | What Happens | Detection | Recovery | Trade-off |
|---------|--------------|-----------|----------|-----------|
| Flaky test fails CI | Blocked merges, distrust of the gate | Flake metrics | Quarantine flaky tests fast | gate trust fades |
| Artifact registry unavailable | Nothing can deploy | Registry health | Replica + read-through cache | dependency |
| Canary SLO fails | Keep pipeline stuck, auto-rollback | SLO alerts on canary slice | Auto-rollback previous digest | deploy latency |
| Bad artifact signed + shipped | Prod impact before human notices | Runtime monitors | Kill switch + rollback + Artifact audit | gate coverage |
| CD secret rotation missed | Deployments fail authentication | Pipeline errors | Rotate + regenerate pipelines | key ceremony |

## 12. Consistency and Correctness
- **Build artifacts are immutable, deployments are idempotent:** the same digest deployed twice yields the same result — never rebuild "from branch" at deploy time.
- **The pipeline must be deterministic:** same input commit → same output digest. Pinned tool versions and build caches guarded against impurity keep the CI trustworthy.
- **Gate ordering is a correctness choice:** deploy-time runtime gates (canary SLO) are the only ones that can catch behavior that looks fine in unit tests — never trust compile/lint as the whole story.

## 13. Performance
- Pipeline latency = the cost of every commit, so speed is a product decision: gate order, layer caching, test sharding, and skipping irrelevant stages matter as much as correctness.
- CD adds runtime performance checks: canary stages that watch p99 and error-budget burn decide speed of promotion; a slow gate isn't just delay, it's longer exposure to the risky change.
- Rolling back is the reciprocal cost: rollback latency should be near-instant (traffic flip) or the pipeline must accept outage-window risk on bad deploys.

## 14. Security
- **Supply chain is the security story, not the build:** sign artifacts, pin digests, scan images/vulnerabilities, and never let a rebuild at deploy bypass the signed artifact (a "trust the digest, not the tag" policy).
- The CI/CD runner's credentials can deploy — least privilege, short-lived tokens ([[oauth-oidc-jwt|OAuth 2.0 / OIDC / JWT]]), isolated runners, and audit of pipeline secrets.
- Secrets must not be baked into images or repos ([[encryption-and-keys|Encryption and Keys]]); a leak in CI is a deployment leak.

## 15. Trade-Offs

| Approach | Advantages | Disadvantages |
|----------|------------|---------------|
| CI only, manual deploys | Simple, human checks at prod | Deploy still slow + error-prone |
| CD to staging, manual prod | Safety at the edge, human override | Prod lag, argues about approval process |
| Full CD (auto prod) | Fastest feedback, consistent gates | Needs trustworthy canaries/SLO; blast radius discipline |
| Pipeline-as-code (shared templates) | Uniform behavior, reviewable | Templates constrain team-specific needs |

## 16. Common Mistakes
- "My code works, the build fails" — breaking main repeatedly erodes CI trust and everyone starts merging anyway; gate hygiene (lock broken builds) matters more than any single fix.
- Deploying a rebuild rather than the tested artifact — the whole CI/CD safety argument evaporates if the image changes.
- Putting secrets in pipeline config or images; one leak and the compiler gets root.
- Treating staging as "lesser" with different data/config — gates that don't reflect prod behavior are theater.
- Manual-only canary decisions: human judgment is fine, but a pipeline that waits on a human for the last mile is a bottleneck that will be skipped in a panic.

## 17. HLD vs LLD Boundary
HLD: pipeline stages and gate policy (which tests, which SLOs, promotion rules), artifact strategy (registry, signing, digest pinning), env topology (staging vs canary vs prod), rollback procedure, monorepo vs micro-repos, runner infra. LLD: the actual pipeline YAML stages, test sharding configs, secret references, the specific gate scripts and alert thresholds.

## 18. Interview Questions

### Beginner
- What problem does CI solve that "running tests locally" doesn't?
- Distinguish CI, CD-as-delivery, and CD-as-deployment with examples.

### Intermediate
- Design a pipeline for a deploy at risk: what gates would you place before 100% prod?
- Your latest merge broke the build. Describe the remediation flow and the prevention policy.

### Advanced
- Design CI/CD where the canary itself is the gate (SLO-driven promotion) with automatic rollback — walk the state machine.
- A signed artifact pipeline at 10k engineers: how do you keep scope, parallelism, and gate uniformity sane?

## 19. Interview Memory Summary

> [!abstract]- Interview Memory Summary
> ### Remember
> - CI: every commit builds + tests; a broken main is an emergency.
> - CD: promote immutable artifacts through staging → canary → prod.
> - The tested artifact is what deploys — never a rebuild at deploy.
> - Quality gates: unit → integration → contract → security → runtime/SLO.
> - Runtime gates (canary SLO) are the ones that catch real behavior.
> - Pipeline-as-code: the pipeline lives in git, reviewed like code.
> - Security is supply chain: sign + pin digests + scan; runner creds least-privilege.
> - Speed is a product choice: layer caching, sharding, affected-path build.

### 30-Second Explanation

CI builds and tests every commit fast, locking main green; the pipeline produces a signed, immutable artifact. CD promotes that artifact through staging and a canary, checking static gates first and runtime SLO gates before widening to 100%, with automatic rollback to the last passing digest. The whole pipeline lives as code in git. What CI/CD actually buys is that deploys become boring, everyone's changes are testable in minutes, and the release ritual collapses to a gate decision.

### Interview Traps

- Pretending local builds replace CI — the point is a *shared* green main, not passing tests.
- Allowing deploy-time rebuilds ("warm me a fresh image") — that breaks determinism and supply-chain safety.
- Trusting lint/unit as sufficient gates — only runtime SLO gates catch real behavior.
- Ignoring runner/credential security — CI credentials deploy; treat them accordingly.

### Key Trade-Off

You buy fast, reviewable, reversible deploys at the price of pipeline complexity and the discipline that the artifact you tested is the artifact you ship — plus a supply chain you must secure end to end.

## 20. Related Concepts

### Prerequisites

- [[containers-and-vms|Containers and VMs]]
- [[deployment-strategies|Deployment Strategies]]

### Commonly Used Together

- [[deployment-strategies|Deployment Strategies]]
- [[feature-flags|Feature Flags]]
- [[infrastructure-as-code|Infrastructure as Code]]

### Alternatives

- Manual release rituals (the thing CI/CD replaces)
- [[deployment-strategies|Deployment Strategies]] (the deployment mechanism gates run)

### Advanced Concepts

- [[feature-flags|Feature Flags]]
- [[autoscaling|Autoscaling]]

Related planned topics (not authored yet): trunk-based development, artifact registry, supply-chain security.

## 21. References
GitHub Actions, GitLab CI, and Jenkins documentation for pipeline semantics. Google SRE Book, chapter on releases and rolling back. "Continuous Delivery" (Humble and Farley) is the canonical book. DORA metrics (deployment frequency, lead time, MTTR, change-failure rate) are the canonical measurement of CI/CD health. Verify current runner hardening guidance against your platform.

## 22. Active Recall

Test yourself before revealing each answer.

> [!question]- Why is a signed, immutable artifact the core invariant of CD?
> Because deploy must run exactly what the pipeline tested. If staging tests A but prod deploy pulls a fresh build B, gates are meaningless and failures become unpredictable. Signing + digest pinning make the artifact a fixed identity that can be reproduced, audited, and rolled back to.

> [!question]- A canary passes unit/integration locally but SLO fails at 5%. Which gates were insufficient and what fixes it?
> Static gates (lint, unit, integration) can't catch load-dependent behavior. The fixes: a canary SLO gate — error rate, p99, saturation — observed over a bake window, plus a rollback handoff that flips traffic back before the drain. That runtime gate is the one that catches what tests can't.

> [!question]- Your merge train has 12 open PRs; the build is flaky-red. What is the correct response?
> Treat it as an emergency: the pipeline is the shared feedback surface, a flaky gate erodes trust. Quarantine the flaky test, restore green, and add a policy — no expiring successes, broken builds lock merges. Allowing merges into a red main is exactly how CD trust dies.

> [!question]- Why must CI/CD runner credentials be least-privilege and short-lived?
> Because the pipeline can deploy, rotate secrets, and reach infrastructure — a compromised runner is essentially a cloud admin. Least-privilege roles bound the blast radius, short-lived tokens limit replay, and audit gives you the chain back when something deploys without approval.

> [!question]- Interview scenario: your team's deploys take 4 hours on Friday because "QA must test everything." Redesign the flow.
> 1) Fast CI (unit) for every commit; keep main green. 2) Immutable artifact + build-once. 3) Canary to 5% with SLO gates — QA becomes quality gates, not humans. 4) Feature flags to decouple merge from visibility further. 5) Auto-rollback from a bad promo. Result: deploys move from 4 hours of Friday ritual to minutes of pipeline decisions.

## 23. When Should I Use This?

### Use it when

- Multiple people commit to a shared codebase and broken builds have real cost.
- Deploys are frequent enough that manual rituals are the bottleneck.
- You want every change observable, reviewable, and reversible through a pipeline.
- You run staging/canary envs your app code deploys through anyway.

### Avoid it when

- You're building a throwaway prototype where a one-liner `deploy.sh` is honestly faster.
- The team has no one to operate pipelines — an abandoned pipeline is worse than none.
- Environments are so bespoke that "pipeline" means a stack of undocumented manual steps (that's a cultural problem first).

### What problem does it solve?

It collapses the time and risk between write-code and run-code: shared green, fast feedback on every commit, immutable artifacts, gates at each promotion, and deploys that are frequent, small, boring, and reversible.

### What problem does it NOT solve?

It does not make tests actually reflect prod, does not fix a broken build culture (bad gates get ignored), does not secure your supply chain by itself (signing, scanning, creds all still on you), and does not pick the deployment shape — that's [[deployment-strategies|Deployment Strategies]].

## 24. Decision Connections

Decisions that go together with CI/CD:

- [[deployment-strategies|Deployment Strategies]] — the mechanisms the pipeline drives (rolling, blue-green, canary).
- [[feature-flags|Feature Flags]] — let the pipeline ship dark code and gate visibility separately.
- [[infrastructure-as-code|Infrastructure as Code]] — infra changes flow through the same review/promote pipeline.
- [[containers-and-vms|Containers and VMs]] — images are the immutable artifacts pipelines produce.
- [[kubernetes|Kubernetes]] — the usual deploy target consuming those images.
- [[observability|Observability]] — SLO gates need per-version metrics to decide promotion.
- [[reliability|Reliability]] — rollback cadence and change-failure rate are reliability metrics, not just process.

Decision tree:

```
Turning commits into running software?
    |
    +-- Prototype, hackathon?
    |      → a deploy script, honestly
    |
    +-- Real team, daily merges?
    |      |
    |      +-- Build + test each commit?  → CI (trunk-based, keep main green)
    |      +-- Ship via immutable artifact? → CD with signed digests
    |      +-- Drive choose rollout shape → [[deployment-strategies|Deployment Strategies]]
    |      +-- Gate canary on SLO?        → runtime gates + auto-rollback
    |
    +-- Infra changes too?
           → [[infrastructure-as-code|Infrastructure as Code]] same pipeline
```