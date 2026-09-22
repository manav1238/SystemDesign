---
title: Encryption and Keys
category: Security
priority: must-know
status: learning
difficulty: hard
interview_ready: false
tags:
  - hld
  - security
  - cryptography
  - key-management
---

# Encryption and Keys (In Transit, At Rest, KMS, Secrets)

## 1. One-Line Definition
Encryption protects data by making it unreadable without a key — **in transit** (TLS between parties), **at rest** (disk/database/object storage), and with **key management** (KMS) and **secrets management** (vaults) organizing the keys and credentials that make it all work.

## 2. Why Do We Need It?
Data crosses untrusted networks, sits on machines that get decommissioned, and lives in backups that leak. Encryption is the difference between "an attacker read everything" and "an attacker has ciphertext." Since keys *are* the security (encryption without good key management is theater), key handling deserves as much design as the encryption itself. And the most common real-world leak isn't crypto-breaking — it's **hardcoded secrets** in repos and configs.

## 3. Simple Intuition
- *In transit:* a sealed armored truck between branches — anyone can see the truck, nobody can open it.
- *At rest:* documents in a locked safe in the office — even if someone steals the safe, they need the combination (key).
- *Keys:* the combination must not be taped to the safe. KMS is the professional locksmith that holds combinations in a tamper-proof vault and lends them only to verified identities, logging every use. Secrets management is the process of never taping credentials to anything.

## 4. What Happens Without It?
- *No TLS:* credentials and PII flow in plaintext; any network hop (coffee-shop Wi-Fi, compromised router, internal sniffer) reads everything. Session hijack, credential theft, MITM.
- *No at-rest encryption:* a stolen disk, leaked backup, or exposed object-storage bucket = full data breach; compliance (PCI-DSS, HIPAA, GDPR) failures.
- *Poor key management:* keys in code repos, shared in chat, never rotated → one leak compromises everything, forever. Notion/GitHub/Codecov-style breaches trace back to credential exposure.

