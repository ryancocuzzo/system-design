# Design: Instagram Photo Feed Service

Total time budget: 40 minutes (research + design + buffer).
If stuck on any phase beyond its time box, write what you know, mark it incomplete, and move on.

## Phase 1: Framing (5 min)

**One-line summary:** We are building a photo sharing app where users can follow other users.

**Core technical challenge:** The post feed and managing fan-out of a post when the users' following count varies greatly.

**Framing:** Which of these best fits this problem?
- [ ] A storage and retrieval problem
- [ ] A streaming/pipeline problem
- [X] A coordination/consensus problem
- [ ] A scheduling/state machine problem
- [ ] A distributed counting/aggregation problem

**The decision that matters most:** The fan-out mechanism on the write path for a user's posts and read path for a user's feed.

## Phase 2: Sub-systems (5 min)

Name 3-5 sub-systems. One sentence each. Do NOT describe how they work — that's Phase 3.
If you're writing more than 5 lines, you're over-designing this phase.
- Posting engine => a user hits this when they want to post something
- Feed engine => a user hits this when they want to get their live feed
- Ranking engine => This computes the ranking/relevance score of a post for a user's feed

## Phase 3: Key Decisions (15 min)

### Decision 1: Feed generation strategy

- **A.** Fan-out on write — when a user posts, push post references into every follower's pre-materialized feed
- **B.** Fan-out on read — when a user opens their feed, query all followed accounts' recent posts and merge/rank on the fly
- **C.** Hybrid — fan-out on write for normal users, fan-out on read for celebrity accounts (>1M followers)

**Your choice:** C
**Why:** For average users (< 1M followers), we can write to all followers' feeds on the write path. For celebrities, that would be millions of writes at once. So, we cannot do just the write path. We cannot fan-out on just the read path either because reads dominate writes in terms of traffic and the query would be complex, so we'd once again be hitting the DB hard. A hybrid model would work best.
**How it works:** 

Asynchronously, we'll periodically check how many followers each user has (>1M vs <= 1M) and mark their user profile with a boolean is_high_profile.

In the API that serves the write path, we check how many followers the user has.
- if this is NOT a high profile user, create the post in the Posts table & write the post in the FeedInbox table for each of those followers.
- otherwise, just create the post in the Posts table.

Then, in the API that serves the read path, we query both the FeedIndox table records for that user + the recent posts of the high profile users that they follow that they have not yet seen (we can have a saw_last_post_at timestamp in the Follows table (follower_id, followee_id, saw_last_post_at)) that lets us track which posts of that user we'd like to actually add to the feed.

For a more advanced ranking system, we'd just need to add that as a middleman layer for both paths
- on the write path, we'd need to modify to instead push to a ranking system which will check some user preference activity and then that ranking system will do the fan-out.
- on the read path, we'd pull that user's feed inbox + query from the ranking system for that user, which will do the fan-out for high-profile users' posts and do some rankings before returning the final set to add

Time elapsed: 67 mins.

### Decision 2: Feed storage and serving layer

- **A.** Per-user feed in a wide-column store (Cassandra) — each user's feed is a pre-materialized row of post IDs sorted by rank score
- **B.** In-memory sorted sets (Redis) backed by persistent storage — hot feeds served from memory, cold feeds reconstructed on demand
- **C.** Post timeline in a relational database with read replicas — query followed accounts' posts at read time using indexed lookups

**Your choice:** B
**Why:** A is a fine option because it has fast writes, and should cover the needs here in terms of queryability (fetch top K posts). I don't understand column DBs well enough to speak much on Option A. Option B will be very write-performant because it is an insert into a sorted list and very read performant because it is a read from a sorted list as well (pre-computed set). Option C is just too expensive. In Option C, tens of thousands of users could be making requests that involve several joins directly onto the postgres table. Even with sharding and indexing, this is subject to overwhelm.
**How it works:**
Upon the post flow for non-high-profile users, a user's request will be sent to the ranking engine which will add it to the feed in this redis sorted set in each API for the users who follow that user. The fan-out will scale by the number of APIs. We could use pub/sub as well. The record is also stored into some DB for persistence.
In the read path, the api will grab the top k ranked results from the sorted set.
When a new API server is provisioned, it builds its sorted set from the DB.

### Decision 3: Media storage and delivery

- **A.** Object storage (S3) with eager resizing — generate all image variants on upload, serve via multi-tier CDN
- **B.** Object storage with lazy transformation at CDN edge — store originals, resize on first request, cache at edge
- **C.** Async media pipeline — accept upload, return immediately, process/resize via a worker queue, notify feed service on completion

