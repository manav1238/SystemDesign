---
title: Distributed File Storage - Full Mock Interview Transcript
status: active
tags: [hld, mock, distributed-file-storage, interview]
---

# Distributed File Storage - Full Mock Interview Transcript

A complete 45-minute session, annotated with the structure a strong answer follows. Read it twice: once for flow, once for the numbers.

**Target:** senior backend / senior infrastructure engineer
**Rubric:** the five phases from [[06-hld-interview-checklist|HLD Interview Checklist]]

---

## Phase 1: Requirements Clarification (0:00 - 6:00)

**Interviewer:** Thanks for joining. Today's problem is a distributed file storage and sync system, Google Drive or Dropbox style. Users log in anywhere, see a folder tree, and upload and download files. The client is often offline, on flaky networks. How would you approach this?

**Candidate:** Before I draw anything, I want to pin down what actually drives the architecture, because "file storage" can mean two very different products. Let me ask.

First, is the local filesystem the source of truth with the server as a sync target, or is this a web app where the server is authoritative? And second, is a file visible to the user the moment the first byte lands, or only after the whole file is committed? And third, when two devices edit the same file while both are offline, what is the expected behavior: automatic merge, keep-both with a renamed copy, or a hard failure asking the user to resolve?

**Interviewer:** Server is authoritative. A file becomes visible only when it is fully committed. For conflicts, automatic merge only for text, keep-both with a suffix for binary. Go ahead.

**Candidate:** Good, that already tells me a lot. Commit-only visibility means I need a staging concept, which means I need an explicit finalization step, which means upload is a two-phase operation. Let me keep going.

**Interviewer:** Go ahead.

**Candidate:** Next I need the file size distribution, not the mean. And the read-to-write ratio. And whether dedup is a hard requirement or a cost optimization we would trade away.

**Interviewer:** Dedup is a cost optimization. We would trade small files first. Read to write is maybe 8 to 1 in bytes but far more in request count because device sync re-downloads constantly.

**Candidate:** That distinction matters. Byte-level ratio suggests a CDN problem. Request-count ratio says my real cost is metadata and index lookups, not bytes.

**Interviewer:** What about durability and availability? Do you have a stated target?

**Candidate:** 11 nines. And I would like to know: single region or multi region? Do I need to survive a whole region loss, or is a zone failure the worst case?

**Interviewer:** Zone failure is the design requirement. Region loss gets a backup-restore, not active-active, for now. And one more constraint: build the chunk storage layer yourself, from disks up. No off-the-shelf object store.

**Candidate:** That is the fork in the road, so good that you told me. Then I am building a distributed chunk store with replication or erasure coding, and I own the failure detection, the repair, and the scrubbing.

**Interviewer:** Correct. And the file size cap?

**Candidate:** 50 GB.

**Interviewer:** Good. Start from requirements.

**Candidate:** Let me lay out what I heard, then the non-functional targets I am designing against.

Functional requirements:

- Upload a file from 1 KB to 50 GB, reliably, resumable across client restarts
- List a folder tree, rename, move, delete
- Download, including ranged partial reads of a single region of a large file
- Shared folders with many members
- Version history with restore
- Search by file name
- Deduplicate identical data
- Reclaim space on delete, and eventually erase

Non-functional requirements I am assuming, and will hold myself to:

- Durability 11 nines. No silent data loss.
- Metadata read p99 under 50 ms, cached reads under 5 ms
- First byte of a download under 200 ms for a small file
- Resumable upload with at most one chunk of wasted re-transfer after any network failure
- Metadata strongly consistent; chunk data immutable, so effectively eventually consistent
- Survive loss of any single availability zone with no data loss and degraded but functional service
- Explicit, documented conflict semantics rather than silent last-write-wins

**Interviewer:** That is a clean list. Go ahead and estimate.

---

## Phase 2: Scale Estimation (6:00 - 11:00)

**Candidate:** Let me do this carefully because the file size distribution is the whole ballgame.

Starting inputs: 500 million registered, 100 million DAU, 2 GB stored per active user.

Total logical volume: 100 million times 2 GB is 200 terabytes of live user data. Add non-current versions, call it 30 percent, so roughly 260 TB logical.

Now file count. If the mean file were 4 MB, that is only 50 billion files, which feels low. Let me instead reason from behavior: a typical consumer has photos, PDFs, documents, and a few videos. I will assume 500 files per active user on average.

100 million times 500 is 50 billion files. That gives a mean file size of 260 TB over 50 billion files, which is about 5 KB. That feels too small, so let me adjust: the mean is dominated by a huge number of tiny files, and the volume is dominated by a tiny number of huge ones. So the honest way to say it is a skewed distribution, and I will model it explicitly.

My model:
- 80 percent of files are under 100 KB. These are documents and thumbnails.
- 17 percent are between 100 KB and 10 MB. Photos, archives.
- 2.9 percent are between 10 MB and 1 GB.
- 0.1 percent are over 1 GB, up to 50 GB.

