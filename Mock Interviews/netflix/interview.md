---
title: "Design Netflix — Interview Study Transcript"
status: active
tags: [hld, mock, netflix]
---

# Design Netflix — Interview Study Transcript

A full 45-minute interview, transcribed. Read it once for flow, then use the section headings to drill individual phases.

## Phase 1 — Requirements Clarification (minutes 0-9)

Interviewer: Thanks for joining. I'd like you to design a global streaming service, Netflix-style. Take a few minutes to make sure you understand the problem before you estimate anything. What do you want to ask?

Candidate: Let me start with the business model, because it drives the architecture. Is this subscription-only, or do we need the ad-supported tier?

Interviewer: Assume both tiers exist. Ads are in scope, but I don't want you building an ad exchange. Ad decisioning and ad insertion into the manifest, and impression counting, yes. Bidding, no.

Candidate: That helps, because the ad tier changes two things. It creates a hard requirement that the manifest is assembled at request time, since the ad break has to be interleaved with content segments, which means the manifest cannot be a static object cached forever. And it creates an extra low-latency dependency, the ad decision call, on the critical path of startup for that tier.

Interviewer: Good catch. Next.

Candidate: How does content get into the system? Studio masters delivered to us, or user uploads?

Interviewer: Studio delivery. We do the encoding. Treat encoding as a real pipeline, not a formality, but note that it is a different shape of problem from a user-upload platform.

Candidate: Then I can already contrast the two systems, which is worth doing out loud. Netflix has a tiny ingest surface and an enormous egress surface. Roughly 300 million paid subscribers against tens of thousands of titles. This is the opposite of a user-generated platform, where ingest is large and the catalog churns constantly. The design consequence is that the catalog is small and mostly immutable, so it can be aggressively cached and replicated, and almost all the engineering effort belongs on the delivery side.

Interviewer: Now the question that most candidates skip. The catalog.

Candidate: Yes. I want to ask about this carefully, because I think the catalog is not a list, it is a time-bounded territorial data problem.

Interviewer: Go ahead.

Candidate: Are content rights licensed per territory, per time window, and sometimes per language or per audio track? And does the same title have different availability in different countries on the same day?

Interviewer: Yes to all. Rights are per-territory, they expire, and some titles are only licensed for a window. Some shows are licensed with an embargo until a future date. Titles also have region-specific audio and subtitle tracks.

Candidate: So the catalog for a user is a function of three things: their account's home territory, the current timestamp, and their profile's age rating. Not one catalog. And a user traveling gets a different answer.

Interviewer: Correct. Anything about geo-blocking?

Candidate: Are we legally required to geo-block in any market, meaning a subscriber in country A must not be able to play country B's licensed content? And are there data-residency jurisdictions where the viewing data itself must remain inside the country?

Interviewer: Yes to geo-blocking in the EU, and yes to data residency in a handful of markets.

Candidate: That answers two architecture questions at once. Geo-blocking means the edge tier has to be enforceable, not advisory, so the signed URL or the manifest itself has to be scoped to an entitlement, otherwise someone shares a manifest URL and we leak licensed content. Data residency means user and viewing data is regional rather than global, and I will come back to that.

Interviewer: Now the product experience targets. I want numbers, not adjectives.

Candidate: Then let me propose SLIs.

Candidate: Startup latency: time from the user pressing play to the first decoded frame. I'll propose under 700 milliseconds at p50 and under 1.5 seconds at p99, and I will say up front that this is not one number, it is four, because a TV, a phone, a browser, and a set-top box have completely different costs. A modern smart TV app launch alone can be 400 milliseconds of the budget before we do anything.

Candidate: Rebuffering: expressed as a percentage of total watch time, under 0.5 percent, and I would treat the p99 as the number that matters, because one user who stalls six times in a session will churn while a thousand users who stall once will not show up in a mean.

Candidate: Playback failure rate: under 0.1 percent of play starts.

Candidate: Availability: 99.9 percent for playback, which is deliberately lower than I would demand for a transactional system, because the failure of a streaming service is visible but not a data-integrity event, and because I can degrade to a lower bitrate rather than fail. Catalog browsing 99.95 percent, because a user who cannot browse will not even try to play.

Candidate: Consistency: this is per-operation, and I will name three tiers now and defend them later. A user's own continue-watching row must be read-your-write, so it can be eventually consistent within about 5 seconds everywhere else. The catalog and entitlement data must be strongly consistent enough that we never serve content we are not licensed for, which means the entitlement check reads authoritative data and the cache TTL is bounded by the license expiry, not by a random number. And viewing history, watch-time analytics, and recommendation features are happily eventually consistent, minutes behind.

Candidate: And the cost framing up front, because it will drive everything. I am going to argue that this business is a bandwidth business, and if that is true then the entire architecture is an argument about how cheaply and reliably we can move 90 terabits per second to 35 million concurrent viewers.

Interviewer: That's the right frame. Estimate.

## Phase 2 — Scale Estimation (minutes 9-16)

Interviewer: Back-of-envelope. Watch time first.

Candidate: 300 million subscribers, 2 hours per day each, so 600 million hours of watch time per day.

Candidate: Concurrent viewers: 600 million hours divided by 24 hours gives 25 million concurrent on average. With the 1.4 peak factor, that is 35 million concurrent streams at peak.

Candidate: Let me sanity check that differently. If I take 300 million households and say about 1.7 people are watching at any given moment, I get 300 million times 0.12, roughly 12 percent, which is 36 million. Same order. I'll go with 35 million peak concurrent.

Candidate: Bandwidth. I need an average bitrate, and this is a distribution, not a number, so let me be explicit. An ad-supported SD-only user is around 0.7 megabits per second. A standard HD user on a good connection is 3 to 5. A 4K user can be 15 to 25. Weighting those, my blended average is about 2.5 megabits per second.

Candidate: Peak bandwidth: 35 million times 2.5 megabits per second, which is 87.5 terabits per second. Call it 90 terabits per second at peak.

Candidate: Let me check with a day total. 600 million hours times 3,600 seconds is 2.16 trillion watch-seconds. Times 2.5 megabits per second is 5.4 times 10 to the 18 bits, which is 6.75 times 10 to the 17 bytes, which is 675 petabytes per day. Divided by 86,400 seconds, that is 7.8 terabytes per second, or 62 terabits per second average, and 1.4 times that is 87. The two agree again, so I trust the average bitrate assumption.

Candidate: With the round-number check: 35 million streams at 10 megabits is 350 terabits per second, which is 4x over. If I use 35 million at 2 megabits it is 70 terabits per second, which is 0.8x. So my answer is inside a 4x band, which is the accuracy anyone can claim at this scale. The important move is that I am within a factor of four, not that I am exact.

Candidate: Now the request counts, and this is where I think most answers go wrong, because they count one request per view and stop.

Candidate: Class one, play starts. 600 million hours of viewing, average session 2 hours, so 300 million sessions per day. That is 300 million over 86,400, about 3,470 per second average, and 1.4 times that is about 4,900. I will say 5,000 play starts per second at peak.

Candidate: Class two, segment requests. At a 4-second segment, a 2-hour session is 1,800 segments. 4,900 starts times 1,800 segments is 8.8 million segment requests per second at peak. If the client uses a 2-second segment for low-latency paths it doubles. So 10 to 18 million segment requests per second. Every one of these is a CDN or Open Connect request, and essentially none of them touch my application servers.

Candidate: Class three, and this is the one that actually stresses my app tier, progress and heartbeat updates. The client reports playback position every 10 seconds. 35 million concurrent streams divided by 10 seconds is 3.5 million heartbeats per second. That is seven times the play-start rate and three hundred times a typical transactional system's scale.

