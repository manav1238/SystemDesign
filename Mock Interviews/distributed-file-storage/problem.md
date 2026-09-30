---
title: Distributed File Storage System Design - Problem Statement
status: active
tags: [hld, mock, distributed-file-storage]
---

# Distributed File Storage System Design

## Problem Statement

> Design the storage backend of a consumer file-sync product, in the style of Google Drive, Dropbox, or iCloud.
>
> A user signs in on any device, sees a tree of folders and files, and can upload, download, share, and delete. The client is frequently offline, on a hotel wifi, or on a metered mobile connection, so the system must tolerate partial uploads, interrupted sessions, and two devices editing the same file at once.
>
> In scope:
> - Uploading a file of any size, from a few kilobytes to tens of gigabytes, reliably over an unreliable network
> - Resuming an interrupted upload from where it stopped rather than starting over
> - Downloading a file, including partial/ranged reads of a single chunk without pulling the whole file
> - A folder tree view, folder rename and move, and search by file name
> - Shared folders where many users see the same tree
> - Version history: a user can see and restore an earlier version of a file
> - Handling the case where the same file is modified on two devices while both were offline
> - Deduplicating identical data, both across a single user's files and across the fleet
> - Reclaiming space when a user deletes a file, and eventually expiring it from storage
>
> Out of scope:
> - Real-time co-editing of a document body (OT/CRDT merge of text)
> - Rich media processing: thumbnail generation, video transcoding, document preview
> - Permissions and sharing link security at production depth
> - Billing and plan tiers
>
> Design for 500 million registered users and 100 million daily active users, with an average of 2 GB of stored data per active user. Ingest averages 200 GB per day and egress averages 1.5 PB per day, because sync products re-download the same bytes constantly. The durability target is 11 nines: no user-visible data loss in any realistic failure.

---

## Clarifying Questions You Should Ask

Strong candidates ask these before drawing anything. Pick the ones you genuinely need answered.

**On the product shape**
1. Is this a sync product where the local filesystem is the source of truth, or a web app where the server is? This changes where conflict resolution lives.
2. What exactly does "restore an earlier version" mean — a full copy, a delta, or a rewind of the edit history? And how far back does history go?
3. When two devices edit the same file offline, what is acceptable: server-side auto-merge, "keep both", or fail and force a human decision? Is the file binary or text? The answer changes everything.
4. Can files be larger than 5 GB, and does a single file have a hard size cap?
5. Do shared folders need real-time propagation of other people's changes, or is eventual visibility acceptable?

**On scale and data**
6. What is the distribution of file sizes? I strongly suspect the mean is meaningless here and I want the 99th percentile.
7. What is the read-to-write ratio, and how much of the read traffic is re-downloading the same file the user just uploaded (device sync) versus a genuine human download?
8. What is the storage growth rate per quarter, and what is the retention requirement for deleted data before permanent erasure?
9. Is the customer base spread across regions, or is this single-region for now?

**On consistency and correctness**
10. Must two devices that are both online see the same tree at the same moment? Is eventual consistency of a few seconds acceptable for the file list?
11. Is a partially uploaded file visible to the user, or is it invisible until fully committed? This decides whether you need a staging concept.
12. What is the acceptable behavior if a chunk is corrupt on read — return an error, or transparently repair from another replica and only fail if the quorum is lost?
13. Is dedup a hard requirement, or a cost optimization we would trade away for simpler storage?

**On infrastructure and cost**
14. Am I allowed to use an existing object store as a primitive, or must I design the chunk storage layer myself from disks up? This is the single biggest fork in the interview.
15. Can I assume a relational database for metadata, and is the constraint on it write throughput, storage size, or both?
16. What is the target latency for a metadata read (folder listing) versus a chunk read?
17. What is the per-user-per-month cost ceiling? It sets how aggressively I dedup, compress, and tier to cold storage.
18. Is there an explicit requirement to survive the loss of an entire availability zone, or even an entire region?

---

## What You Are Evaluated On

### Phase 1: Requirements
Separate functional from non-functional. Pin down the semantics that actually drive the architecture: resumable upload, commit visibility, versioning identity, conflict behavior, and the durability target. State assumptions explicitly rather than silently absorbing them.

### Phase 2: Estimate
Do the back-of-envelope math out loud with real numbers: users to bytes, chunk count, write and read QPS including peak factors, metadata row volume and size, cache sizing. Show the arithmetic, not just the conclusion. The file-size distribution question is the one that separates strong answers from average ones.

### Phase 3: High-level design
Draw the architecture. Identify the client chunker, the upload path, the metadata service, the chunk index, the chunk storage nodes, the download/CDN path, and the background garbage collector. Name the shard key for the metadata store and say why it is the user and not the file.

### Phase 4: Deep dive
Pick two or three areas and go genuinely deep. The expected core is the chunking and resumable-upload protocol with an explicit commit step, the metadata schema and its sharding, and the consistency split between strong metadata and immutable eventual-consistent chunks. Be ready to talk through dedup with refcounting, hot chunk caching, and the mark-and-sweep garbage collector.

### Phase 5: Trade-offs and follow-ups
Defend the choices against alternatives. "Why not just use S3", "why not HDFS-style blocks on a single big cluster", "why not store the whole file as one object", "what happens when the metadata primary dies", "how do you handle one hot shared folder", "how do you survive replica lag and a cache stampede on a viral file". Know exactly which trade-off you accepted and what it cost.