Check the volume: the top 2.9 percent band, 1.45 billion files averaging 60 MB, is about 87 TB. The 0.1 percent band, 50 million files averaging 8 GB, is 400 TB. That overshoots, so let me dial the tail down: 0.1 percent averaging 2 GB gives 100 TB, and the 2.9 percent band averaging 40 MB gives 58 TB. Total from the big two bands is about 158 TB, and the small bands contribute the remaining tens of TB. That lands at roughly 200 to 260 TB. Good enough, and the shape is right: the tail carries the bytes, the head carries the row count.

Now chunking. I chunk at an average of 4 MB, so any file under 4 MB is a single chunk. Only files over 1 MB get chunked meaningfully, which is about 20 percent of files, roughly 10 billion files. Those average 20 MB, so about 5 chunks each, giving 50 billion chunk instances. With dedup and content addressing, call it 40 billion unique chunks.

**Interviewer:** Four megabyte chunks on a 50 GB file?

**Candidate:** That is 12,500 chunks. And that is deliberate, and I will come back to it, because it is the single number I am least sure about and I will show you the trade.

**Interviewer:** Go on.

**Candidate:** Metadata rows: 50 billion file rows at roughly 200 bytes of payload is about 10 TB, times 3 for replication is 30 TB of relational database. That is large but it is a solved problem: sharded MySQL or Postgres behind a directory.

Now write QPS. Ingest is 200 GB per day. That is 2.3 MB per second on average, which sounds absurdly small, but the load is not uniform. Device sync is an evening and morning phenomenon, so I will use a peak factor of 10, giving about 23 MB per second at peak.

Sessions: 10 million uploads per day. 200 GB over 10 million uploads is 20 MB per upload average. At 4 MB chunks, that is 5 chunks per upload, so 50 million chunk writes per day. That is 580 QPS average, and roughly 6,000 QPS at peak. That is a very comfortable number, and it is the number that tells me the design should be optimized for durability and correctness, not for raw chunk throughput.

**Interviewer:** So chunk writes are cheap. What is expensive?

**Candidate:** Reads. 1.5 PB per day of egress is 17 GB per second average, and with a peak factor of 4 that is 70 GB per second, which is 560 Gbps. That absolutely has to come off the origin through a CDN. And request count: 100 million DAU times 30 downloads a day is 3 billion downloads a day, 35,000 QPS average, 175,000 QPS at peak. So the real cost centers are metadata reads for folder listings and egress bytes, not chunk writes.

Cache sizing. The hot chunk index. 40 billion unique chunks at about 80 bytes of index entry is 3.2 TB, which does not fit in Redis cheaply, so I will tier it: a Bloom filter for existence probes, and a sharded in-memory index holding the hottest 5 percent, which is 2 billion chunks, 160 GB across 40 nodes of 4 GB each. And a separate SSD cache on the chunk nodes for the hottest chunk data, about 30 percent of a 10 TB hot set.

**Interviewer:** Where did the 10 TB hot set come from?

**Candidate:** 5 percent of 260 TB. I am assuming the top 5 percent of chunks by access frequency carry about half of all read traffic, which is the standard long-tail shape.

**Interviewer:** Good. Now the API and then the architecture.

---

## Phase 3: API Design and High-Level Architecture (11:00 - 18:00)

**Candidate:** Three APIs, and the third one is the internal one I will not expose.

**Interviewer:** Go.

**Candidate:** The metadata API is a REST or gRPC service with a small surface.

```
POST   /v1/uploads                       create upload session -> uploadId
POST   /v1/uploads/{uploadId}/parts      register a committed chunk -> dedup + placement
POST   /v1/uploads/{uploadId}/complete   assemble -> new file version
GET    /v1/files/{fileId}                metadata + chunk manifest
GET    /v1/files/{fileId}/versions       version list
GET    /v1/nodes/{parentId}/children     folder listing, cursor paginated
GET    /v1/chunks/{chunkHash}/locations  where a chunk lives (internal, auth'd)
POST   /v1/files/{fileId}:delete         soft delete
POST   /v1/search                        file name search
```

The create-session contract:

```json
POST /v1/uploads
{
  "fileName": "quarterly-report.pdf",
  "totalSize": 204800000,
  "parentId": "fld_9a2c",
  "baseVersionId": "ver_7f11",
  "clientId": "dev_android_8842",
  "dedupe": true
}
-> 201
{
  "uploadId": "upl_55b1",
  "chunkSize": 4194304,
  "expectedChunks": 49,
  "partUrls": ["https://s3part...", "..."]
}
```

The server picks the chunk size, so it can change the algorithm per file without shipping a new client. `baseVersionId` is the client's promise, and that is the conflict hook. I will come back to it.

The commit contract, and this is the important one:

```json
POST /v1/uploads/{uploadId}/complete
{
  "parts": [{"index": 0, "hash": "9f2c...", "size": 4194304}, ...]
}
-> 200
{
  "fileId": "file_3c19",
  "versionId": "ver_2b04",
  "status": "committed",
  "dedupedChunks": 31,
  "storageClass": "HOT"
}
```

Now the architecture.