Candidate: Let me put the three classes side by side, because the design follows directly from this table.

```
 Class                  Peak rate    Handler                    Latency budget
 ---------------------  ----------  -------------------------  ---------------
 Play starts            5K/s        BFF + playback service      700ms to first frame
 Segment fetches        10-18M/s    Open Connect / CDN edge    n/a, off app tier
 Heartbeats/progress    3.5M/s       ingest service, batched     200ms, fire-and-forget
 UI / rows / artwork    ~500K/s     BFF + rows service          100ms
 Telemetry upload       ~1M/s       client -> ingest -> Kafka   100ms, best effort
```

Candidate: The design conclusion: my application tier is sized by play starts, roughly 5,000 per second, plus heartbeats which I will make cheap by batching. The segment path is entirely someone else's problem, or rather entirely the edge tier's problem. That is the whole shape of the system.

Interviewer: Storage.

Candidate: Storage, and I want to make the contrast explicit.

Candidate: Catalog size. Films, series, and individual episodes as separate playable assets. Call it 50,000 assets. Average 1.5 hours.

Candidate: Masters, at roughly 6 megabits per second of high-quality source: 6 megabits times 5,400 seconds is 32,400 megabits, about 4 gigabytes per master. 50,000 times 4 gigabytes is 200 terabytes of masters. I keep masters indefinitely, because a re-encode must always be possible.

Candidate: Derivatives, and this is the number people get wrong. A Netflix-class ladder is not 8 files. It is roughly 20 renditions, because you have to cross bitrate with resolution with frame rate with codec with HDR, and the ad tier and the SD-only tier need their own low-bitrate variants. Summing the ladder, call it 60 megabits per second. 60 megabits times 5,400 seconds is 324,000 megabits, about 40 gigabytes per asset.

Candidate: 50,000 times 40 gigabytes is 2 petabytes of derivative video. With 3x replication, about 6 petabytes physical.

Candidate: Now the contrast I want to land. Six petabytes of video storage against 675 petabytes per day of egress. Storage is roughly one thousandth of one day's traffic. The entire storage bill is rounding error next to the bandwidth bill.

Candidate: So, unlike a user-upload platform, storage engineering here is not about capacity, it is about three specific things. First, how fast can I get a title to a new edge location when a license activates in a new country, which is a replication-time problem not a capacity problem. Second, per-title encoding optimization, which is a bandwidth problem stored in the encoding pipeline. Third, retention, because masters must be kept forever for re-encoding while 4K derivatives for a title that is delisted can be reclaimed.

Candidate: Metadata, and again it is tiny. 50,000 titles at 200 kilobytes each including synopsis, cast, artwork references, and 30 language localizations, is 10 gigabytes. That catalog fits in the memory of a small number of application instances. This is a genuinely different system from one with 60 billion videos, and I will design the catalog cache accordingly, aggressively, because it is small enough to be entirely resident.

Candidate: User data. 300 million profiles at 2 kilobytes is 600 gigabytes. Viewing history: 300 million users times roughly 200 titles viewed over their lifetime, at 40 bytes per row, is 2.4 terabytes. Continue-watching progress: sparse, 300 million users times about 20 active rows, is under 200 gigabytes. All manageable.

Candidate: The one that is not obvious: artwork. Tens of thousands of titles times maybe a thousand images each, every image, thumbnail, and banner, is tens of millions of small objects, and they are requested by the catalog service at enormous fan-out. That is a real problem, and it is a caching and CDN problem, not a database problem. Ten million images at 50 kilobytes is half a terabyte, served maybe 50 million times an hour.

Interviewer: Design it.

## Phase 3 — High-Level Architecture (minutes 16-24)

Interviewer: Draw the system. I want the device tier, the edge tier, and the control plane to be visibly different.

Candidate: Here is the system.

```
   DEVICES (TV / mobile / web / set-top box)   ~1B registered devices
   +---------------+   +---------------+   +---------------+
   | TV / Console  |   | Mobile / Web  |   | Set-top Box   |
   +-------+-------+   +-------+-------+   +-------+-------+
           \                  |                  /
            \                 |                 /
             \--------+--------+--------+--------/
                      |                        |
                      v                        v
          +--------------------+   +---------------------------+
          |  Device BFF Layer  |   |  CLIENT TELEMETRY         |
          |  TV BFF            |   |  (Scrubber + polly.js)    |
          |  Mobile BFF        |   |  startup, rebuffer,       |
          |  Web BFF           |   |  bitrate, CDN node, errors|
          |  (GraphQL/gRPC)    |   +-------------+-------------+
          +---------+----------+                 |
                    |  sign-in, profiles,        v
                    |  rows, playability,  +---------------+
                    |  manifest, progress    | Telemetry    |
                    v                        | Ingest + Kafka|
        =============================================================
                                  |
                                  v
                    +-----------------------------+
                    |  GLOBAL ANYCAST / EDGE ROUTER |
                    |  picks nearest serving PoP  |
                    +--------------+--------------+
                                   |
              +--------------------+------------------------+
              |                    |                        |
              v                    v                        v
    +------------------+  +------------------+     +------------------+
    |  Open Connect    |  |  3rd-party CDNs  |     |  Regional CDN    |
    |  appliances in   |  |  Akamai, Fastly, |     |  fallback        |
    |  ISP networks   |  |  CloudFront      |     |                  |
    |  (~1000 sites)  |  |                  |     |                  |
    +--------+---------+  +--------+---------+     +--------+---------+
             |                     |                        |
             +---------+-----------+------------------------+
                       |  video segments (90 Tbps, app tier not in path)
                       v
    +--------------------------------------------------------------+
    |  ORIGIN: object storage of per-title rendition ladders        |
    |  masters kept forever, derivatives tiered, 6 PB, 3x replicated |
    +--------------------------------------------------------------+
                                  ^
                                  |  fill, replica, per-title encoding output
                       +--------+-------------+
                       |  Media Pipeline      |
                       |  encode -> ladder    |
                       |  package HLS/DASH    |
                       |  encrypt DRM         |
                       |  publish -> fill     |
                       +----------------------+


    ======================= REGIONAL CONTROL PLANE (x4) ==============
                                                                   
      +----------+   +-----------+  +---------+  +----------------+  +--------+
      | Catalog |   |  Rows /   |  | Watch   |  | Entitlement /  |  | A/B    |
      | Service |   | Recommend.|  | Progress|  | Licensing      |  | Server |
      | (small, |   | (row-wise |  | Service |  | (territory,    |  |(bucket |
      |  cached)|   |  + column)|  |         |  |  effective-dt) |  | assign)|
      +----------+   +-----------+  +---------+  +----------------+  +--------+
            |               |             |                |             |
            +---------------+------+------+----------------+-------------+
                                   |
                    +-----------------------------+
                    |  Sharded stores per region  |
                    |  catalog SQL, progress KV,  |
                    |  history KV, entitlement SQL|
                    |  1 primary + 2 replicas,    |
                    |  3 AZ, async cross-region  |
                    +-----------------------------+
```

Candidate: Three things I want to name about this diagram.

Candidate: One, the BFF layer is real and it is not a vanity abstraction. TVs have no keyboard, less memory, older CPUs, different input models, and a ten-foot UI. A phone has touch, gestures, small memory, and is usually on a metered connection. If one API serves both, every response is the union of both models and the slowest device sets the contract. A [[backend-for-frontend|BFF]] per device class lets each response be small and purpose-built, which matters most on a TV where a 500-kilobyte rows payload over a slow connection is visible to the user. See [[backend-for-frontend|Backend for Frontend]] and [[api-composition|API Composition]].