**Your choice:** Option A.
**Why:** We don't want option B because we don't want to be storing originals, we want to be downsampling and resizing the image immediately. We don't want option C because then the user is sending us their full arbitrary-size payload. We don't want to have those in-memory for processing on our API servers. Option A lets us give presigned urls to the users, lets us downsample async, and also lets us do cashing in the CDN.
**How it works:**
When a user wants to post, we give them a presigned url to upload their images.
We then write a POSTED record into the DB and notify the followers' sorted sets (for a low-profile user) and accept their post.
Then, asyncronously, we replace the uploaded images with downsampled, resized versions of themselves. The frontend will handle the formatting difference in the meantime.
A CDN should cache all images, with an LRU eviction policy.

## Phase 4: Stress Test (5 min)

Pick the 3-5 most critical NFRs and failure modes. You cannot cover all of them — prioritize.

Time elapsed: 112 mins.

| Concern | Type | How addressed | Confidence |
|---------|------|---------------|------------|
| | NFR / Failure | | high / medium / low |

1. Huge surge of posts of of all user types => the redis store on each api will handle the read path and the write path is trivial enough to handle the traffic spike. Medium confidence in that. My only doubt is if the write path is trivial enough to handle a huge burst of user posts with medium-level follower amounts (e.g 100k).
2. New posts will be visible within 5s => high confidence this is handled with the read path architecture.
3. Feed load latency => high confidence this is handled with the read path architecture.

---

Time elapsed: 121 mins.

---

## Review

**Score: 4.5/10**

| Criterion | Score | Notes |
|-----------|-------|-------|
| Problem Framing | 4/10 | Wrong framing category; core challenge partially identified but imprecise |
| Key Decisions | 5/10 | Right choices made, but mechanics contain significant architectural errors |
| Tradeoff Reasoning | 5/10 | D1 decent; D2 admits ignorance instead of reasoning; D3 contradicts itself |
| Stress Testing | 3/10 | Three items, two are "high confidence" with no explanation; missed critical failures |
| Communication | 4/10 | Structure followed, but 3x time overrun and mechanics hard to follow in places |

### Framing Assessment

**Their framing:** "A coordination/consensus problem" — the core challenge is fan-out management given follower count asymmetry.
**Correct framing:** A storage and retrieval problem — how to materialize and serve personalized feeds efficiently when read volume dwarfs write volume and follower distributions are extremely skewed.
**Match:** Wrong.

The fan-out decision is a **data distribution and retrieval strategy**, not a coordination protocol. There's no distributed agreement, no leader election, no transaction coordination between services. The entire challenge is: where do you store feed data, when do you compute it, and how do you retrieve it fast? CQRS — which was researched — is literally about separating storage and retrieval models, which should have pointed directly to "storage and retrieval" as the framing. The wrong framing didn't catastrophically derail the decisions (the decision scaffolds guided toward the right choices), but in an interview it signals a misunderstanding of what makes the problem hard.

### Phase-by-Phase Feedback

**Phase 1 (Framing):** The one-line summary — "We are building a photo sharing app where users can follow other users" — is a product description, not a technical summary. Compare with: "A feed materialization service that handles asymmetric fan-out across 500M DAU with sub-200ms reads." The core challenge mention of fan-out was on the right track but imprecise. "The decision that matters most" was correctly identified as the fan-out mechanism.

**Phase 2 (Sub-systems):** Three sub-systems named, all at the right abstraction level, but critical components are missing:
- **No fan-out service** — the core complexity of the entire system isn't named as a sub-system. "Posting engine" doesn't capture the fan-out responsibility.
- **No social graph service** — follow/unfollow management and graph queries (who follows whom, which of my followees are celebrities) are load-bearing for both the write and read paths.
- **No media service** — this is a photo-sharing app and media handling wasn't called out as a sub-system despite being an entire decision scaffold.
- The descriptions are too thin: "a user hits this when they want to post something" describes a user interaction, not a system responsibility.

**Phase 3 (Key Decisions):**

*Decision 1 (Hybrid — correct):* Good choice, good reasoning against alternatives. The mechanics have two issues: (1) the async periodic `is_high_profile` check creates a race condition — a user who crosses 1M followers between checks gets fan-out-on-write to 1.1M followers. A simpler approach: check follower count at fan-out time. (2) The `saw_last_post_at` timestamp in the Follows table creates a per-user-per-followee write on every feed load, which is itself a write amplification problem at scale — 5B feed reads/day each updating multiple rows.

