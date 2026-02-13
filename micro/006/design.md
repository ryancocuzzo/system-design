# Design: URL Shortener Service

Total time budget: 40 minutes (research + design + buffer).
If stuck on any phase beyond its time box, write what you know, mark it incomplete, and move on.

## Phase 1: Framing (5 min)

**One-line summary:** We're designing a service that allows users to access their urls through our shortened url variant in a low-latency way.

**Core technical challenge:** Keeping a high likelihood of cache hits, mapping the user's provided short url to their true url.

**Framing:** Which of these best fits this problem?
- [ ] A storage and retrieval problem
- [ ] A streaming/pipeline problem
- [X] A coordination/consensus problem
- [ ] A scheduling/state machine problem
- [ ] A distributed counting/aggregation problem

**The decision that matters most:** The primary url mapping being in-memory to meet latency requirements.

## Phase 2: Sub-systems (5 min)

Name 3-5 sub-systems. One sentence each. Do NOT describe how they work — that's Phase 3.
If you're writing more than 5 lines, you're over-designing this phase.

- The API servers => the users pass through these
- The distributed key-value store => these store all the url mappings

## Phase 3: Key Decisions (15 min)

### Decision 1: Key generation strategy

- **A.** Base62-encode a truncated hash (e.g., MD5/SHA256) of the long URL — deterministic, no coordination, but collision risk grows with scale
- **B.** Distributed counter with range allocation — each node claims a numeric range, encodes sequentially, zero collisions but sequential output
- **C.** Pre-generated random key pool — batch-generate random keys offline, workers claim keys from the pool at write time

**Your choice:** C
**Why:** We don't need A because we don't need determinism. This is an operation we're doing exactly once. And there is no clear fallback for collision scenario. B is not an option because one of the problem constraints lists that the keys are not sequentially enumerable. C gives us randomization, but requires us to manage locks in a distributed way.
**How it works:**
We can store the generated keys in a relational DB. We can acquire a lock over that row. I don't have much hands-on practice with distributed locking, but I believe this is accomplishable with a `SELECT * FOR UPDATE`. Then, we have the next ID.

The actual key generation can be a string generator function done in batches and stored into the DB batches in an async worker job.

Total time elapsed: 46 mins.

### Decision 2: Storage engine for URL mappings

- **A.** Sharded PostgreSQL — relational, strong consistency, rich querying, mature operational tooling
- **B.** DynamoDB or Cassandra — purpose-built for high-throughput point lookups, native horizontal scaling, limited query flexibility
- **C.** Redis Cluster as primary store with async disk persistence — sub-millisecond reads, but cost and durability implications at ~18B keys

**Your choice:** C
**Why:** This feels like a tradeoff on durability vs latency, where I believe the priority here will be durability. A core value of the system is the p99 time, but with DynamoDB we'll only lose 1-5ms. We can still hit our latency goals. Added latency is the tax that we pay for DynamoDBs durability gaurantees. Using postgres is a non-starter because the roundtrip latency will be minimum 100ms every request. Redis cluster would give us better latency, but at the cost of only eventual-consistency correctness for each node in the cluster. dynamodb also, by default with the replication, could potentially handle hot key distribution. I'm not certain of that.
**How it works:**
- several dynamodb instances, each with the same data due to dynamodb's replication.
- each instance has keys stored in the form [short-url] -> [long-url]. The API servers do a direct lookup for the long url.
- API servers pick the one they'll read from by random selection.

Total time elapsed: 70 mins.

### Decision 3: Read path and caching strategy

- **A.** 301 (permanent) redirects with CDN edge caching — browsers and CDN cache the mapping, dramatically reduces origin load, but loses per-click analytics and prevents target URL changes
- **B.** 302 (temporary) redirects with application-level Redis cache — every click reaches your service (retaining analytics), Redis absorbs hot-key reads, no CDN caching
- **C.** 302 redirects with tiered caching (short-TTL CDN + application Redis) — short CDN TTL reduces origin load while retaining approximate analytics, adds invalidation complexity

**Your choice:** C
**Why:** A is good because it moves work to the CRN, but risks being stale in the event of a target url change. Per-click analytics are not mentioned as a requirement of this system, so that is a not a drawback. Option B doesn't leverage the CDN, which is an unneccessary performance hit. C seems like a good fit to get CDN performance support while minimizing the odds of an invalid mapping.
**How it works:**
- We have a redis cache on each api server.
- We have a CDN configured to be the first point users hit before our application gateway