Candidate: Two, the edge tier is not a line item, it is the product. Open Connect appliances are boxes Netflix places inside ISP networks, which puts capacity at the last mile instead of across transit. And the critical point: video segments never traverse my application tier. 90 terabits per second and 10 to 18 million segment requests per second, all terminating at the edge.

Candidate: Three, the control plane is regional and small. The catalog is 10 gigabytes, so the catalog service is a cache over a small store, and it is replicated per region because licensing rules and data residency demand it.

Interviewer: Walk me through the play flow. That is the one that matters.

Candidate: Five steps, and I will time each against the 700-millisecond budget.

Candidate: Step one, the device authenticates and calls the BFF to request a play. The BFF calls the Entitlement service, which is the licensing check. This is the most important correctness call in the system, and it returns a decision, a license window, and possibly a DRM token. Entitlement is a hard dependency and cannot be cached optimistically, because serving unlicensed content is a contractual and legal problem, not a latency problem.

Candidate: Step two, the Rows service assembles the personalized context: artwork assignments, continue-watching position, the description and cast for the title, and the experiment assignments. All of it from cache.

Candidate: Step three, the device requests the manifest. For the ad-free tier the manifest can be a static object at the edge. For the ad tier, the manifest is assembled per request so ad breaks interleave with content, which means it is a generated response, cached for seconds rather than hours, and it now has an ad-decision dependency on the critical path.

Candidate: Step four, the edge selects a server for the first segment. This is the step that makes or breaks startup, and I will come back to it in detail.

Candidate: Step five, playback begins and the client starts sending progress heartbeats every 10 seconds. Each heartbeat goes to the Watch Progress service, which coalesces writes in memory and flushes on a delay.

Candidate: Latency budget, end to end.

```
 Budget component                          p50      p99     Notes
 ---------------------------------------  -------  ------  --------------------
 Device app launch (out of our control)     250ms    600ms   TV worst case
 Auth + token validation                     15ms     60ms   local verify, no call
 Entitlement + licensing check               40ms    120ms   cached, bounded TTL
 Rows + artwork + progress lookup            50ms    180ms   all in cache
 Manifest fetch from edge                    25ms     90ms   nearest PoP
 Segment selection + routing decision        10ms     35ms   local, in the client
 First segment fetch (10-20MB range req)     90ms    250ms   the real variable
 Decode + first frame render                 60ms    200ms   device-dependent
 ---------------------------------------  -------  ------  --------------------
 Total                                     ~540ms  ~1535ms  vs 700ms / 1500ms target
```

Candidate: Note the shape of that table. The part we engineer, the API, is about 100 milliseconds. The part we do not engineer, device launch and decode, is over 300 milliseconds on a TV. That is a genuinely useful insight: on this system, most of the startup budget is not ours, so the highest-leverage optimization is often pushing work off the critical path rather than making our services faster.

Candidate: The one trick that matters most, and I would highlight it: do not wait for the full manifest before starting the first segment. Get the manifest and the first segment of a conservative bitrate in parallel, start decoding that, and then step up. The classic failure is a user pressing play, waiting for a full quality negotiation, and then buffering. Optimistic start, then switch.

Interviewer: Adaptive bitrate. Go deep.

Candidate: Three distinct layers, and people conflate them.

Candidate: Layer one, the per-title encoding ladder. This is the single biggest bandwidth lever in the whole company and I would lead with it. The naive approach is one global ladder, say 0.5, 1, 2, 4, and 8 megabits, applied to every title. The problem is that films have wildly different complexity. An animation encodes at 4 megabits and looks great. A recent action film at 4 megabits looks like mush. So you encode each title across a grid of bitrates, measure the quality, and pick a per-title bitrate-versus-quality curve. The result is a bespoke ladder per title that is, typically, 25 to 40 percent more efficient at equal quality.

Candidate: The concrete implementation: for each title, build a rate-quality curve per resolution using objective metrics, then run a perceptual evaluation with human raters, then pick the set of operating points on the curve. The output is a per-title JSON file that the packager uses, and it is versioned per title. Study separately: [[media-processing|Media Processing]] and [[compression|Compression]].

Candidate: The cost math, because this is the money argument. 25 percent off 675 petabytes per day is 170 petabytes per day avoided. At roughly 0.04 dollars per gigabyte that is on the order of seven million dollars per day. The encoding pipeline is a rounding error in cost and it is the highest-return component in the system by orders of magnitude. Say that out loud in the interview.

Candidate: Layer two, per-session adaptation. The client measures throughput continuously and picks a variant from the ladder, with hysteresis, so it does not oscillate between adjacent renditions, which is the classic visible artifact. A conservative startup bitrate, roughly 80 percent of measured throughput, then step up after a sustained period of headroom. Down-switch fast, on a rebuffer risk, because a stall costs more than a blurry picture. Up-switch slowly, because a bad up-switch causes the stall it was trying to avoid.

Candidate: Layer three, per-user and per-device constraints. The SD-only tier caps the ladder at the lowest variants regardless of the network. A device capability cap applies, so we do not offer 4K HDR to a 2015 TV. A user or parental setting can cap it. And a metered-mobile setting can cap it. These constraints are applied when the BFF builds the playback options, not in the client, because the client cannot be trusted and because the parental control must be enforced server-side. See [[access-control|Access Control]].

Candidate: The manifest itself, briefly. HLS and DASH, with the master manifest listing variants, each variant pointing at its segments. Segments are immutable objects, 2 to 10 seconds, with byte-range segmenting within a file as an alternative that reduces object count. For a catalog served at this scale I would use byte-range segmenting on a per-rendition file, because it turns 1,800 objects per asset into 20 objects, which makes edge filling dramatically cheaper. That is a real trade-off, and it costs me per-segment CDN cache granularity, so long-tail titles would want whole-file while hot titles want segment granularity. Study separately: [[chunking-and-uploads|Chunking and Uploads]].

Interviewer: Now the edge tier. Why do you want your own appliances, and walk me through fill and routing.

Candidate: Let me make the case, then the design, then the honest counter-argument.

Candidate: The problem being solved is cost per bit and control. If all video came from third-party CDNs at roughly 0.04 dollars per gigabyte, 675 petabytes a day is about 27 million dollars a day. That is not sustainable, and the reason it is not sustainable is transit, not the CDN's margin. A byte that crosses the public internet in three hops, from an origin in one region, across two transit providers, into an ISP, costs the most per bit. If instead the appliance sits inside the ISP's own network, the byte crosses one link, an interconnect or peering link that is far cheaper, often at a fixed cost rather than per-gigabyte.

Candidate: So Open Convert economics: fixed-cost appliances plus cheap peering, versus per-gigabyte on a third-party CDN. At 90 terabits per second, the crossover point is somewhere around 5,000 appliances. Above that, the owned fleet wins. And it also gives operational control, because we can run our own software, push new join strategies, and roll out changes without negotiating with a vendor.

Candidate: Now the design. An Open Connect appliance is a 100-gigabit-class box in an ISP's data center. I need 87.5 terabits per second of peak, at 50 percent design utilization for headroom, so about 1,800 appliances, roughly 1,000 sites with one or two per site, which matches the publicly reported scale.

Candidate: Fill. Three paths. First, Netflix-origin delivery for install, so an ISP gets the full catalog over a dedicated link before the site opens. Second, an ISP-to-ISP peering mesh, so a small ISP can fill from a large one nearby rather than from origin. Third, the appliance fetches on demand from the third-party CDN, which is the "fill from a CDN" mode and is the only option for smaller or less-connected partners. Study separately: [[cdn|CDN]] and [[erasure-coding|Erasure Coding]] for the appliance's local disk.