## 5. Core Idea
- **In transit (TLS):**
  - TLS 1.2/1.3, strong cipher suites, valid certs (Let's Encrypt/ACME or managed), **HSTS** to force HTTPS.
  - *mTLS:* both sides present certs → service-to-service authentication (zero-trust internal traffic). See authentication-vs-authorization.
  - Certificate lifecycle: issuance, rotation (short-lived certs, automation), revocation/OCSP.
- **At rest:**
  - Full-disk/volume encryption (protects against disk theft) vs **application-layer/field-level encryption** (protects against DB compromise, per-tenant keys → crypto-shredding).
  - Envelope encryption: a **data encryption key (DEK)** encrypts the data; a **key encryption key (KEK)** in KMS encrypts the DEK. Enables rotation + per-object keys without re-encrypting everything.
  - Searchable/compute needs: keep indices and encrypted fields separate, or use deterministic encryption cautiously (leaks patterns).
- **Key management (KMS):**
  - Central service storing/using master keys; crypto operations via HSM/KMS API (keys never leave), audit logs, rotation schedules.
  - Envelope keys per tenant/object; **crypto-shredding** = delete the tenant's key to make their data unrecoverable (GDPR right-to-erasure for immutable stores).
- **Secrets management:** credentials (DB passwords, API keys, tokens) in a vault (Vault, AWS/GCP Secrets Manager, K8s secrets + external operators), injected at runtime, never committed. Rotation, least-privilege access, audit, short-lived/dynamic credentials (e.g., DB creds minted per pod).
- **Hashing ≠ encryption:** passwords are *hashed* (one-way, salted, slow — bcrypt/argon2), never encrypted. Encryption is reversible with a key; hashing is not.

## 6. Important Terminology

| Term | Simple Meaning |
|------|----------------|
| Encryption | Reversible scrambling with a key |
| Hashing | One-way digest (not reversible) |
| TLS / HTTPS | Encrypted transport |
| mTLS | Both parties authenticate with certs |
| Encryption at rest | Data encrypted on disk |
| Envelope encryption | DEK encrypts data, KMS KEK encrypts DEK |
| KMS | Key management service (HSM-backed) |
| DEK / KEK | Data key / key-encryption key |
| Rotation | Replacing keys periodically |
| Crypto-shredding | Delete key → data unrecoverable |
| Secrets vault | Managed credential storage |

## 7. Basic Architecture

```mermaid
flowchart LR
    C[Client] -->|TLS| LB[Edge]
    LB -->|mTLS| S1[Service A]
    S1 -->|mTLS| S2[Service B]
    S1 --> DB[(DB - at-rest encryption)]
    S1 --> K[KMS: KEK/HSM]
    K -->|wrap/unwrap DEK| S1
    V[Secrets vault] -->|runtime creds| S1
    S1 --> OBJ[(Object store) - field encrypted]
```

## 8. Request or Data Flow
1. Client connects via TLS (cert validated, HSTS enforced); session keys established.
2. Service fetches secrets/credentials from the vault at startup (short-lived, rotated).
3. To write data: get DEK (wrapped by KMS), encrypt payload + write ciphertext + wrapped DEK; store.
4. To read: unwrap DEK via KMS (audited), decrypt.
5. Key rotation: rotate KEK/DEK without re-encrypting all data (rewrap DEKs) or progressively re-encrypt.

## 9. Practical Example
**Healthcare SaaS (HIPAA, assumptions):**
- TLS 1.3 everywhere; mTLS between services; HSTS at edge.
- DB at-rest encryption + per-tenant DEKs; KMS with KEK per region; audit logs of unwrap calls.
- Secrets in Vault with dynamic DB credentials rotated every 24h; no static passwords in config.
- Right-to-erasure: delete the tenant DEK → crypto-shred (records remain but are unreadable). Compliance and operations satisfied without hunting every row.
- A stolen backup disk yields ciphertext; a leaked app config yields nothing (no secrets in it).

## 10. Scaling
- **TLS at scale:** terminate at edge/LB, but keep encryption between internal hops for zero-trust (mTLS) — CPU cost offset by modern hardware/AES-NI.
- **KMS at scale:** a KMS call per encrypt/decrypt is a bottleneck if per-record — use envelope encryption (few KMS calls) and cache unwrapped DEKs in memory with short TTL; watch KMS quotas.
- **Key granularity:** per-tenant keys increase key count (manageability) but improve blast-radius and enable crypto-shredding. Balance granularity against operations.
- **Rotation:** automate; large datasets re-encrypt lazily (background) with dual-key read support.

## 11. Reliability and Failure Scenarios

| Failure | Happens | Detection | Recovery | Trade-off |
|---------|---------|-----------|----------|-----------|
| KMS unavailable | Can't unwrap DEKs | Health/error spike | Cache DEKs, HA KMS, retry | bootstrapping risk |
| Key lost | Data unrecoverable | — | Key backup/escrow, versioning | security vs recoverability |
| Cert expired | TLS failures | Expiry monitoring | Auto-renewal (ACME) | automation needed |
| Secret leaked in repo | Full compromise | Secret scanning | Rotate immediately | incident response |
| Rotation mishandled | Reads fail mid-rotation | Error metrics | Multi-key read, staged roll | complexity |
| Weak cipher/downgrade | MITM | Security scan | Enforce TLS 1.2+/PFS | legacy clients |

## 12. Consistency and Correctness
Encryption introduces a coupling between data and keys: lose the key, lose the data; rotate badly, break reads. Use versioned keys with metadata (`key_id` per ciphertext), keep old keys for decrypt during migration, and treat key loss as equivalent to data loss in your RPO/DR plan (keys must be replicated to DR — see disaster-recovery). Encryption at rest doesn't protect from a compromised application with the key — that's where field-level + least privilege helps.

## 13. Performance
- AES-NI makes bulk encryption cheap; TLS handshakes are the cost (session resumption, keep-alive reduce it).
- Envelope encryption minimizes KMS calls; per-record KMS encryption kills throughput.
- Full-disk encryption adds slight I/O overhead; app-level encryption adds serialization + CPU — measure per path.
- Hashes for passwords are intentionally slow — tune work factor to UX.

## 14. Security
- Cipher choice: AES-256-GCM / ChaCha20-Poly1305 (AEAD — authentication included); avoid ECB and custom crypto.
- Never roll your own crypto; use vetted libraries and protocols.
- Keys in HSM/KMS, never in code/env plaintext; rotate; least-privilege key policies; log all crypto operations.
- TLS: enable forward secrecy (ECDHE), disable old protocols, pin where appropriate (mobile).
- Secrets: dynamic + short-lived, scanned for leaks, rotated on suspicion.

## 15. Trade-Offs

| Choice | Advantages | Disadvantages | When to Use |
|--------|------------|---------------|-------------|
| TLS only | Protects transit | Nothing at rest | All systems (baseline) |
| At-rest disk encryption | Cheap, disk theft safe | DB compromise sees plaintext | All storage |
| Field-level encryption | Granular, tenant keys | Query/index complexity | Sensitive fields (PII) |
| Per-tenant keys | Blast radius, crypto-shred | Key sprawl, ops | Multi-tenant/regulated |
| External vault | Central, audited, rotated | Dependency | All secrets |
| Env vars for secrets | Simple | Leak-prone, no rotation | Dev only |

## 16. Common Mistakes
- Secrets committed to git/env files; "we'll rotate later" after a leak (rotate immediately).
- Encrypting passwords instead of hashing (reversible = breach).
- At-rest encryption assumed to protect from app-level compromise or over-privileged DB users.
- Per-record KMS calls (throughput collapse) instead of envelope encryption.
- No key rotation/versioning → can't rotate without downtime; keys lost = data lost.
- Crypto-shredding not designed for immutable/backup stores, so GDPR erasure claims fail.

## 17. HLD vs LLD Boundary
HLD: transit/at-rest policy, KMS/vault architecture, envelope + key granularity, rotation/crypto-shred strategy, compliance mapping. LLD: TLS config/ciphers, mTLS cert plumbing, DEK wrapping code, secret injection, rotation jobs, cipher library usage.

## 18. Interview Questions

### Beginner
- What's the difference between encryption and hashing?
- Why is hardcoding secrets so dangerous, even in a private repo?

### Intermediate
- Explain envelope encryption and why it's used at scale.
- How do you support GDPR erasure on immutable object storage? (crypto-shredding)

### Advanced
- Design key management for a multi-tenant, multi-region fintech: key hierarchy, KMS topology, rotation, DR for keys, per-tenant blast radius, and the failure modes of losing KMS.
- A backup disk is stolen and one app credential leaked simultaneously. Give the full incident response for both, including rotation ordering and blast containment.

## 19. Interview Memory Summary

> [!abstract]- Interview Memory Summary

### Remember

- Encrypt in transit (TLS/mTLS) and at rest — always.
- Envelope encryption: DEK encrypts data; KMS/KEK wraps the DEK.
- KMS/HSM holds master keys, audits use; keys never leave.
- Secrets live in a vault — dynamic, rotated, never in code/repos.
- Passwords are hashed (slow + salted), never encrypted.
- Version keys so you can rotate + crypto-shred without re-encrypting everything.
- Lose a key = lose the data; encrypt at rest ≠ safe from your own application.

### 30-Second Explanation

TLS everywhere and mTLS internally; encrypt storage with per-tenant DEKs wrapped by KMS (HSM-backed, rotated, audited); keep all secrets in a vault with short-lived credentials; version keys so you can rotate and crypto-shred; treat keys as the real asset.

### Interview Traps

- Equating "encrypted at rest" with "safe from our own application" — field-level encryption, least privilege, and key isolation limit a compromised service.
- Encrypting passwords instead of hashing them (reversible = breach).
- Per-record KMS calls (throughput collapse) instead of envelope encryption.
- No key rotation/versioning; keys lost = data lost.

### Key Trade-Off

Envelope encryption + KMS buys rotation, per-tenant blast-radius isolation, and crypto-shredding — at the cost of key-sprawl, a KMS dependency where availability and key loss are now existential, and having to trade key granularity against operational manageability.

## 20. Related Concepts

### Prerequisites

- [[http-and-https|HTTP and HTTPS]]

### Commonly Used Together

- [[authentication-vs-authorization|Authentication vs Authorization]]
- [[oauth-oidc-jwt|OAuth 2.0 / OIDC / JWT]]
- [[web-vulnerabilities|Web Vulnerabilities]]

### Advanced Concepts

- [[disaster-recovery|Disaster Recovery]]
- [[rpo-rto|RPO and RTO]]

Related planned topics (not authored yet): Password Hashing.

## 21. References
NIST SP 800-57 (key management), OWASP Cryptographic Storage Cheat Sheet, AWS KMS / GCP KMS envelope-encryption docs. Verify current guidance before interviews.

## 22. Active Recall

Test yourself before revealing each answer.

> [!question]- What's the difference between encryption and hashing?
> Encryption is reversible scrambling with a key — when you have the key you get the data back. Hashing is a one-way digest that is not reversible. So passwords are *hashed* (one-way, salted, slow — bcrypt/argon2) and never encrypted, while stored data is *encrypted* so it can be decrypted later.

> [!question]- Explain envelope encryption and why it's needed at scale.
> A data encryption key (DEK) encrypts the actual data; a key encryption key (KEK) stored in KMS encrypts/wraps the DEK. The ciphertext is stored with its wrapped DEK. This enables per-object/tenant keys and rotation without re-encrypting data: rotate the KEK, rewrap DEKs; decrypt by unwrapping DEK via KMS. It avoids a KMS call per record.

> [!question]- How do you support GDPR right-to-erasure on immutable object storage?
> Crypto-shredding: store the tenant's rows with a per-tenant DEK wrapped by a tenant KEK; to erase, simply delete that tenant's key. The ciphertext remains but is unrecoverable — no need to hunt and delete every row in an immutable store. This requires per-tenant key granularity and access to key deletion.

> [!question]- Why per-tenant keys in a multi-tenant system, and what do you pay for them?
> Per-tenant keys shrink the blast radius (a compromised tenant key only unlocks that tenant's data), enable crypto-shredding per tenant, and allow independent rotation. The cost is key sprawl — more keys to manage, monitor, and back up — so you balance granularity against operational manageability (e.g., per-tenant DEKs under shared regional KEKs).

> [!question]- What are the failure modes of losing the KMS, and how do you recover?
> KMS unavailable → can't unwrap DEKs → can't read encrypted data; recovery via caching unwrapped DEKs with short TTL, HA KMS/multi-region KMS, and retry. Key *lost* → data unrecoverable; mitigation is key backup/escrow and versioning, and replicating keys to DR — treat key loss as equivalent to data loss in your RPO plan.

> [!question]- Why is "encrypted at rest" not equivalent to "safe from our app"?
> At-rest encryption protects against disk theft, leaked backups, and exposed storage buckets — but a compromised application holding the key (or running with the service account) can decrypt whatever it can read. Field-level encryption, least privilege, and key isolation are what actually limit a compromised service's reach.

> [!question]- A backup disk is stolen and one app credential leaked simultaneously. Give the immediate containment steps.
> 1) Block/rotate the leaked credential immediately (never "rotate later"). 2) Confirm at-rest encryption means the disk yields ciphertext, not plaintext. 3) Check key access logs/audit for unwrap use. 4) Rotate DEKs/KEKs as needed for that scope. 5) Review where the secret was committed, scan repos, move to vault with dynamic short-lived creds. Order rotation by what the leaked credential can reach.