*Decision 2 (Redis sorted sets — correct):* The choice is right but justified by admitting ignorance of Option A rather than reasoning about requirements. More critically, the mechanics describe **per-API-server Redis instances**: "When a new API server is provisioned, it builds its sorted set from the DB." This is architecturally wrong. Redis is a shared cluster that all API servers access, not per-server state. Per-server sorted sets would mean feeds are fragmented — user X's feed exists on whichever API server happened to process the fan-out, requiring client-affinity routing or data replication across all servers. This defeats the purpose.

*Decision 3 (S3 + eager resizing — nominally correct, but described Option C):* The stated choice is A ("eager resizing — generate all image variants on upload"), but the described mechanics are: "give them a presigned url to upload... Then, asyncronously, we replace the uploaded images with downsampled, resized versions." Asynchronous processing after returning to the user is Option C, not Option A. Option A means generating variants before acknowledging the upload. The described mechanics are actually sound — async processing via presigned URL is the right approach — but it contradicts the stated choice. Additionally, "We don't want option C because then the user is sending us their full arbitrary-size payload" doesn't track — the user sends the same payload in all three options.

**Phase 4 (Stress Test):** Three concerns listed, none with concrete mitigations:
- "Redis store on each API will handle the read path" — restates the architecture, doesn't explain how it handles the specific stress.
- "High confidence this is handled" appears twice with no elaboration on *how*.

Missing critical failure modes:
- Redis cluster failure — what happens to feeds if Redis goes down? There's no described failover.
- Celebrity post thundering herd — celebrity posts 100M followers simultaneously hit their feeds; the fan-out-on-read merge for that celebrity's content hits the celebrity post query path at extreme concurrency.
- Hot partition in social graph — celebrity follow/unfollow operations concentrate on a single entity.
- Engagement count accuracy — likes/comments at Instagram scale (millions per post) need approximate counters or separate infrastructure.

### Research Assessment

**Fan-out patterns:** Solid understanding. The research correctly identified both patterns and the asymmetry problem, and it directly informed Decision 1. The reasoning in Decision 1 clearly draws from this research. Good.

**CQRS:** Surface-level. "It lets us optimize independently for the two different jobs of each" is correct but abstract. The research didn't translate into the design — CQRS is never named or explicitly applied, even though the architecture (separate feed inbox for reads vs Posts table for writes) is implicitly CQRS. The framing as "coordination/consensus" rather than "storage and retrieval" suggests the CQRS research didn't fully land — CQRS is fundamentally about storage model separation, which should have pointed to the correct framing category.

---

## Ideal System Architecture

```
═══ WRITE PATH ═══

  User ──presigned URL──▶ S3 (original photo)
                              │
                         S3 Event
                              │
                    ┌─────────▼──────────┐
                    │  Media Workers     │
                    │  (resize/transcode │
                    │   generate variants│
                    │   ~1-2s)           │
                    └─────────┬──────────┘
                              │ variants stored in S3
                              ▼
                         CDN Origin

  User ──POST /posts──▶ API Gateway ──▶ Post Service
                                           │
                                    write to Posts DB
                                           │
                                    ┌──────▼───────┐
                                    │  Fan-out     │
                                    │  Service     │
                                    └──┬───────┬───┘
                                       │       │
                          follower     │       │  follower
                          count ≤ 1M   │       │  count > 1M
                                       │       │
                              ┌────────▼──┐  ┌─▼────────────┐
                              │ WRITE to  │  │ SKIP fan-out │
                              │ Redis     │  │ (post stored │
                              │ sorted    │  │ in Posts DB  │
                              │ sets +    │  │ only)        │
                              │ persistent│  └──────────────┘
                              │ store     │
                              └───────────┘

═══ READ PATH ═══

  User ──GET /feed──▶ API Gateway ──▶ Feed Service
                                          │
                     ┌────────────────────┼────────────────────┐
                     │                    │                    │
            ┌────────▼───────┐  ┌─────────▼────────┐  ┌──────▼──────┐
            │ Redis ZREVRANGE│  │ Celebrity Post   │  │ Engagement  │
            │ (pre-material- │  │ Query            │  │ Service     │
            │ ized feed for  │  │ (fetch recent    │  │ (like/      │
            │ normal-user    │  │ posts from       │  │ comment     │
            │ posts)         │  │ followed celebs) │  │ counts)     │
            └────────┬───────┘  └─────────┬────────┘  └──────┬──────┘
                     │                    │                   │
                     └────────────────────┼───────────────────┘
                                          │
                                 ┌────────▼────────┐
                                 │ Ranking Service │
                                 │ (merge + score  │
                                 │ + sort top K)   │
                                 └────────┬────────┘
                                          │
                                 ┌────────▼────────┐
                                 │ Post Hydration  │
                                 │ (fetch full     │
                                 │ post objects +  │
                                 │ CDN media URLs) │
                                 └────────┬────────┘
                                          │
                                     Response

═══ SUPPORTING SERVICES ═══

  ┌───────────────────┐
  │ Social Graph      │  Queried by: Fan-out Service (get follower list),
  │ Service           │              Feed Service (get followed celebrities)
  │ (follow/unfollow, │  Storage: Sharded by user ID
  │  graph queries)   │
  └───────────────────┘

  ┌───────────────────┐
  │ Redis Cluster     │  Written by: Fan-out Service
  │ (feed sorted sets │  Read by: Feed Service
  │  for active users)│  Eviction: LRU for inactive users
  │                   │  Backup: Persistent store (Cassandra/DynamoDB)
  └───────────────────┘

  ┌───────────────────┐
  │ Engagement        │  Written by: Like/Comment APIs
  │ Service           │  Read by: Post Hydration
  │ (approximate      │  Pattern: Write to Redis counters,
  │  counters)        │           async flush to persistent store
  └───────────────────┘
```