Candidate: Routing, which is the part that decides startup latency. The device asks a Netflix-run edge router which server to use. The router scores candidates by a combination of: appliance or CDN server that is closest by network distance, the measured health and throughput capacity of that server, whether the requested title is actually present there, and which server class the device is entitled to.

Candidate: The title-presence check is the hard part, and this is where startup latency is won or lost. The router maintains a per-site inventory, which of the 50,000 assets is present at which appliance. That is a 1,000 by 50,000 boolean matrix, which is 50 million bits, about 6 megabytes if stored densely, and it fits in the router's memory. So the routing decision is: nearest site that has the title and has capacity. If the nearest site lacks it, the fill mechanism was supposed to have pre-staged it, and if it did not, we fall back.

Candidate: The fallback chain, and this is what protects startup: try nearest appliance with the title, then a nearby appliance with capacity that will trigger a fill, then a third-party CDN edge, then a lower rendition at a closer server, and only as a last resort degrade to SD. I would also add pre-staging driven by predicted demand per site, so the top titles per site are always resident, and a pre-warm call when a title launches in a territory. See [[cache-warming|Cache Warming]] and [[consistent-hashing-load-balancing|Consistent Hashing Load Balancing]].

Candidate: The honest counter-argument, which I should raise myself.

Candidate: First, only partner ISPs host an appliance. For a user on a cable operator that will never install one, or in a country with no Netflix partner, third-party CDNs are the only option. So the answer is not "own CDN", it is a hybrid, and the router decides per request which to use. Study separately: [[cdn|CDN]].

Candidate: Second, Netflix has publicly described reducing reliance on its own CDN after a large outage where appliances were a single point of failure. That is the correct lesson and I would design around it: the appliance must be a cache, never an authority. If it fails, the request must still be serviceable from a third-party CDN, so the routing logic always has a non-appliance option. A cache that can also be a failure mode is not a cache, it is a dependency.

Candidate: Third, operational cost. A 1,000-site fleet is a real hardware and field-ops business, not a software problem.

Candidate: So my answer is a hybrid with appliance-first routing, third-party CDN as a permanent, not temporary, fallback, and a hard rule that no request may depend on any single appliance.

Interviewer: The catalog. Licensing. This is where I expect you to struggle.

Candidate: Right, let me take it seriously, because it is a data-modeling problem disguised as a business problem.

Candidate: The naive model is a titles table with a boolean per country. With 30 countries and 50,000 titles that is 1.5 million booleans, and it is wrong the moment anything has a date.

Candidate: My model treats rights as first-class, effective-dated, versioned rows.

```
CREATE TABLE titles (
  title_id        BIGINT       NOT NULL,
  type            ENUM('FILM','SERIES','EPISODE') NOT NULL,
  parent_series_id BIGINT      NULL,          -- for episodes
  season_no       INT          NULL,
  episode_no      INT          NULL,
  runtime_ms      INT          NOT NULL,
  maturity_rating VARCHAR(16)  NOT NULL,      -- content rating, NOT a rights field
  master_key      VARCHAR(512) NOT NULL,
  ladder_version  INT          NOT NULL DEFAULT 1,  -- per-title encoding opt version
  created_at      TIMESTAMP    NOT NULL,
  PRIMARY KEY (title_id)
);

CREATE TABLE territorial_rights (
  rights_id       BIGINT       NOT NULL,
  title_id        BIGINT       NOT NULL,
  territory       CHAR(2)      NOT NULL,      -- ISO-3166-1 alpha-2, or a rights-group key
  valid_from      TIMESTAMP    NOT NULL,      -- inclusive; embargo = future value
  valid_to        TIMESTAMP    NOT NULL,      -- exclusive
  rights_profile  ENUM('SVOD','AVOD','TVOD')  NOT NULL,
  max_streams     INT          NOT NULL DEFAULT 4,   -- concurrency limit per household
  geo_blocked     BOOLEAN      NOT NULL DEFAULT TRUE,
  status          ENUM('PENDING','ACTIVE','EXPIRED','TERMINATED') NOT NULL,
  PRIMARY KEY (rights_id),
  KEY idx_territory_window (territory, valid_from, valid_to),
  KEY idx_title (title_id)
);

CREATE TABLE title_localizations (
  title_id    BIGINT      NOT NULL,
  locale      VARCHAR(16) NOT NULL,   -- en-US, pt-BR, ja-JP, hi-IN, ...
  field       ENUM('SYNOPSIS','CAST','TAGLINE','DISPLAY_TITLE') NOT NULL,
  value       TEXT        NOT NULL,
  PRIMARY KEY (title_id, locale, field)
);

CREATE TABLE title_assets (                -- renditions actually published
  title_id       BIGINT       NOT NULL,
  rendition_id   INT          NOT NULL,     -- bitrate/res/fps/codec combination
  object_key     VARCHAR(512) NOT NULL,
  segment_mode   ENUM('OBJECT','BYTE_RANGE') NOT NULL,
  bytes          BIGINT       NOT NULL,
  is_hd          BOOLEAN      NOT NULL,
  max_device_tier INT         NOT NULL,
  PRIMARY KEY (title_id, rendition_id)
);
```

Candidate: Four design decisions here that I would defend.

Candidate: One, rights are rows, not columns. The query that matters is "for user U, at time now, in territory T, with profile age band B, which titles are playable", and that is a temporal range query: `territory = T AND valid_from <= now AND valid_to > now AND status = 'ACTIVE'`, plus a maturity filter. Expressing it as a temporal range query is correct and indexable; expressing it as a boolean matrix is not, because it cannot express a window. Study separately: [[time-series-at-scale|Time Series at Scale]] for the effective-dating pattern.

Candidate: Two, a rights profile, not just a territory. A title can be SVOD in one country and only ad-supported in another, or licensed for a promotional window only. So the entitlement check is a function of territory plus time plus profile plus device tier. That is why I have a profile column and a max-streams column rather than a simple yes.

Candidate: Three, the cache TTL is derived, not chosen. A territorial-rights cache entry's TTL must be `min(configured_ttl, valid_to - now)`. If a license expires in 20 minutes, the cache entry lives 20 minutes at most, and I add a safety margin. This single rule removes an entire category of "we served a title 4 minutes after its license expired" incidents. I would not rely on TTL alone either; I would push an explicit invalidation event on a rights change, and a nightly reconciliation job that compares active rights against the entitlement cache contents.

Candidate: Four, localizations are separate rows, not a JSON blob per title. Because catalog rendering is per-user-locale, and "My Netflix" plus recommendations plus search all need localized display titles. A wide-row JSON blob would be 30 times bigger than needed for the common case of one locale.

Candidate: The catalog query flow, and why the catalog is not a bottleneck.

Candidate: The full catalog is 50,000 assets. Metadata is about 10 gigabytes including localizations. Even a naive approach of loading the whole active catalog for a territory into memory is fine, because the whole thing is 10 gigabytes. So the design is: an entitlement service that owns a compact, in-memory index of active rights, roughly one entry per (title, territory, window), built at startup from the primary and rebuilt incrementally on a change stream. A request is a hash lookup plus a range compare, which is microseconds. [[caching|Caching]] turns the entire catalog problem into an in-memory set membership test.

Candidate: What is not in memory: artwork, because there are tens of millions of images and each is a separate object. Artwork is served from the edge with a CDN URL, and the image chosen per user is what makes this interesting, which is the personalization question.