```text
                        +-------------------------------+
                        |         Desktop / Mobile       |
                        |   chunker, hasher, sync engine |
                        +--------+-------------+--------+
                                 |             |
                    control API   |             |  data plane (direct,
                    (small reqs) |             |  signed, resumable)
                                 v             v
              +---------------------------------------------+
              |            Edge / API Gateway              |
              |   auth, rate limit, request id, LB        |
              +---+------------------+------------------+---+
                  |                  |                  |
        +---------v--------+  +------v-------+  +-------v----------+
        | Metadata Service |  |  Chunk Index |  |  Upload Session  |
        | (stateless, many |  |  (in-memory  |  |  Tracker         |
        |  replicas)       |  |   + Bloom)   |  |  (state machine) |
        +---------+---------+  +------+-------+  +------------------+
                  |                   |
        +---------v---------+  +------v-------------------+
        |  Metadata Store   |  |  Chunk Placement / Quorum  |
        |  sharded RDBMS,   |  +---------+-------------------+
        |  user_id shard,   |            |
        |  3 replicas/AZ    |     +------v-----------------------+
        +-------------------+     |  Chunk Storage Cluster      |
                                  |  append-only per node,      |
                                  |  fsync'd WAL, 3x replica   |
                                  |  RF3 hot / 8+3 EC cold      |
                                  +------+---------------------+
                                         |
                                  +------v---------------------+
                                  |  Garbage Collector          |
                                  |  mark-sweep + refcount      |
                                  +----------------------------+

        DOWNLOAD PATH (separate, cache-heavy)

   client --> Edge --> CDN edge cache  ---- hit ---->  client   (95% of bytes)
                   --> Origin shield --> Chunk Cache --> Chunk Nodes --> storage
```

**Interviewer:** Two things jump out. First, why does the client talk to storage nodes directly instead of everything going through your services?

**Candidate:** Three reasons. First, bytes. If all download traffic transits my origin, I pay 70 GB per second of egress from my own network and I am bandwidth-bound on every hop. Direct client-to-node with signed URLs lets the CDN terminate it and lets the client have a long-lived, resumable, parallel transfer. Second, failure isolation. A metadata outage should not take down downloads of files whose locations I already cached. Third, the upload path is a long-lived stream, potentially many minutes, and holding a service thread for that is wasteful.

**Interviewer:** Second thing: you have a separate Chunk Index service. Why is that not just a table in the metadata database?

**Candidate:** Three reasons, and they are all about access pattern. One, the hot working set is huge, 160 GB of hottest entries, and reads are point lookups at very high QPS, which is an in-memory hash workload, not an indexed scan. Two, the write pattern is append and update-by-refcount, and I want that isolated from the transactional metadata writes so a chunk-refcount hot spot cannot stall a folder rename. Three, it has a different availability requirement, which I will explain when we get to failure.

**Interviewer:** Now walk me through the flows.

---

## Phase 4: Request and Data Flow (18:00 - 24:00)

**Candidate:** Upload first, end to end.

**Candidate:** Step one, the client hashes the whole file. For a 20 MB file that is fast. For a 50 GB file I do not hash the whole file before starting, because that would be a full read before any write. I chunk first, then hash each chunk in memory, and I only do a whole-file hash at commit time if the format requires it.

Step two, the client calls `POST /v1/uploads`. The metadata service validates auth, checks the parent folder, checks the `baseVersionId` promise, creates an upload session row in `PENDING` state, and returns the chunk size and part URLs.

Step three, for each chunk the client asks `POST /uploads/{uploadId}/parts` with the chunk hash. This is a dedup probe. If the chunk already exists, the index returns its location and the client skips the upload entirely, which on a re-sync of an unchanged file means zero bytes on the wire. If not, the index allocates the chunk id, picks the placement nodes by [[consistent-hashing|Consistent Hashing]] over the live node set, and returns signed per-node upload URLs with a short TTL.

Step four, the client uploads chunks directly to the chunk nodes, in parallel, four at a time, with per-chunk checksum verification. Each node does a local append, an fsync to its WAL, and an in-memory copy in the page cache. The node acks only after a replication quorum. This is the durability point.

Step five, `POST /uploads/{uploadId}/complete`. The metadata service validates that all expected chunk indices are present, that every chunk hash is resolvable, and that the sizes add up. Then, in one transaction against the shard that owns this `parentId`, it flips the session from `PENDING` to `COMMITTED`, writes a new `file_version` row, appends the chunk manifest, and decrements refcounts on whatever the previous version referenced. Commit is the visibility boundary. Nothing before this is visible to any reader.

Step six, asynchronously, a Kafka topic `chunk-commits` lets the index, the search indexer, the thumbnailer, and the tiering service react. That is [[message-queue|Message Queue]] fanout, and I will use [[kafka-architecture|Kafka Architecture]] with a small number of partitions keyed by file id.

**Interviewer:** What happens if the client dies halfway?

**Candidate:** The session stays `PENDING`. Chunks that landed are already durably stored and are referenced by nothing, so they are garbage. The client restarts, calls `GET /v1/uploads/{uploadId}`, gets back the list of chunk indices already confirmed, and resumes from the first missing one. Wasted re-transfer is at most one chunk, which is 4 MB. A background sweeper marks sessions older than 24 hours `ABORTED` and releases the chunk reservations.