I'm not sure how we configure a cache with our CDN. But we'd need another worker component do also manage invalidation, assuming the CDN system cannot.

A simple but effective strategy to ease the burden on the cache from the api server is to use request coalescing. I'm not sure how to apply other strategies to this example.


Time elapsed: 80 mins.

## Phase 4: Stress Test (5 min)

Pick the 3-5 most critical NFRs and failure modes. You cannot cover all of them — prioritize.

| Concern | Type | How addressed | Confidence |
|---------|------|---------------|------------|
| | NFR / Failure | | high / medium / low |

- 10M new urls created per day. A new url mapping should be no more than 50 bytes = 500MB per day. That should fit easily in one DynamoDB instance. I have high confidence in that.
- 99.99% availability. This system's availability is highly tied to its storage mechanism, DynamoDB. I have high confidence this is covered simply becuase of DynamoDB is highly available.
- A huge wave of requests for one url come in => They'll get spread out across different instances of dynamodb who all have the same data. I have medium confidence this is handled.

---

Time elapsed: 92 mins.

---

## Review

**Score: 3.5/10**

| Criterion | Score | Notes |
|-----------|-------|-------|
| Problem Framing | 3/10 | Wrong framing category; core challenge imprecise |
| Key Decisions | 4/10 | D1 correct, D2 contradictory (wrote C, argued B), D3 reasonable but hand-waved |
| Tradeoff Reasoning | 4/10 | Good elimination on D1, factual errors undermine D2 reasoning |
| Stress Testing | 2/10 | Surface-level, wrong storage math, no table format, hand-waved mitigations |
| Communication | 4/10 | Phases followed but D2 contradiction, factual errors, 2.3× time overrun |

### Framing Assessment

**Their framing:** A coordination/consensus problem, where the core challenge is "keeping a high likelihood of cache hits."

**Correct framing:** A storage and retrieval problem. The system ingests 10M writes/day but serves 1B reads/day — a 100:1 read-to-write ratio with extreme popularity skew. The core challenge is serving billions of point lookups at <10ms p99 against 18B stored mappings, where a tiny fraction of keys account for most traffic. There is no consensus to reach, no distributed agreement between nodes, no ordering guarantee to maintain.

**Match:** Wrong.

The coordination/consensus framing led to overweighting the key generation locking problem (which is a minor write-path concern) and underweighting the read path, which is where 99%+ of the system's traffic lives. The "decision that matters most" — keeping mappings in-memory — is a downstream implementation detail, not the architectural decision. The decision that actually matters most is how you architect the read path: what combination of storage, caching, and CDN absorbs 50K reads/sec with skewed access patterns at sub-10ms latency.

### Phase-by-Phase Feedback

**Phase 1 (Framing):** The one-line summary restates the product goal rather than capturing the technical challenge. "A service that allows users to access URLs through our shortened variant" — every URL shortener does this. A strong summary would be: "A read-dominated key-value lookup service where power-law access skew and 18B stored mappings make the caching and storage tier design the central challenge." The framing checkbox, core challenge, and "decision that matters most" all flow from the coordination/consensus misframe.

**Phase 2 (Sub-systems):** Only two sub-systems — API servers and a key-value store. This misses the key generation pool, cache tier, CDN layer, and TTL expiry worker. The descriptions are vague: "the users pass through these" doesn't tell you what the API servers do. A URL shortener has distinct write and read paths that deserve separate sub-system treatment. Two sub-systems for a system with 100:1 read-write asymmetry signals the read path wasn't decomposed.

**Phase 3 (Key Decisions):**

*Decision 1 (Key generation → C):* Correct choice. The elimination reasoning is sound — A has collision risk that grows with 18B keys and no clean recovery path, B violates the non-enumerable constraint directly. However, the mechanics are weak. `SELECT * FOR UPDATE` on a row is a serialization bottleneck — every write across all API servers contends on the same lock. The correct mechanic is batch claiming: workers pop 1,000–10,000 keys from the pool at once (via `DELETE ... RETURNING` or Redis `LPOP` with count), hold them in local memory, and assign them to incoming writes without any per-request coordination. The pool itself is refilled by an async worker that generates random base62 strings, checks for uniqueness, and bulk-inserts.