Candidate: The failure I would design against, because it is the one that actually gets companies in trouble: a rights row is updated by the content-operations team, the cache is not invalidated correctly, and a title is served in a territory it was never licensed for. Defenses: TTL bounded by `valid_to`, explicit invalidation on the rights change stream, a separate deny-list checked on the critical path and updated with priority, and an audit job that replays entitlement decisions against the authoritative rights table. The deny-list is the one I insist on, because it is a small, bounded, fast-to-update set that can be checked in microseconds and that fails safe if the whole rights system is unavailable. Study separately: [[fault-tolerance|Fault Tolerance]] and [[disaster-recovery|Disaster Recovery]].

Interviewer: Watch history, continue watching, and progress. This is 3.5 million writes per second.

Candidate: Let me first show why 3.5 million per second is not the real number.

Candidate: The client sends progress every 10 seconds. 35 million concurrent streams over 10 seconds is 3.5 million per second, and that is the ingest rate. But the data has 40 bytes and a heartbeat has two purposes, liveness and position, and position only needs to be accurate to about 30 seconds for continue-watching.

Candidate: So I coalesce. The Progress service holds an in-memory map keyed by (user, title) of the latest position, and it flushes per key at most once every 30 seconds of wall time, and only if the position actually changed. 35 million streams divided by 30 seconds is 1.17 million per second before dedupe, and dedupe kills the rest, because a user watching continuously produces one write per 30 seconds, not three.

Candidate: Critically, this coalescing is safe because progress is idempotent. It is a last-writer-wins value, and an out-of-order or lost update costs at most 30 seconds of resume accuracy. So I do not need exactly-once, which is what makes it cheap. Study separately: [[idempotent-consumer|Idempotent Consumer]] and [[exactly-once-effect|Exactly Once Effect]] for the case you would need it.

Candidate: Storage, then. A wide-column or key-value store, not the metadata relational store, because the access pattern is point reads and the write rate is high and the schema is trivially simple.

```
 Key:   wp:{userId}:{titleId}
 Value: { position_ms, duration_ms, updated_at, playback_speed, device_id, completed }
 TTL:   none for in-progress; 30 days after completion
 Shard: hash(userId, titleId) % 256

 Read:  the user's Continue Watching row = an index of (titleId -> position)
       maintained by the same shard as a per-user sorted set, so "top 20 by
       updated_at" is a range read on one shard, not a scatter-gather.

 Replicas: 1 primary + 2, 3 AZ, per region
 Cross-region: async, 1-5s lag
```

Candidate: 256 shards at 1.17 million writes per second is about 4,600 writes per second per shard, which is comfortable for a key-value store. And the per-user index trick matters: without it, "show me my continue watching row" is a 256-shard scatter-gather on the hottest read in the product, and with it, it is one range read. That is the same lesson as the comment table in a user-upload system, and it generalizes: any table keyed by (user, entity) needs a denormalized per-user index, because the per-user list is always a hot read. Study separately: [[normalization-vs-denormalization|Normalization vs Denormalization]].

Candidate: Consistency, precisely. Read-your-write for the user's own continue-watching row, because the user just watched for ten minutes, closed the app, and reopened it, and seeing a stale position is an obvious bug. I get that with a 30-second client-side cache of the last known position plus a read-your-writes token, not with global consistency. The mechanism: the client sends the last progress update timestamp it knows about; if the server's value is older, it returns what the client has. That is a monotonic-read guarantee implemented with one extra integer, and it is much cheaper than routing reads to a primary. Study separately: [[strong-vs-eventual-consistency|Strong vs Eventual Consistency]] and [[replication-lag|Replication Lag]].

Candidate: Viewing history, which is different. It is append-only, it is high volume, and no one reads it synchronously. It goes to Kafka and lands in the analytics lake, and the "because you watched" rows are served from a precomputed structure. History for a user's own profile page is read from a secondary index built asynchronously. So history is a write-heavy, read-rare, latency-insensitive data type, and it gets a completely different storage treatment from progress. That contrast is the point.

Interviewer: Personalization. Rows, ranking, and artwork.

Candidate: Two ranking problems, and Netflix's own framing is the cleanest one: row-wise and column-wise.

Candidate: Column-wise, which is which title gets into the Top 10 row, is a precomputed batch problem. Run every few hours over the full catalog for each territory, produce a candidate set and a score, and store it. It is a heavy job over 50,000 titles times territories, and it is fine to be 4 hours stale because a Top 10 that is 4 hours old still looks fresh to a user.

Candidate: Row-wise, which is how do I order the items inside Continue Watching or Because you watched X, is a real-time problem because it is computed on the specific context the user is looking at right now. Candidate generation retrieves roughly 100 to 500 candidates, and a small ranker scores them. This has a hard 50 to 100 millisecond budget inside the 700-millisecond startup budget, so the ranker must be cheap, which is why candidate generation is precomputed and the ranker only re-orders a small set. See [[search-ranking|Search Ranking]].

Candidate: Artwork, and this is the detail most candidates miss entirely. Each title has hundreds of images: a hero, dozens of posters and thumbnails and stills. The artwork shown to a user is chosen to maximize the chance that user clicks. So artwork is a personalization model with its own ranking, evaluated online, and it needs a precomputed mapping.

```
 Store:  art:{userId}:{titleId} -> { asset_id, experiment_bucket, assigned_at }
 Size:   only for titles on that user's home page, ~100 rows per user
         300M users x 100 x 24 bytes = ~720 GB
 Reads:  ~1 per visible title on a page
 Writes: on page render, batched, ~2M/s
```

Candidate: The images themselves are a CDN problem, not an application problem. Tens of millions of small objects, requested at enormous fan-out, so the artwork is served entirely from the edge with immutable URLs and long TTLs, and the personalization only decides which URL. That separation is what makes it tractable.

Candidate: The failure mode to name: if artwork selection fails, the correct degradation is a default image, immediately, never an empty box and never a slow load. Image selection must never be on the critical path of the catalog render. Study separately: [[graceful-degradation|Graceful Degradation]].

Interviewer: Telemetry and analytics. This is the part that makes Netflix what it is.

Candidate: Two sources, and they are separate systems.

Candidate: Client telemetry, using an in-app agent, collects playback quality signals: startup time, rebuffer events and their durations, selected bitrate over time, throughput measurements, CDN or appliance node serving each segment, and errors with stack traces. This goes to a dedicated telemetry ingest service, which is fire-and-forget from the app's perspective, batches, and writes to Kafka. Scale: roughly 1 million events per second, easily 20 to 50 kilobytes per session. Study separately: [[observability|Observability]] and [[distributed-tracing|Distributed Tracing]].

Candidate: Business and product telemetry: play starts, completion, search queries, every row impression, every scroll, and critically every A/B exposure. Same path into the warehouse.

Candidate: The warehouse is a lake, and the reason is volume and cost. Raw events land in object storage in Parquet, partitioned by date and by territory, and they are queryable directly. Curated layers sit on top: a datasets-and-indexes layer that answers the specific questions the product and encoding teams ask, which is things like "for title T, what is the rebuffer ratio on ISP X in country Y for device class Z". Serving those queries from raw Parquet would be too slow, so there is a curated, indexed tier. The operational rule: no query touches raw data in an interactive path, and nothing ad hoc ever reads the raw lake. Study separately: [[data-warehouse-lake|Data Warehouse / Lake]] and [[oltp-vs-olap|OLTP vs OLAP]].

Candidate: The consumer of all this is the encoding pipeline. Rebuffer and bitrate data per title per network is the input to the per-title ladder decision. That closes the loop: better ladders mean less rebuffering and less bandwidth, which means better data, which means better ladders.