**Interviewer:** Now download.

**Candidate:** `GET /v1/files/{fileId}` returns the metadata and the chunk manifest, plus a CDN URL prefix. The client then fetches chunks from the CDN, in parallel, with `Range` requests if the user seeks inside a large file. Because the manifest is per chunk, a range request that lands inside a single 4 MB chunk is a single partial read of that chunk, and I do not need to re-read the neighbors.

The CDN caches by chunk hash, not by file id. That is the important trick. Two different files that share a chunk share one cache entry, so the dedup win and the cache win are the same win. Cache key is `GET /cdn/chunk/{hash}`, TTL is long because chunks are immutable, and eviction is impossible to get wrong.

**Interviewer:** You mentioned content-defined chunking earlier but your API is fixed-size. Pick one and defend it.

**Candidate:** Let me show the trade honestly.

Fixed 4 MB chunking: simple, no CPU, deterministic, trivially parallel, and the server can change the size later. It fails dedup on insertion and deletion. Insert one paragraph at the top of a 200 MB video and every subsequent chunk shifts, so I re-store 200 MB to change 4 KB.

Content-defined chunking with a rolling hash: the average chunk is 4 MB, boundaries are determined by content, so an insertion only shifts the region until the next resync point, typically changing a handful of chunks. That is the whole reason [[content-addressable-storage|Content-Addressable Storage]] works well. It costs me a rolling-hash pass over the file on both client and any server-side verifier, and it makes the exact chunk set depend on both the algorithm and its parameters.

My decision: fixed-size for the general case, because I control the client and the server can evolve it, and the dominant cost is storage, not CPU. But I would implement CDC for the specific case of large media files, where the dedup win is large and the client can afford the CPU. I would version the chunking algorithm in the manifest so a file recorded under fixed-size is never parsed under CDC.

**Interviewer:** Now the schemas.

**Candidate:** Four tables. The first two are the transactional core, sharded by `tenant_id` plus a hash of `owner_id`, so all of a user's files land on one shard and a folder listing is a single-shard query.

```sql
CREATE TABLE file (
  file_id         VARCHAR(40)  NOT NULL,
  owner_id        BIGINT       NOT NULL,
  parent_id       VARCHAR(40)  NOT NULL,
  name            VARCHAR(512) NOT NULL,
  current_version VARCHAR(40)  NOT NULL,
  size_bytes      BIGINT       NOT NULL,
  mime_type       VARCHAR(128) NOT NULL,
  chunking_algo   VARCHAR(16)  NOT NULL,   -- FIXED | CDC_V1
  chunk_size      INT          NOT NULL,
  state           VARCHAR(16)  NOT NULL,   -- ACTIVE | TRASHED
  trashed_at      TIMESTAMP    NULL,
  created_at      TIMESTAMP    NOT NULL,
  PRIMARY KEY (file_id)
) PARTITION BY HASH(owner_id);

CREATE UNIQUE INDEX ux_file_name_live
  ON file (owner_id, parent_id, name, state);
```

That unique index is what gives me "two devices cannot create the same filename twice" in one database transaction, instead of a distributed lock.

```sql
CREATE TABLE file_version (
  version_id    VARCHAR(40) NOT NULL,
  file_id       VARCHAR(40) NOT NULL,
  version_no    BIGINT      NOT NULL,
  chunk_algo    VARCHAR(16) NOT NULL,
  manifest_uri  VARCHAR(256) NOT NULL,   -- pointer to chunk_manifest store
  manifest_hash VARCHAR(64) NOT NULL,
  size_bytes    BIGINT      NOT NULL,
  created_by    VARCHAR(64) NOT NULL,
  created_at    TIMESTAMP   NOT NULL,
  PRIMARY KEY (version_id)
);

CREATE TABLE upload_session (
  upload_id       VARCHAR(40) NOT NULL,
  owner_id        BIGINT      NOT NULL,
  parent_id       VARCHAR(40) NOT NULL,
  file_name       VARCHAR(512) NOT NULL,
  total_size      BIGINT      NOT NULL,
  chunk_size      INT         NOT NULL,
  expected_chunks INT         NOT NULL,
  base_version_id VARCHAR(40) NULL,       -- the client's promise
  state           VARCHAR(16) NOT NULL,   -- PENDING | COMMITTED | ABORTED
  created_at      TIMESTAMP   NOT NULL,
  expires_at      TIMESTAMP   NOT NULL,
  PRIMARY KEY (upload_id)
);

CREATE TABLE chunk_manifest (
  version_id VARCHAR(40) NOT NULL,
  seq        INT         NOT NULL,
  chunk_hash CHAR(64)    NOT NULL,   -- SHA-256
  size_bytes INT         NOT NULL,
  PRIMARY KEY (version_id, seq)
);
```

The manifest is deliberately its own table and not a blob column, because I want to range-scan it. But it is also cached hot in Redis as a packed array, because reading a 12,500-entry manifest for a 50 GB file on every download would be silly.