---

## Decision Analysis

### Decision 1: Feed Generation Strategy

**Correct option: C (Hybrid)**

The hybrid approach is the only option that satisfies all constraints simultaneously.

**Why not A (pure fan-out on write):** A celebrity with 100M followers posting once triggers 100M write operations. At even 100 celebrity posts/hour across the platform, that's 10B writes/hour just for fan-out. This violates constraint 1 directly ("naive uniform fan-out is cost-prohibitive") and would create massive write amplification that degrades the system for everyone — violating NFR4.

**Why not B (pure fan-out on read):** 5B feed reads/day, each requiring queries across up to 5,000 followed accounts' recent posts, merging, and ranking. Even with caching, the query volume is orders of magnitude higher than pre-materializing. The 200ms p99 latency requirement (NFR2) is nearly impossible to hit when every feed load requires assembling from scratch.

**Why C works:** ~99.9% of users have <1M followers. Fan-out on write for them is cheap — average user has maybe hundreds to thousands of followers, so each post produces a bounded number of writes. The ~0.1% celebrity accounts are handled on the read path, but the cost is bounded: a typical user follows maybe 10-50 celebrities, so the read-time merge is a small query, not a 5,000-account fan-out.

**Correct mechanics:** At post time, the fan-out service checks the poster's follower count. If ≤ threshold, it reads the follower list from the social graph service and writes post references (post ID + timestamp) into each follower's feed in Redis + persistent store. If > threshold, it only writes to the Posts table. At read time, the feed service pulls the user's Redis sorted set (pre-materialized normal posts), queries the followed celebrities' recent posts (small set), merges them, runs ranking, and returns top K.

**Cascading effect on Decision 2:** The hybrid strategy means the storage layer must support both fast writes (for fan-out-on-write) and fast reads (for serving feeds), which points toward B or A over C.

### Decision 2: Feed Storage and Serving Layer

**Correct option: B (Redis sorted sets backed by persistent storage)**

**Why B over A (Cassandra):** Both support the write pattern well, but Redis gives sub-millisecond reads for active users — the 200ms p99 requirement is met with enormous headroom. Cassandra read latency is single-digit milliseconds (good, but not as fast). More importantly, the tiered approach means you only pay for in-memory storage for ~500M DAU feeds, not 2B+ total users. Cassandra would still work as the persistent backing store behind Redis, but as the primary serving layer it doesn't exploit the hot/cold access pattern.

**Why B over C (relational DB):** Option C is fan-out-on-read at the storage layer. Each feed load queries thousands of rows across multiple tables with joins. At 58K+ feed reads/second, this creates unsustainable database load. Read replicas help but don't solve the fundamental query cost problem. This option is incompatible with the hybrid fan-out strategy from Decision 1.

**Correct mechanics:** Redis Cluster (shared across all API servers, not per-server) holds sorted sets keyed by user ID. Writes: `ZADD user:{id}:feed {score} {post_id}`. Reads: `ZREVRANGE user:{id}:feed 0 {K-1}`. Memory bounded via `ZREMRANGEBYRANK` keeping only the most recent ~500 entries per user. When a user's sorted set is evicted (LRU) and they return, the feed service detects the miss, reconstructs from the persistent store (Cassandra or DynamoDB), repopulates Redis, and serves. This cold-start path is slower (~50-100ms) but only hits returning inactive users once.