Candidate: And the consumer is experimentation. Thousands of experiments, each producing a control and treatment cohort, each needing a clean read on play and retention. If the analysis pipeline is wrong, the company makes a decision on noise, and that decision costs more than any infrastructure saving in this system.

Interviewer: A/B testing. Is it in scope?

Candidate: Briefly, because the isolation constraint is the interesting architectural part.

Candidate: A dedicated experiment-assignment service, called once per page render or per play start, that returns a bucket assignment for every experiment the user is eligible for. The requirement is that assignment is sticky per user per experiment, and it must be low-latency, so the assignment is computed once and cached, and the client also caches it.

Candidate: The architectural rule that matters: experiments are on the read and serve path but must never be on the data-write path in a way that can block. If the experiment service is down, every request falls to control. Failing to experiment is free; failing to serve is not.

Candidate: The measurement side is a separate warehouse problem, and the hard part is not the assignment, it is the statistical analysis and the guardrails against peeking at results before the sample is complete.

Interviewer: Let me push you. Traffic doubles overnight. What breaks first?

Candidate: Doubling means 70 million concurrent streams and 175 terabits per second, and 10,000 play starts per second. Let me be honest that my first answer is "the edge, and the edge is the only part I do not control."

Candidate: The edge. 87 to 175 terabits per second is well beyond the design point of the appliance fleet. A 100-gigabit appliance at 50 percent utilization holds 50 gigabits, and I need 1,800 of them for 87 terabits, so doubling needs 3,600. Appliances are not a same-week procurement. So the first action is not scaling, it is offloading: raise the third-party CDN share of traffic, which is elastic on a timescale of hours because you change a routing weight, not a hardware order. That is a concrete payoff of running a hybrid rather than owning the whole fleet. See [[autoscaling|Autoscaling]].

Candidate: The second thing that breaks is startup latency, not throughput. Adding capacity to a saturated edge does not make startup faster, because the routing decision now has fewer good candidates, so the router falls back to a more distant server or a lower rendition. A saturated network shows up as a slow start, not an error. This is the failure mode I would page on first, because it is invisible in error rates.

Candidate: Third, the heartbeats. 3.5 to 7 million per second into the Progress service, which I sized at 256 shards. That goes to 9,000 writes per second per shard, and the in-memory coalescing map doubles in size. That is a memory pressure problem, and the fix is to flush more aggressively, which is a trade of write efficiency for memory, and I would rather make that trade than OOM.

Candidate: Fourth, the app tier. 5,000 to 10,000 play starts per second against a BFF layer sized for 5,000. That is a horizontal scale problem and the honest answer is that the BFF is stateless, so it is a capacity and autoscaling exercise, not an architecture change. See [[horizontal-vs-vertical-scaling|Horizontal vs Vertical Scaling]].

Candidate: Fifth, and this is the interesting one: encoding. It does not scale with traffic, but it shares a budget. If traffic doubled because a new season dropped globally, and every territory needs a new rendition for it, the encoding pipeline now has an enormous fan-out job and it is competing for the same budget as everything else. So encoding is a scheduled, batched, capacity-planned workload, and the discipline is that a global drop is an encoding capacity event, not a traffic event. I plan for a 20x fan-out on launch day and I run it off-peak where I can.

Candidate: What does not break: the catalog. It is 10 gigabytes and is entirely in memory. Traffic doubles, the catalog does not care. That is the payoff of the catalog being small, and it is the reason I could spend all of my complexity budget on the delivery path and none on the catalog.

Interviewer: The primary database dies. Go.

Candidate: Which primary, because the answer differs. Let me take the Progress store first since it is the hottest and the most write-heavy.

Candidate: The Progress service holds the in-memory coalescing map, so a sudden primary loss does not lose the last 30 seconds of position for every user; that is a real, if accidental, benefit of the coalescing. The service sees write errors, starts buffering new position updates in memory with a bounded buffer, and the shard coordinator promotes the most up-to-date replica after fencing the old primary. Since position is last-writer-wins and idempotent, a brief write pause is far better than a partial write.

Candidate: Blast radius during promotion, call it 30 to 60 seconds: continue-watching positions may be up to 30 seconds stale, which is invisible. A user who just finished a title might not see it in Continue Watching for a minute. No data loss, no user-visible error, and I could make this a non-event in the UI by never showing a position update within the last 30 seconds anyway.

Candidate: The Entitlement primary, which is the one that genuinely matters. If it goes down, I must not fail open and serve unlicensed content. So: the entitlement service keeps a last-known-good deny-list and a compact in-memory rights index that survives for its TTL, which means it can keep answering for minutes without the primary. If the index is empty or stale beyond a hard bound, I fail closed for new plays and return a retryable error, and I do not fail open. Meanwhile the catalog browse surface, which does not require a per-play entitlement check, keeps serving from cache, so the product looks alive even though play is degraded. That is [[graceful-degradation|Graceful Degradation]] done properly: degrade the ability to start a new stream, not the ability to browse.

Candidate: The A/B server, which is the easy one and worth calling out as easy: on failure, assign everyone to control. Zero user impact.

Candidate: Detection and safety: I do not promote on a health check alone, because a network partition can leave two writable primaries. Promotion goes through the shard coordinator with a fencing token, so exactly one is writable. See [[split-brain|Split Brain]], [[failover|Failover]], and [[heartbeat-health-checks|Heartbeat and Health Checks]].

Candidate: Numbers: with semi-synchronous replication, RPO under a second within a region and RTO under 30 seconds. Across regions, async, so RPO is a few seconds and RTO is minutes. I would accept a few seconds of RPO on Progress, and I would not accept it on Entitlements, which is why Entitlements gets synchronous local replication and cross-region is read-only.

Interviewer: A new season drops. Every subscriber in 4 territories starts the same title. Walk me through the hot-title problem.

Candidate: The title is hot, the network is fine, and the danger is a specific one, so let me name what does not break first.

Candidate: Segments do not break. A 100-gigabit appliance serving the same title to 5 million people is precisely the case a cache is built for, and the hit ratio is near 100 percent because the content is identical and immutable.

Candidate: What breaks is startup latency, because of routing contention, and the entitlement path, because 5 million play starts per minute hit the one lookup everyone shares.

Candidate: Entitlement, concretely. 5 million play starts concentrated in a 10-minute window is about 8,300 per second, all against a handful of (title, territory) rights rows. Those rows are in the in-memory index, so the lookup is cheap, but if the entitlement service does a database read per request, that is a stampede on a few rows. The fix is the one I already designed: an in-memory rights index keyed by (territory, title), so the hot lookup is a hash hit. And the cache TTL rule of `min(ttl, valid_to - now)` already handles the correctness question for the hottest title.

Candidate: The bigger risk is the *pre-staging*. The title's renditions must be resident at the appliances that will serve it, before the demand arrives. If they are not, every viewer triggers an origin or third-party CDN fill, the origins are hammered at 90 terabits per second worth of new demand, and the fill latency shows up directly as startup latency. So the operational answer is: a launch checklist. Encode, then publish, then pre-stage to every appliance in the launch territories, verify presence via the inventory matrix, then flip the catalog entry to active, then notify. The ordering is the whole point, and the verification step is what catches a missed site. Study separately: [[cache-warming|Cache Warming]].

Candidate: And the client-side safety net, because planning fails: if the routing decision indicates the title is not present locally, the client should not block. It should start playback of the nearest available rendition, at possibly a lower bitrate, and upgrade when a better path is available. Optimistic start, which is the same trick I use for the first segment, applied at the title level.

Interviewer: Cache stampede and cold cache. And a regional failure.

Candidate: Let me take the regional failure first, because it is the one that produces a cold cache, and cold cache is worse.