**Interviewer:** Why is the manifest not stored in the same row as the file version?

**Candidate:** Row size. A 50 GB file at 4 MB chunks is 12,500 rows. Embedded, that is a multi-megabyte value, and it would be rewritten in full on every version, and it would blow out your buffer pool and your replication bandwidth. Separate table, range-scanned, cached packed. And I put the durable copy in an object-backed manifest store so the chunk index, which is in-memory, can rebuild it entirely from scratch.

**Interviewer:** Chunk index schema.

**Candidate:** The chunk index is the interesting one because it is a key-value store, not rows.

| Key | Value | Notes |
|---|---|---|
| `chunk:{sha256}` | `{size, refcount, nodeIds[3], storageClass, lastAccess}` | in memory, hot 5 percent only |
| `chunk:{sha256}:all` | node list | cold, backed by the storage cluster's own index |
| `refcount:{sha256}` | integer | updated transactionally with the manifest write |
| `file:{fileId}:chunks` | packed hash array | Redis, for download path |

The reference count is the crux of dedup. Every version references chunks, and a chunk is deleted only when its refcount reaches zero. Without it, a shared chunk that one user deletes would vanish for everyone.

**Interviewer:** What about the transactional consistency between the metadata shard and the chunk index refcount? Those are two different systems.

**Candidate:** That is the real problem with dedup, and I am not going to pretend it is free. I have two options. One, the index's refcount is derived, not authoritative, and a reconciler recomputes it by scanning manifests. Slow to converge. Two, the metadata shard is the source of truth and the refcount lives in the same transaction.

I take a third path. The metadata shard owns the refcount, in a `chunk_ref` table on the same shard as the committing upload, updated in the same transaction that commits the version. The in-memory index holds only placement, which is additive and safe to rebuild. So the durable, transactional state is single-shard and I never need a distributed transaction. A commit increments refcounts; a delete decrements; GC only acts when the durable refcount is zero and the row is older than the grace window.

**Interviewer:** Caching.

**Candidate:** Four caches, with different jobs.

One, Redis for folder listings and file metadata. Key `listing:{parentId}`, TTL 60 seconds, invalidated on rename, move, or commit under that parent. The load here is 23,000 QPS average and 115,000 at peak, so this is the cache that actually absorbs the traffic.

Two, the in-memory chunk placement index. Point lookups, no TTL, rebuilt from the durable index on restart.

Three, a Bloom filter per node, or per cluster, for negative existence checks. This is the one that kills the thundering herd on upload. Before uploading, the client probes the cluster. If the Bloom filter says definitely-absent, the client uploads immediately. If it says maybe-present, the client does a synchronous index lookup, and only that subset pays the round trip. With a 1 percent false positive rate, 99 percent of first-time chunk uploads skip the index call entirely. This is the classic [[caching|negative caching]] pattern and it matters more here than the positive caching does.

Four, the CDN edge cache for chunk bytes, keyed by hash.

**Interviewer:** Messaging.

**Candidate:** Topics and why each exists.

| Topic | Producer | Consumers | Why |
|---|---|---|---|
| `chunk-commits` | metadata service | index, search, thumbnails, tiering | post-commit fanout, decouples commit latency |
| `chunk-uploads` | chunk nodes | index | placement updates, refcount debounce |
| `file-deleted` | metadata service | GC, search, CDN purge | triggers space reclamation |
| `tiering-scan` | tiering service | storage cluster | hot to cold transitions |
| `sweeper-ticks` | scheduler | session sweeper | aborts stale uploads |

Everything that is not on the synchronous commit path goes through a queue. That is the whole point: the commit transaction stays short, and index, search, and thumbnails are free to be slow, retry, and lag without affecting the user.

Study separately: [[outbox-pattern|Outbox Pattern]] - I need to make sure a committed version and its `chunk-commits` event cannot diverge, and the outbox is how I get that without a distributed transaction.

**Interviewer:** Sharding and replication.

**Candidate:** Two different data sets, two different strategies.

Metadata store: sharded by `hash(owner_id)`, 64 shards to start, 3 replicas per shard across 3 zones, one primary with synchronous replication to the second and asynchronous to the third. All of a user's files and folder tree on one shard, because folder listing is a range scan and I do not want a cross-shard join. I use [[consistent-hashing|Consistent Hashing]] with virtual nodes so rebalancing moves keys rather than reshuffling the world, and I keep an explicit directory mapping, which is basically [[shard-routing|shard routing]] plus a lookup cache.

Chunk data: sharded by chunk hash over the node set, 3 replicas for hot chunks, and [[erasure-coding|erasure-coding]] at 8 plus 3 for cold chunks, which is a 1.375 times storage overhead against 3 times for replication. So cold data costs 2.2 times less to store. Since the top 5 percent of chunks carry half the traffic, and the rest is 95 percent of the volume, that is a large win and it is the single biggest lever on storage cost.

**Interviewer:** Now consistency, and I want you to be precise rather than hand-wavy.

**Candidate:** Three different answers for three different things.