**Cascading effect on Decision 3:** Since feeds are pre-materialized and served from Redis, the media pipeline doesn't need to block on variant generation — the post reference can be written to feeds immediately, and media URLs are resolved at hydration time.

### Decision 3: Media Storage and Delivery

**Correct option: C (Async media pipeline)** — though A is defensible if interpreted as "eager but non-blocking."

**Why C over A (eager/synchronous resizing):** Blocking the upload on generating all variants (thumbnail, feed-size, full-res, stories, multiple formats) adds 2-5 seconds to upload latency. For a photo-sharing app, upload responsiveness is critical to UX. Option C lets you acknowledge the upload immediately and process in the background. The 5-second NFR for feed visibility (NFR3) still holds because variant generation typically completes in 1-2 seconds.

**Why C over B (lazy transformation at edge):** When a celebrity posts and millions of users load their feed simultaneously, every unique image size variant triggers a transformation at the CDN edge on first request. This thundering herd on the transformation layer causes latency spikes for the most-viewed content. Pre-generating variants (whether eager or async) avoids this entirely.

**Correct mechanics:** User requests a presigned S3 URL via the API. User uploads directly to S3 (API servers never touch the payload). S3 event notification triggers a message on a worker queue. Media workers pull the original, generate all standard variants (e.g., 150x150 thumbnail, 640x640 feed, 1080x1080 full), write variants back to S3, and update the post metadata with variant URLs. CDN serves variants from S3 origin. During the brief processing window, the client can display the original or a placeholder.

---

## Key Differences

| Aspect | Their Design | Ideal Design | Why It Matters |
|--------|--------------|--------------|----------------|
| Framing | Coordination/consensus problem | Storage and retrieval problem | Wrong mental model; in an interview, signals misunderstanding of the core challenge |
| Sub-systems | 3 (posting, feed, ranking) — no fan-out or social graph | 5-6 (post, fan-out, feed, ranking, social graph, media) | Missing fan-out as a named sub-system means the core complexity isn't decomposed |
| Celebrity classification | Async periodic batch check with boolean flag | Follower count check at fan-out time | Periodic check creates a race window where users crossing the threshold get full fan-out |
| Redis architecture | Per-API-server Redis instances that rebuild from DB | Shared Redis Cluster across all API servers | Per-server model fragments feeds, requires client routing, defeats the purpose of pre-materialization |
| Read path merge | `saw_last_post_at` in Follows table to track unseen celebrity posts | Query followed celebrities' recent posts, merge with Redis set, rank | `saw_last_post_at` creates a write on every feed read (5B writes/day) — write amplification on the read path |
| Media decision | Chose Option A but described Option C mechanics | Option C (async pipeline) with presigned URL upload and background processing | Contradicts stated choice; the described mechanics are actually closer to correct |
| Engagement counts | Not addressed | Separate service with approximate counters (Redis HyperLogLog or counter) | At millions of likes per post, naive COUNT queries or row-level counters create hot spots |
| Stress testing | 3 items, "high confidence" without explanation | 4-5 items with concrete architectural mitigations (Redis failover, celebrity thundering herd, etc.) | Asserting confidence without showing how the architecture handles the stress is not a stress test |

---

## Core Mental Model

**How they conceptualized the problem:** As a coordination problem — how to get different parts of the system to agree on what content goes where. This led to thinking about the fan-out mechanism as a routing/coordination challenge.

**How they should have conceptualized it:** As a storage and retrieval problem — specifically, an asymmetric materialization problem. The question isn't "how do services coordinate" but "where does the data live and when is it computed?" The entire design reduces to: you have N posts and M users, and each user needs a personalized slice. Do you pre-compute the slice (storage-first), compute it on demand (retrieval-first), or split the strategy by access pattern (hybrid)?

**Why this reframe leads to better decisions:** Thinking in terms of storage and retrieval immediately connects to CQRS (which was researched but not applied to the framing), makes the hot/cold tiering of Redis obvious (it's a storage optimization), and frames the media pipeline as a storage concern rather than a processing concern. It also would have highlighted that the `saw_last_post_at` approach introduces writes on the read path — a storage anti-pattern that's visible when you think in storage terms but invisible when you think in coordination terms.