Candidate: A full region loss, and I will reference the real precedent: in 2016 a third-party DNS provider failure took Netflix offline in a US region for hours, which is a reminder that DNS is a critical dependency and that anycast, redundant resolvers, and health checking of the DNS path itself are non-optional. See [[dns-load-balancing|DNS Load Balancing]] and [[disaster-recovery|Disaster Recovery]].

Candidate: The failure sequence is the interesting part. The surviving regions' traffic increases by 33 percent, which is survivable. But the failing region's playback service dies, and if the cache warm in the surviving region was sized for normal load rather than for absorbing a region, then the surviving region experiences a stampede: every cache entry that was region-local is cold, and 33 percent more traffic arrives at once.

Candidate: Defenses, in order of how much they help.

Candidate: One, and this is the big one, make the catalog data not region-local cold. The catalog is 10 gigabytes, so every region can hold a complete copy. It is replicated, not sharded, precisely so a region loss does not create a cold cache. A 10-gigabyte dataset should never be a cache-coherence problem. This falls straight out of the earlier estimate, and that is the point of estimating before designing.

Candidate: Two, the Progress and history stores are not pre-warmed, and that is fine, because they fail to a read-through on a sharded store with a hot key, not a cold-cache storm, and because losing recent progress is a small user harm. Deliberately not pre-warming the large stores is a deliberate cost saving.

Candidate: Three, throttle at the edge during the transition. As traffic shifts, I ramp the surviving region over 5 to 10 minutes rather than cutting DNS over instantly, and I enable per-user rate limits on play starts so that a returning user gets a graceful "starting up" experience rather than a hard error. See [[rate-limiter|Rate Limiter]] and [[load-shedding|Load Shedding]].

Candidate: Four, a real ramped DNS change with a TTL low enough to matter, 30 to 60 seconds, so the shift is not instantaneous, plus monitoring that watches the actual edge request rate rather than the config.

Candidate: RTO and RPO for a region: I want playback serving in under 5 minutes and full steady state under 30. RPO of a few seconds for Progress, and for Entitlements the regional store is authoritative for its own territory, so a region failure is a rights-availability problem handled by a documented runbook, not a replication problem, because you cannot fail over someone else's legal territory rights without a decision.

Candidate: Now the narrower cache stampede. Same four-part fix as always: TTL jitter so a million identical keys do not expire together, stale-while-revalidate so a stale value is served instantly while a refresh happens behind, probabilistic early expiration so only the first unlucky reader refreshes, and single-flight so a million misses produce one origin read. For the catalog specifically, I would add: pre-computed resident state, since the whole catalog fits in memory, which makes catalog stampedes largely a non-event by design. Study separately: [[caching|Caching]] and [[cache-warming|Cache Warming]].

Interviewer: Why not [alternative]? Push me on three.

Candidate: Why not skip Open Connect and just use a third-party CDN everywhere? Simpler, no hardware, no field ops, and you would be functionally fine.

Candidate: The answer is purely unit economics at this volume. 675 petabytes a day at roughly 0.04 dollars per gigabyte is about 27 million dollars a day on transit-inclusive CDN pricing. Appliances plus peering convert a large fraction of that into fixed cost. The crossover is a business decision, and I would frame it to you as: at 90 terabits per second we are well past the crossover, and the third-party CDN is retained for coverage and for failover, not for the bulk. But if you told me we were at 5 terabits per second, I would drop the appliance fleet entirely and you would be right. The lesson is that the answer is a function of volume, not a matter of virtue.

Candidate: Why not one global bitrate ladder instead of per-title encoding optimization?

Candidate: This one I would argue hard against, and the argument is quality, not cost. A single ladder cannot be right for both a low-complexity animation and a high-complexity recent action film, so you end up either wasting bits on simple content or starving complex content, and users judge the service on the worst titles. Per-title optimization gives 25 to 40 percent savings and, more importantly, makes the bad titles good. The only reason to move toward a global ladder is operational simplicity, and I would mitigate that by making the per-title ladder a data file produced by a batch job rather than a per-title code path, so operational cost is roughly the same as a global template.

Candidate: Why not multi-master writes across regions for Progress and History, since you are multi-region anyway?

Candidate: I would, and this is a case where multi-master is the right call. Progress is a last-writer-wins value, so a conflict costs at most 30 seconds of resume accuracy, which is the same tolerance I already accepted. The gains are real: a user in Europe writing in Asia, and no region-level write pause. So, active-active for Progress and History with last-writer-wins and last-write-wins-by-timestamp, replicated in both directions.

Candidate: But I would explicitly not do it for Entitlements or for the catalog's authoritative data, where a conflict means serving a title in a territory it is not licensed in, or resurrecting an expired rights row. There, one writer region, and the cost of that choice is write latency and a documented runbook. The general principle: multi-master is acceptable exactly when the conflict domain is small and the value of a lost update is bounded and small. Study separately: [[conflict-resolution|Conflict Resolution]] and [[multi-region-consensus|Multi-Region Consensus]].

Candidate: Why not store video in the relational database, and one more, why not a single service instead of the BFF, rows, entitlement, and progress services?

Candidate: Video in the database: 6 petabytes of blobs against a 10-gigabyte metadata table in the same system, which means the database's backup, restore, replication, and indexing behavior is now dictated by the video, which is immutable, huge, and read-only. Video belongs in object storage with a CDN in front; metadata belongs in a store whose backup and migration you can reason about in hours rather than weeks. Study separately: [[file-block-object-storage|File/Block/Object Storage]].

Candidate: One service: it is the failure mode I would fear most. Progress writes at 1.2 million per second, entitlement reads with sub-millisecond latency requirements, and catalog reads at 10 million per second from cache, in one deployment, means one deploy, one scaling group, one blast radius, and a single noisy-neighbor problem where a progress write storm starves an entitlement read. The one legitimate argument for consolidation is the BFF itself, which is deliberately a thin aggregation layer and not a place for business logic. So: thin BFFs, and real separation between data types with different latency and throughput profiles.

Interviewer: Where are the bottlenecks, and how do you know?

Candidate: Ranked, by blast radius, with the signal that detects each.

```
 Rank  Bottleneck                Signal                       User symptom        Primary lever
 ----  ------------------------  ---------------------------  -----------------  --------------------
 1     Edge segment capacity     per-node throughput, hit     slow startup,      3rd-party CDN share,
                                  ratio                        rebuffering        appliance procurement
 2     Startup latency budget    p50/p99 time-to-first-frame  "it just doesn't   device profiling,
                                                             start"             pre-staging, optimistic
                                                                                 start
 3     Progress ingest           write lag, buffer depth      continue-watching  more shards, more
                                                             position stale     aggressive flush
 4     Playback service + BFF    p99, saturation, HPA lag     rows slow, play    scale, response size
                                                             start slow         budget per device
 5     Encoding pipeline         queue age, per-title SLA     new titles not     pre-encoded asset
                                  miss rate                   watchable          library, schedule
 6     Entitlements              p99, index age               play fails         in-memory index
 7     Edge inventory accuracy   presence-miss rate at        fallback to         reconciliation
                                  routing time                3rd-party
 8     Telemetry ingest          dropped-event ratio          blind spot, no     sampling, drop
                                                             data               non-critical fields
```

Candidate: Two observations that I would say out loud. First, the top two bottlenecks are both about the network, not about code, and neither shows up in an error rate. Second, the one I would instrument most aggressively is number seven, edge inventory accuracy, because it is the quiet failure: if an appliance thinks it has a title and does not, the request fails late and the user sees a spinner, and it is invisible unless you measure presence-miss at routing time. That is a metric only this architecture produces, and a good candidate volunteers it. Study separately: [[bottleneck-identification|Bottleneck Identification]] and [[golden-signals|Golden Signals]].