Chunks: immutable and strongly consistent, which is easy. A chunk written is never modified. Once it exists and has a quorum, it is byte-identical forever. There is no "eventual consistency" problem because there is no update path.

Metadata: strongly consistent for the commit path. The `file` and `file_version` writes are in one transaction on one primary. A reader that lists a folder immediately after a commit sees it. I will not offer read-your-writes on replicas for that path.

Metadata reads on replicas: I allow replica reads for folder listings with a bounded staleness, typically under 200 milliseconds, because a user cannot tell the difference. But a user who just renamed a file and immediately refreshes will notice. The fix is a session token: after any mutation, the client carries the returned `commitToken` with subsequent reads, and the read router pins it to the primary until it is satisfied or the token expires after 30 seconds. That is read-your-writes without making the whole system primary-only. Study separately: [[strong-vs-eventual-consistency|Strong vs Eventual Consistency]] and [[replication-lag|Replication Lag]].

Cross-region: single region for writes in phase one, per your answer. Reads served from a replica region can be up to a few seconds stale on the folder tree, which I will label in the product.

---

## Phase 5: Failure, Scale, and Trade-offs (24:00 - 38:00)

**Interviewer:** Walk me through the failure scenarios.

**Candidate:** Six of them, and I will go in severity order.

**Failure one: the metadata primary for a shard dies.** This is the one that matters. A synchronous replica is promoted, typically within a few seconds, and I accept unavailability on that shard for that window. Reads and commits for other shards are unaffected, so this is a partial, not total, outage. I mitigate it with the three things that matter. Multi-subnet replication so the promoted replica is in a different failure domain. A fast promotion path, tested in a game day, because a 30 second promotion is a 30 second outage. And a write availability SLO that is 99.95 percent, not 99.99, because I am explicitly choosing single-primary over a consensus-based multi-writer.

**Failure two: an availability zone is lost.** Chunk data is already 3 replicas across 3 zones, so I lose nothing. The metadata store loses a replica, and I either promote a survivor or serve degraded reads if the primary was the casualty. Because a zone is roughly a third of the metadata fleet, I can rebalance shard primaries onto the surviving zones over the following hours without data movement, only leader reassignment.

**Failure three: a chunk node dies with data on it.** Chunks are 3-replicated, so reads are served from two surviving replicas, possibly with a small penalty. A repair service is triggered on the first failed read, re-replicates the affected chunks from a healthy replica, and marks the node as draining. Repair bandwidth is a scarce resource, so I throttle it to a percentage of cluster bandwidth, prioritizing by replica count first, then by data at risk. A node with 1 surviving replica goes first.

**Failure four: silent corruption.** A bit flip in a 50 GB video. Replicas do not save me because corruption is replicated. So there is a background scrubber that reads every chunk periodically, verifies its hash, and repairs from a peer. Scrub rate is sized to complete a full pass over the fleet every 30 days, and it is the main reason I can claim a durability number above what three replicas alone would give. This is what gets me from 4 nines toward 11.

**Failure five: a cache stampede.** Two flavors. A viral file: a URL is shared, 500,000 users open it in an hour. That is a hot key. Mitigations: the CDN absorbs it, the chunk cache absorbs the rest, and I add a request coalescer so that N concurrent misses for the same chunk result in one fill and the rest wait on it. A folder listing stampede: a parent folder with 100,000 children gets a mass rename. The Redis key is invalidated and 115,000 concurrent requests all miss. I use a distributed lock keyed on the cache key, so one request does the fill and the others block on a short poll or serve a slightly stale value. This is [[distributed-locks|Distributed Locks]] used carefully, as a mutex and not as a source of truth, and I have a lock expiry so a dead lock holder does not wedge the key forever.

**Failure six: a bad deploy corrupts metadata.** This is the one that actually loses data, and no amount of replication helps because the corruption is replicated. Mitigations: point-in-time recovery on the metadata store with 30 days of WAL, so I can roll back. Append-only, immutable versions so a bad write to `file_version` is a new row I can delete rather than an in-place mutation of history. And a quarterly restore drill, because an untested backup is a rumor.

**Interviewer:** Bottlenecks and scaling.

**Candidate:** In order of how much they worry me.

One, the metadata primary write path. Every commit is a write to one shard. At 6,000 QPS peak across 64 shards that is under 100 QPS per shard, comfortable. But the folder-listing read path and the refcount updates are hot spots inside a single user. A user who syncs 50,000 files is a single-shard storm. I mitigate with the listing cache and by rate-limiting per-user write bursts.

Two, the chunk index in memory. It is a single logical service; I shard it by hash prefix and the shard key for a chunk is the same everywhere, so a lookup is a single-shard hop. Its risk is a full restart, which empties a 160 GB working set. That is what the Bloom filter and the on-disk index are for, and I keep one warm replica per shard.

Three, egress. 560 Gbps at peak. This is a money problem more than a technical one, and the levers are cache hit ratio, compression for text, and cold-storage policy for the long tail.

Four, the CDN origin shield. If I ship a file that is not edge-cached, the origin sees the full request. I use a shield layer so that a cache miss at a distant edge is a miss once at the shield, not once per edge.