> [!question]- Why do you version keys and why is key loss equivalent to data loss in DR?
> Versioned keys carry a `key_id` per ciphertext so you can rotate KEK/DEK, keep old versions for decrypt during migration, and re-encrypt lazily with dual-key read support. Keys are single-source-of-truth assets: lose the key, lose the data — so key backup/escrow and replication to DR belong in the disaster recovery/RPO plan before interviews.

> [!question]- Interview scenario: design key management for a multi-tenant, multi-region fintech.
> Hierarchy: per-tenant DEKs wrapped by regional KEKs living in KMS/HSM. Topology: KMS per region/HA, cached JWKS-free DEK unwrap with short TTL, audit logs per operation. Rotation: automated, staged roll with versioning + dual-key read. DR: keys replicated/escrowed across regions. Blast radius: per-tenant keys crypto-shred; incident drills include KMS loss and key compromise with rotation ordering.

## 23. When Should I Use This?

### Use it when

- Any data crosses networks (TLS baseline) or sits on disk / in DBs / in backups — always.
- You hold PII or regulated data (HIPAA, PCI, GDPR) needing at-rest encryption + audit.
- Multi-tenant systems need per-tenant isolation, blast-radius control, and erasure (right-to-erasure).
- Secrets (DB passwords, API keys, tokens) need central, rotated, least-privilege management.
- You need rotation without downtime or re-encryption of everything (envelope encryption, versioned keys).