Interviewer: Wrap up. Where does the money go, and give me the architecture in one picture.

Candidate: The money, first, because it is the shortest path to the right design.

```
 Cost driver                    Scale                    Share of cost   Design lever
 ----------------------------  -----------------------  -------------  ------------------------
 Video egress                   675 PB/day, 90 Tbps      ~85-90%        owned edge fleet,
                                                                  hybrid   per-title encoding,
                                                                          segment sizing
 Transcoding / encoding         20x fan-out on drop day   ~3-5%         per-title ladder,
                                                                  off-peak   reuse, spot compute
 Origin storage                 6 PB total               ~1%            tiered, EC on cold,
                                                                  bytes/day  keep masters forever
 App tier (compute)             5K starts/s, 1.2M wr/s   ~2-3%         coalesce writes,
                                                                  + DB         BFF response budgets
 Analytics / warehouse           ~1M events/s             ~2-3%         sampling, lake storage
```

Candidate: 85 to 90 percent of the cost is egress, so the design is an argument about egress, and every other decision is downstream of that. Per-title encoding optimization alone is worth roughly seven million dollars a day, from a pipeline that costs a rounding error. That asymmetry is the single most important fact in this design.

Candidate: Trade-offs I accepted, named.

Candidate: I traded operational simplicity for unit cost by running my own edge hardware, and I mitigated it with a hybrid so no request depends on a single appliance.

Candidate: I traded global write simplicity for regional correctness by using one writer region for entitlements, while going active-active for progress where conflicts are cheap.

Candidate: I traded freshness for latency by accepting 4-hour-stale Top 10 rows and 6-hour-stale profile models, because the alternative is a 4-hour model training loop that nobody wants.

Candidate: I traded per-request correctness simplicity for a derived-TTL rule, bounding every rights cache entry by `valid_to`, plus a deny-list as a fail-safe, because that removes a whole class of licensing incidents.

Candidate: I traded 4-second startup for reliability by starting on a conservative bitrate immediately and stepping up, because a stalled first impression is worse than a blurry one.

Candidate: The final architecture.

```
                          +------------------------------------------+
   DEVICES                |  EDGE ROUTER (anycast, ~1ms)             |
   TV | Mobile | Web      |  scores: distance x health x presence x  |
   |  |      |            |  entitlement tier  ->  server selection  |
   +--+-------+-----+      +------------------------------------------+
      |       |                    |              |              |
      |       |                    v              v              v
      |       |         +--------------+  +-------------+  +-------------+
      |       |         | Open Connect |  | 3rd-party   |  | regional    |
      |       |         | appliances    |  | CDNs       |  | CDN         |
      |       |         | ~1,000 sites |  | (fallback) |  | (fallback)  |
      |       |         +--------------+  +-------------+  +-------------+
      |       |                    \              |              /
      |       |                     +-------------+--------------+
      |       |                                   |
      |       |                    segments: 90 Tbps, 10-18M req/s
      |       |                                   |  (app tier NOT in path)
      |       |                                   v
      |       |         +---------------------------------------+
      |       +-------->|  OBJECT STORAGE: per-title ladders    |
      |                 |  2 PB derivatives, 200 TB masters,     |
      |                 |  byte-range segments, 3x replicated   |
      |                 +-------------------+-------------------+
      |                                     ^  fill / install / mesh
      |                     +---------------+-------------------+
      |                     |  MEDIA PIPELINE (per region)        |
      |                     |  per-title encode -> ladder opt ->  |
      |                     |  package HLS/DASH -> DRM -> publish |
      |                     +-----------------------------------+
      v
   +------------------------------------------------------+
   |  DEVICE BFFs:  TV | Mobile | Web    (per-device-class)|
   +--------+----------------------+----------------+------+
            |                      |                |
            v                      v                v
   +----------------+  +------------------+  +------------------+
   | Entitlements   |  |  Rows / Recs     |  |  Progress Service|
   | (territory,    |  |  row-wise rank   |  |  in-memory       |
   |  effective-dt, |  |  + precomputed   |  |  coalescing map  |
   |  deny-list)    |  |  column-wise     |  |  flush 30s       |
   |                |  |  + artwork       |  |                  |
   +----------------+  +------------------+  +------------------+
            |                      |                |
            |                      |    +-----------+-----------+
            |                      |    |  A/B Server (fail to  |
            |                      |    |  control)             |
            |                      |    +-----------------------+
            v                      v                v
   +------------------------------------------------------------+
   |  REGIONAL STORES (x4 regions)                               |
   |  catalog SQL (replicated, ~10GB)   | progress KV 256 shards |
   |  entitlement SQL (authoritative   | history KV (active-     |
   |    per region)                     |   active, 2-way)       |
   |  1 primary + 2 replicas, 3 AZ,    | multi-master OK        |
   |  async cross-region                |                        |
   +-------------------------------+-----------------------------+
                                   |
                     +-------------+--------------+
                     v                            v
          +--------------------+     +-----------------------+
          | TELEMETRY INGEST   |     |  ANALYTICS LAKE        |
          | client + server    |---->|  raw -> curated/indexed|
          | events -> Kafka    |     |  -> experiment analysis|
          +--------------------+     +-----------+-----------+
                                               |
                                               v
                                    (feeds the ladder optimizer)
```

Candidate: The two sentences I would leave you with. This is a bandwidth system, so 90 percent of the effort belongs on the delivery path and almost none of it belongs on the catalog, because the catalog is 10 gigabytes and fits in memory while the traffic is 90 terabits per second. And the system is deliberately two-tiered in its consistency: licensing and entitlement are hard, conservative, and fail closed, while history, progress, ranking, and analytics are fast, eventually consistent, and fail soft, because those are the things where a wrong answer costs nobody money and a slow answer costs the user everything.

Candidate: Study separately: [[media-processing|Media Processing]] for the per-title encoding ladder and its cost model, [[edge-computing|Edge Computing]] for the appliance fleet, fill, and routing, [[data-residency|Data Residency]] for territory and residency, [[time-series-at-scale|Time Series at Scale]] for the effective-dated rights model, and [[graceful-degradation|Graceful Degradation]] for the browse-versus-play degradation split.

Interviewer: Good hour. Thanks.

## Where This Transcript Is Thin

- Per-title encoding optimization internals: rate-quality curves, objective versus perceptual metrics, and operating-point selection. Study separately: [[media-processing|Media Processing]] and [[compression|Compression]].
- Open Connect fill topology, ISP peering mesh, and the hardware form factor of an appliance. Study separately: [[edge-computing|Edge Computing]] and [[erasure-coding|Erasure Coding]] for appliance local storage.
- The experimental statistics side: sample ratio mismatch, guardrail metrics, and sequential testing. Study separately: [[sli-slo-sla|SLI / SLO / SLA]] and [[alerting|Alerting]].
- Multi-master conflict handling for the progress store across regions, and how a conflict is detected rather than assumed absent. Study separately: [[multi-region-consensus|Multi-Region Consensus]] and [[conflict-resolution|Conflict Resolution]].
- DRM and device trust, which I deliberately excluded and which a real system cannot. Study separately: [[encryption-and-keys|Encryption and Keys]] and [[authentication-vs-authorization|Authentication vs Authorization]].

## Related

- [[problem|Problem Statement]] for this system
- [[evaluation|Evaluation and Scoring]] for this session
- [[06-hld-interview-checklist|HLD Interview Checklist]]
- [[01-rapid-revision|Rapid Revision]]