**Interviewer:** How does this scale as traffic doubles?

**Candidate:** Doubling traffic to 400 GB per day of ingest and 3 PB per day of egress does not change the architecture, and I want to be explicit about which numbers are already headroom and which are the actual constraint.

Chunk writes go from 6,000 to 12,000 QPS peak. Trivial. Metadata commits go from 6,000 to 12,000 QPS across 64 shards. Trivial. So the honest answer is that the write path is not the scaling problem here, and a candidate who only talks about adding metadata shards has misread the workload.

What does change: egress doubles to 1.1 Tbps, and that is a contractual and cost problem. Metadata storage grows from 30 TB to 60 TB, so I add shards, and each shard is an independent primary set, so that is a clean horizontal add. The chunk index doubles to 320 GB of hot set, so I go from 40 nodes to 80. And the number of chunk nodes grows roughly linearly with volume, which is where [[shard-rebalancing|Shard Rebalancing]] earns its keep: new nodes join, and consistent hashing moves only their share of the keyspace rather than reshuffling 260 TB.

The two things that would genuinely change the design. First, if the file size distribution shifted so that the majority of bytes were in many mid-size files, fixed chunking would become the wrong choice and I would move to CDC everywhere. Second, if a customer required cross-region write, I would have to confront the multi-writer problem, and I would look at a cell-based architecture rather than a single global namespace, because a single globally-consistent namespace for a 500 million user file system is a real tax.

**Interviewer:** Push me on the alternatives. Why not just use S3?

**Candidate:** Two answers, depending on the interview. If the interviewer means "why not use managed object storage as a dependency", the honest answer is cost and control: at 260 TB logical, 470 TB physical, with a 1.5 to 1 read amplification in a sync product, per-request pricing and per-GB egress fees are the dominant line item in the P&L, and this workload's access pattern, mostly sequential multi-chunk reads of a small number of files per user, is the worst case for a request-priced API.

If the interviewer means "why not build a single big block store like HDFS instead of per-file chunk placement", the answer is the failure domain and the unit of work. A HDFS-style design is superb for a few thousand huge append-heavy files processed by a batch pipeline. This is the opposite: tens of billions of small, immutable, randomly accessed objects across 100 million independent users, where a single file must be independently durable and independently deletable. Per-chunk placement gives me independent durability and independent refcounted deletion, which a block-oriented store with a single replication pipeline cannot give me without significant work.

**Interviewer:** Why not one object per file? Why chunk at all?

**Candidate:** Three reasons. One, resume. Re-uploading a 50 GB file from 90 percent is unacceptable; chunking makes the unit of retry 4 MB. Two, dedup. Chunk-level content addressing means two files that share 80 percent of their content share 80 percent of their storage. Three, parallelism. A single object is one stream; chunks give me 8 to 16 parallel connections, which is a 5 to 10 times throughput win on a high-latency client connection.

The cost is real and I will name it. More requests, a manifest to store, and a consistency window where the manifest and the chunks can disagree. I accept all three.

**Interviewer:** And a hot folder with 100,000 files?

**Candidate:** The listing read is a range scan paginated by cursor on `(name, file_id)`, so it is O(page size) with an index seek, not O(folder size). The metadata shard owning that parent is a single point of contention, and the listing cache absorbs the repeat traffic. If one folder genuinely gets 100,000 QPS, which a viral shared folder could, I add a read-replica promotion scoped to that one parent, and I pre-warm the cache on a change signal rather than on expiry. The write side, a 100,000-file rename, is a batched metadata operation with a job id, not 100,000 individual calls.

**Interviewer:** Now versioning and conflict resolution, properly.

**Candidate:** Identity and version identity are separate, and getting this right is most of the design.

A `file_id` is the stable identity of the logical file. A `version_id` is one immutable snapshot. Every commit creates a new `version_id` and points `file.current_version` at it. Nothing is ever mutated in place. That means version history is free, restore is a pointer move plus a new commit, and I can serve an old version for direct download without a copy.

The client sends `baseVersionId` at upload-create time. The commit transaction checks `file.current_version == baseVersionId`. If it matches, clean commit. If not, this is a conflict and the commit returns 409 with both manifests. For a text file the client does a three-way merge against the base, and if the merge is clean it retries the commit with the current version as the new base. For a binary file the server keeps both and names the loser `report (conflicted copy 2026-09-29 1432).pdf`.

The two properties I deliberately gave up. First, I do not do last-writer-wins silently, because silently discarding one device's work is the failure mode users actually hate. Second, I do not do automatic server-side merge of binary content, because a wrong merge of a 40 MB video is worse than a conflict prompt. If the product wanted character-level co-editing of text, that is a different design and a different concept: Study separately: [[crdt|CRDT]].

**Interviewer:** Dedup and garbage collection.

**Candidate:** Dedup is scoped. Global dedup across 500 million users leaks information, because chunk hashes are shared between tenants, and that is a privacy problem for shared folders and a compliance problem generally. So dedup scope is per-tenant, and optionally per-shared-folder. That is a real cost trade, since cross-user dedup would be better for storage, and I am trading a few percent of storage savings for not building an inference channel.