*Decision 2 (Storage engine → wrote C, argued B):* The answer says "Your choice: C" (Redis Cluster) but then every sentence argues for DynamoDB (B). This is either a typo or mid-reasoning pivoting without updating the choice — either way, it's a significant clarity issue in an interview setting. Reading the intent as B (DynamoDB): the choice is correct. But the reasoning contains factual errors:

- "Using postgres is a non-starter because the roundtrip latency will be minimum 100ms every request" — Postgres within a data center has 1–5ms read latency for indexed lookups. 100ms would be a cross-continent roundtrip. This error dismisses a viable option for the wrong reason.
- "Redis cluster would give us better latency, but at the cost of only eventual-consistency correctness for each node in the cluster" — Redis replicas lag, but reads from the primary are strongly consistent. This isn't "eventual consistency" in the Cassandra/DynamoDB sense. The actual problem with Redis as primary is cost ($135K+/month at 18B keys) and durability (async persistence), not consistency.
- "Several dynamodb instances, each with the same data... API servers pick the one they'll read from by random selection" — this is not how DynamoDB works. DynamoDB is a single logical service; you don't have multiple instances with replicated data that you manually route to. DynamoDB internally partitions your table and handles routing transparently. Global Tables replicate across regions, but within a region it's one endpoint.

*Decision 3 (Read path → C):* Reasonable choice. The observation that per-click analytics aren't a requirement is sharp. But the mechanics are thin: "a redis cache on each api server" is ambiguous (in-process cache? local Redis? shared cluster?), and "I'm not sure how we configure a cache with our CDN" is a gap that needed to be addressed, since CDN caching is the core of option C. The request coalescing mention connects well to the guided research.

**Phase 4 (Stress Test):** Three concerns, none in the required table format. The storage math is wrong — a mapping is roughly short_code (7B) + long_url (~200B avg) + metadata (~50B) ≈ 250B, not 50B. That's 2.5GB/day, ~4.5TB over 5 years. Still manageable, but 5× off. "DynamoDB is highly available" is circular — the question is what happens when it isn't. Missing critical concerns: key pool exhaustion under write spikes, cache stampede on a viral URL (directly relevant to the guided research), handling TTL expiry at scale across 18B mappings, and what happens if the CDN goes down and all traffic hits origin.

### Research Assessment

The guided research showed genuine understanding — particularly the cache stampede section, which correctly described request coalescing, locking, and probabilistic early expiration with clear explanations beyond surface definitions.

However, the research only partially translated into decisions:
- **Cache stampede → applied.** Request coalescing was mentioned in Decision 3 as a mitigation strategy. Good.
- **Hot partitions → misapplied.** The research described "store the key on multiple different servers and do a routing calculation," which then showed up in Decision 2 as "several dynamodb instances, each with the same data... pick by random selection." This is taking a correct concept (replicate hot keys across nodes) and incorrectly mapping it onto a managed service that handles partitioning internally. The right application would be: use DynamoDB DAX (in-memory cache layer) or application-level Redis to absorb hot-partition reads before they hit the DynamoDB partition.
- **Probabilistic early expiration → researched but not applied.** This was well-described in context.md but never appeared in the caching strategy despite being directly relevant to preventing stampedes on TTL-expiring cache entries.

---

## Ideal System Architecture

```
═══ WRITE PATH ═══

Client
  │
  ▼
API Gateway ───▶ Write Service ───▶ DynamoDB
                      │              (short_code → long_url,
                      │               created_at, ttl)
                      │
                 Key Pool (Redis List)
                 Claim key via LPOP
                      ▲
                      │ refill when low
              ┌───────┘
              │
        Key Gen Worker
        (batch-generate random
         base62 codes, check
         uniqueness, RPUSH)


═══ READ PATH ═══

Client
  │
  ▼
CDN (30s–60s TTL)
  │
  ├─ HIT ──▶ 302 Redirect
  │
  └─ MISS ──▶ API Gateway ───▶ Read Service ───▶ Redis Cache
                                                     │
                                        ├─ HIT ──▶ 302 Redirect
                                        │
                                        └─ MISS ──▶ DynamoDB
                                                       │
                                              (request coalescing
                                               + single-flight
                                               on cache miss)
                                                       │
                                                       ▼
                                              populate cache ──▶ 302 Redirect


═══ SUPPORTING SERVICES ═══

┌──────────────────────────┐   ┌──────────────────────────┐
│ Key Generation Worker    │   │ TTL Expiry Worker         │
│                          │   │                           │
│ - Generates random       │   │ - Scans DynamoDB TTL      │
│   base62 7-char codes    │   │   index for expired URLs  │
│ - Bulk uniqueness check  │   │ - Deletes from DynamoDB   │
│ - Fills Redis key pool   │   │ - Invalidates Redis cache │
│ - Alarm if pool < 10K    │   │ - Purges CDN cache entry  │
└──────────────────────────┘   └──────────────────────────┘

┌──────────────────────────┐
│ Custom Code Service      │
│                          │
│ - Validates custom code  │
│ - Uniqueness check       │
│   against DynamoDB       │
│ - Writes directly,       │
│   bypasses key pool      │
└──────────────────────────┘
```