### Avoid it when

- Data is genuinely public/non-sensitive and adding encryption buys nothing but latency.
- You can't commit to key lifecycle ops (rotation, escrow, DR replication) — weak key management makes encryption theater.
- Field-level encryption would break required search/analytics and the cost isn't justified (keep indices vs encrypted fields separated first).
- The problem is actually insider access at the app layer — encryption alone can't fix a service that decrypts on behalf of the attacker.

### What problem does it solve?

Plaintext data leaks across networks, disks, backups, and buckets, and hardcoded secrets turn a minor incident into a full compromise. The bottleneck: encryption is only as strong as its key management. The solution: encrypt in transit (TLS/mTLS) and at rest, use envelope encryption + KMS/HSM so keys never leave and rotate cleanly, keep secrets in a vault, and crypto-shred per tenant for erasure with keys treated as the real asset.

### What problem does it NOT solve?

A compromised application that holds/uses the key, memory/process-level attacks, and at-rest encryption doesn't protect from over-privileged DB users or insider access. It also doesn't engineer your failover or DR translation of key loss into data loss by itself — those are separate decisions.

## 24. Decision Connections

Decisions that go together with Encryption and Keys:

- [[http-and-https|HTTP and HTTPS]] — TLS is the transport encryption baseline for everything over a network.
- [[authentication-vs-authorization|Authentication vs Authorization]] — mTLS provides service identity; least-privilege key access is an authZ problem.
- [[oauth-oidc-jwt|OAuth 2.0 / OIDC / JWT]] — JWT signing/verification and JWKS are key management; secure key storage underpins trusted identity.
- [[web-vulnerabilities|Web Vulnerabilities]] — SSRF/secret exposure is how keys get leaked; hashing (not encryption) for credentials.
- [[disaster-recovery|Disaster Recovery]] and [[rpo-rto|RPO and RTO]] — key loss = data loss; keys must be escrowed/replicated as part of DR.
- [[database-fundamentals|Database Fundamentals]] / [[database-indexing|Database Indexing]] — at-rest encryption interacts with searchability and indexes; encrypted fields need separate handling.

Decision tree:

```
Need to protect data?
    |
    +-- Data crosses a network?
    |      → [[http-and-https|HTTP and HTTPS]] → TLS (HSTS)
    |      +-- Service-to-service / zero trust? → mTLS
    |
    +-- Data at rest (disk/DB/object storage)?
    |      → at-rest encryption (baseline)
    |      +-- Sensitive fields / tenant isolation? → field-level, per-tenant keys
    |      +-- Many records, needs efficiency?      → envelope encryption (DEK + KEK)
    |
    +-- Who holds the keys/secrets?
    |      → KMS/HSM (keys never leave, audited) + secrets vault
    |      → [[authentication-vs-authorization|Authentication vs Authorization]] (least privilege on keys)
    |
    +-- Rotation / erasure requirements?
    |      +-- Rotate without re-encrypt?  → versioned keys, rewrap DEKs, dual-key read
    |      +-- Right-to-erasure?            → crypto-shredding (delete tenant key)
    |
    +-- Disaster / loss scenarios?
           → [[disaster-recovery|Disaster Recovery]] + [[rpo-rto|RPO and RTO]] — escrow & replicate keys
```