Garbage collection is a mark-and-sweep with an explicit grace period. Mark: a full scan of live file versions to a mark set, or more practically, the durable `chunk_ref` table maintained transactionally, which is already the answer to "is anything referencing this". Sweep: any chunk with refcount zero and last-referenced time older than 7 days becomes a deletion candidate. The 7-day grace exists because of exactly one case: a concurrent commit that read a refcount of one, then the version is deleted, then the sweep runs. The grace window plus the transactional refcount on the same shard closes that race.

The sweep runs at a rate bounded by cluster free capacity. If I am at 85 percent utilization the sweep accelerates, and above 90 percent the ingest path applies backpressure to clients before the cluster fills. Deleting data I cannot delete is the one unrecoverable error in this system.

---

## Phase 6: Follow-ups and Final Summary (38:00 - 45:00)

**Interviewer:** Rapid fire. One hot key, a file with 500,000 concurrent readers. What happens?

**Candidate:** CDN absorbs the vast majority because the cache key is the content hash, and it is immutable so it never needs invalidation. Origin sees a long tail of misses. The chunk cache serves those. Request coalescing collapses concurrent misses on the same chunk into one fill. And per-client rate limiting on that file caps abusive clients without penalizing normal ones. Study separately: [[hotspot-handling|Hotspot Handling]].

**Interviewer:** The index is down. Can I still upload?

**Candidate:** Yes, and this is why the index is a separate service. The upload session state and the durable refcount live in the transactional metadata store, not in the index. What degrades is the dedup probe, so newly uploaded chunks are stored without dedup benefit, and the placement allocation has to fall back to a round-robin or a durable index read. Chunk writes continue. New file versions may be created from chunks that already exist, so I can store a duplicate rather than a pointer to a chunk I am not sure about. Storing a duplicate is safe. Storing a pointer to a chunk that does not exist loses data, and that asymmetry drives the fallback.

**Interviewer:** Why is your metadata store relational when everything else is NoSQL?

**Candidate:** Because the metadata workload is transactional and relational in shape. Folder rename is a multi-row update that must be atomic with the unique-name constraint. A commit updates a version, a file pointer, a session row, and a set of refcounts, and I want that to be one transaction with real isolation rather than a hand-rolled saga. The access pattern, range scan by parent plus point lookup by id, is exactly what a B-tree index is good at, and the data, 10 TB across 64 shards, is well within what sharded Postgres or MySQL handles. I am not storing blobs in it, which is the actual reason people reach for NoSQL, and I have separated that out.

**Interviewer:** What is the one number you would watch most closely?

**Candidate:** Scrub completion rate versus corruption detected, and free capacity versus sweep rate, in that order. The first one is what separates 11 nines from 4 nines, and the second is the only number in this system where running out causes unrecoverable loss.

**Interviewer:** Give me the final architecture in ninety seconds.

**Candidate:** The client chunks and hashes every file, fixed 4 MB chunks, content-defined for large media. It creates an upload session, which returns signed part URLs and the chunk size chosen by the server. Each chunk is probed for existence: a Bloom filter for definite-absence, an in-memory index for maybe-present. Existing chunks are skipped entirely, which is what makes re-sync free. Missing chunks upload directly to three storage nodes chosen by consistent hashing, and a node acks only after a fsync plus a quorum. Upload completion is one transaction on a shard keyed by owner id, and it is the visibility boundary: the version row, the file pointer, and the chunk refcounts all flip together. Everything else, indexing, search, thumbnails, tiering, is a Kafka consumer downstream of the commit event.

Reads go the other way. Metadata is served from a Redis listing and metadata cache with a read-your-writes token to pin the primary after a mutation. Chunks are served from the CDN keyed by content hash, with a request coalescer in front of the origin chunk cache, and a background scrubber reading every chunk daily to catch silent corruption.

The three trade-offs I accepted. Single-writer metadata means a shard-level write outage of a few seconds during promotion, and I did not pay for multi-writer consensus. Per-tenant rather than global dedup means a few percent more storage, in exchange for not leaking content across a tenant boundary. And 4 megabyte fixed chunks for general files means weaker dedup on edited large media, in exchange for server-controlled chunk sizing and no client CPU.

---

## Concepts to Study Separately

- [[chunking-and-uploads|Chunking and Resumable Uploads]] - the commit protocol and checkpoint bookkeeping in detail
- [[content-addressable-storage|Content-Addressable Storage]] - Merkle trees, validation, and where CDC belongs
- [[outbox-pattern|Outbox Pattern]] - guaranteeing the commit event matches the committed state
- [[erasure-coding|Erasure Coding]] - 8 plus 3 versus replication, repair cost, and the 11-nines math
- [[rpo-rto|RPO and RTO]] - what my backup and restore targets actually are
- [[soft-delete-audit-tables|Soft Delete and Audit Tables]] - trash, retention, and legal hold
- [[caching|Caching]] - negative caching and Bloom filter sizing
- [[distributed-locks|Distributed Locks]] - when a mutex is safe and when it is a liability