---

## Decision Analysis

### Decision 1: Key generation — C (Pre-generated key pool) is correct

**Why C wins:** It decouples key generation from the write path. No per-request coordination, no collision retry loops, no sequential output. Workers claim keys in bulk from a pre-filled pool, so the write path is a simple "grab next key from local batch + write to DB."

**Why A fails:** Truncated hashing is deterministic — same long URL always produces the same short code. This seems efficient but creates problems: (1) collision probability grows toward birthday-paradox territory at 18B keys with 7-char codes, (2) collision handling requires check-and-retry loops on the write path, (3) it conflicts with custom codes — if two users shorten the same URL, deterministic hashing can't give them different codes. The lack of a clean collision fallback at this scale makes it operationally risky.

**Why B fails:** Constraint 2 says "short codes must not be sequentially enumerable." A distributed counter produces sequential numbers. Even base62-encoded, sequential counters yield predictable output — `Ab3xK9`, `Ab3xKa`, `Ab3xKb`. An attacker can enumerate all URLs by incrementing. You could shuffle with a bijective function (e.g., Knuth multiplicative hash), but that adds complexity for a property you get for free with random generation.

**Correct mechanics:** A background Key Generation Worker continuously generates random 7-character base62 strings, checks uniqueness against DynamoDB (batch `GetItem`), and pushes valid keys into a Redis list (the pool). Write-path servers `LPOP` 1,000 keys at a time into local memory. When a create request arrives, the server assigns the next key from its local batch — zero coordination, zero locking. An alarm fires if the pool drops below a threshold (e.g., 10K keys), triggering the worker to generate more aggressively. Custom codes bypass the pool entirely — just check uniqueness and write.

**Cascading effect:** Because key generation is offline and random, the write path needs no coordination. This means the storage engine (Decision 2) doesn't need to support atomic check-and-set or distributed locks — it only needs fast point writes.

### Decision 2: Storage engine — B (DynamoDB) is correct

**Why B wins:** This is a point-lookup workload: given a 7-character key, return the corresponding URL. No joins, no range scans, no complex queries. DynamoDB is purpose-built for this — single-digit-ms reads, automatic partitioning across 18B keys, managed infrastructure, and synchronous replication for durability. At ~250 bytes per item and 18B items, that's ~4.5TB — well within DynamoDB's capacity. On-demand pricing handles the 100:1 read-write asymmetry naturally.

**Why A is worse:** Sharded PostgreSQL can work, but at 18B rows you're managing dozens of shards, shard routing logic, rebalancing as data grows, connection pooling across shards, and failover per shard. All for a workload that doesn't need SQL's relational features — no joins, no transactions, no complex WHERE clauses. You're paying operational complexity for capabilities you don't use.

**Why C fails:** 18B mappings × ~250 bytes = ~4.5TB in RAM. Managed Redis (ElastiCache) at ~$30/GB/month = ~$135K/month just for storage. DynamoDB stores the same data on disk for a fraction of the cost. Beyond cost, Redis's persistence (AOF/RDB) is async — a primary crash before the next fsync loses recent writes. At 120 writes/sec, even a 1-second window means 120 lost URL mappings. DynamoDB's synchronous multi-AZ replication is a fundamentally stronger durability guarantee for a primary store.

**Cascading effect:** With DynamoDB as the durable store, the read path needs a cache layer to hit <10ms p99 — DynamoDB's 1–5ms single-digit latency is close but inconsistent at tail. This makes Decision 3 (caching strategy) critical.

### Decision 3: Read path — B (302 + Redis cache) or C (302 + tiered caching) are both defensible

**Why A fails:** 301 (permanent redirect) tells the browser to cache the mapping forever. This is great for load reduction but violates FR4 — URLs expire after a configurable TTL. Once a browser has cached a 301, you cannot expire that URL for that user. You also lose the ability to correct a bad target URL. The functional requirements rule out 301.

**Why B works:** 302 redirects mean every click reaches your service. A Redis cache sits in front of DynamoDB — with power-law access distribution, a relatively small cache (say 100M entries, ~25GB) captures the vast majority of reads. Cache miss rate is low because popular URLs are always warm. You retain full control over expiry and analytics. Stampede protection via request coalescing (single-flight pattern): when a cache miss occurs, only one request fetches from DynamoDB while others wait on the same in-flight response.

**Why C also works:** Adding a short-TTL CDN layer (30–60 seconds) in front absorbs traffic spikes — a viral URL that gets 100K clicks/minute hits your origin only once per CDN TTL per edge location. The cost is slight staleness (a URL expired 30 seconds ago might still serve a redirect) and cache invalidation complexity. For a URL shortener where TTLs are measured in years, 30 seconds of staleness is negligible.

**The right choice depends on whether you need spike absorption.** At 50K reads/sec peak, a Redis cluster can handle this without a CDN. But if a single URL goes viral (millions of clicks in minutes), CDN edge absorption prevents your Redis cluster from becoming the bottleneck. C is the safer choice for the stated NFRs.

---

## Key Differences

| Aspect | Their Design | Ideal Design | Why It Matters |
|--------|--------------|--------------|----------------|
| Problem framing | Coordination/consensus | Storage and retrieval with read-skew | Wrong framing led to overweighting write-path locking and underweighting the read path, where 99%+ of traffic lives |
| Sub-system decomposition | 2 sub-systems (API servers, KV store) | 6 sub-systems (write service, read service, key pool, cache, CDN, expiry worker) | Missing sub-systems means missing failure modes — no expiry handling, no key pool management, no cache tier |
| Key pool mechanics | `SELECT FOR UPDATE` per write (serial lock) | Batch LPOP of 1K keys to local memory | Per-write locking serializes all writes globally; batch claiming eliminates write-path coordination entirely |
| DynamoDB understanding | "Several instances with same data, pick by random selection" | Single managed service with internal partitioning | Misunderstanding the abstraction leads to incorrect hot-key mitigation strategy and incorrect capacity planning |
| Postgres latency | "100ms minimum per request" | 1–5ms for indexed point lookups within a datacenter | 20× overestimate; dismisses a viable option for the wrong reason |
| Read path architecture | Vague "redis cache on each api server" | Shared Redis cluster with single-flight coalescing + optional CDN edge layer | Per-server caches have cold-start and consistency issues; shared cache with coalescing handles skew and stampede |
| Stress testing | 3 surface-level concerns, wrong storage math | Prioritized: cache stampede on viral URLs, key pool exhaustion, TTL expiry at 18B rows, CDN origin fallback | Missing the hard failure modes means the design has unaddressed single points of failure |
| Research application | Hot partitions research misapplied to DynamoDB routing; probabilistic early expiration researched but unused | Hot partitions mitigated via cache tier absorbing skewed reads; early expiration applied to Redis TTLs | Research that doesn't translate to decisions is wasted time |

---

## Core Mental Model

**How they conceptualized the problem:** As a coordination problem — how do you generate unique keys without collisions across distributed nodes? This led to focusing on locking, distributed state, and the write path.

**How they should have conceptualized it:** As a read-dominated storage and retrieval problem with extreme access skew. The write path is trivial at 120 writes/sec — a single Postgres instance could handle that. What makes this hard is serving 50K reads/sec (with viral spikes far above that) at <10ms p99 across 18B stored mappings, where 1% of URLs generate 99% of traffic. The architecture is dictated by the read path: how do you layer CDN, cache, and storage so that hot keys never touch the database and cold keys still resolve quickly?

**Why this reframe matters:** When you see URL shortening as a write-coordination problem, you spend your design budget on key generation locking mechanics. When you see it as a read-skew problem, you spend your budget on cache tiering, stampede protection, and the CDN-to-origin pipeline — which is where the actual architectural complexity lives. The key generation is a solved sub-problem (batch pre-generation). The read path is where your design lives or dies.
