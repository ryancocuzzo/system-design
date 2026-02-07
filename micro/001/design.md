# Design: Metrics and Logs Ingestion Pipeline

Target time: 45 minutes total.
If stuck on any phase beyond its time box, write what you know, mark it incomplete, and move on.

## Phase 1: Problem Framing (5 min)

**One-line summary:**

We are designing a metrics ingestion engine that supports structured metrics querying, clear cost visiblity and "metrics cardinality governance"

**Core technical challenge:**

**Framing:** Queryability of the metrics

**The decision that matters most:** how metrics and their tags are stored and queried

Time elapsed: 7 mins.

## Phase 2: Sub-systems (5 min)

| Sub-system | Responsibility | Why it's separate |
|------------|---------------|-------------------|
1. Ingestion engine. users send their metrics here. it processes them. this is separate to give it one responsibility (process metrics) and achieve low-latency + loose coupling. That might be a hand-wavy explanation.
2. query engine. users hit this service to query their metrics. this is separate again mostly to adhere to single responsibility principle and loose coupling.
3. Metrics movement engine - moves metrics from "hot" storage to "cold" storage in intervals. Separate to isolate the metrics movement from the other processes that interact with the metrics.
4. Storage (hot, cold, aggregated). stores the metrics and their tags. separate because we need a dedicated persistence layer.
5. Metrics aggregation engine - turns cold metrics into aggregated, sparse metrics to be useful for long-range queries

The "Why separate?" descriptions were mostly hand-wavy, but mostly just proxies for SRP and coupling.

Time elapsed: 14 mins.

## Phase 3: Core Sub-system Deep Dive (20 min)

Pick the 2-3 sub-systems where the real complexity lives. Design those. Skip the straightforward ones.

### [Hardest sub-system]

Storage, for metrics and tags.

A metric entry itself is a name, a set of tags, a timestamp and a value.

Forgetting about performance constraints, the simple storage solution would be a metrics table in postgres where each metrics has a clientId and a MetricsEntries table where each entry (instance of a metric) has a metricId, a set of tags (array of strings), and a value (float or int).

My doubts are that the latency of hitting postgres feels high.

The components here makes me lean towards a time series DB.

We could definitely have a blob store with older artifacts. A reaonsable way to partition this would be per-client, per-day (e.g a file path would be `client-id/date/metrics-store.log`).

We could have a process that queries the metrics older than X days and moves them into a new file in that blob store.

The aggregated storage would be another postgres table that contains cumulative totals and selectively picked (via some probability model) metric entries.

I don't have a crisp picture of how the aggregated storage would represent cumulative totals.

### [Second hardest sub-system]

The query engine. This would essentially be querying the hot postgres table based on the user's query if the timeframe is within the hot data window (X days) and, if the timeframe extends past that, combining the result with the aggregated storage results.

Time elapsed: 40 mins.

## Phase 4: Integration (5 min)

How do the sub-systems connect?

| From | To | What flows | Sync/Async | Why |
|------|----|-----------|------------|-----|
1. the ingestion engine should accept metrics and publish a messaging signal to process them. the metrics are flowing async to deliver low-latency. A worker process stores those metrics into the hot postgres table used by the query engine
2. the query engine pulls data from two postgres sources, the hot and aggregated, to deliver metrics query results. this is sync to deliver instant user query results
3. the metrics movement engine moves the data from hot to cold. this is an async process to avoid interference with the core flow (querying).
4. th aggregation engine is very similar to the metrics movement engine in that it moves data from cold to aggregated, but it also performs random or probabalistic selection to decide which metrics to store and performs calculations on the true totals for usage calculations

total time elapsed: 46 mins.

## Phase 5: Addressing Non-Functional Requirements (5 min)

| Requirement | How addressed | Confidence |
|-------------|---------------|------------|
1. ingestion throughput: well-addressed. We can horizontally scale the ingestion engine because it's stateless and it's sole operation is a messaging store operation. To process a peak of 15M entries per second, assuming each server can address ~1k / second, we'd need 15k of these APIs to fully process all data every second. That smells of a design or math flaw. That would be a huge scale and cost consideration. The 2TB/day of log data imply that we'd need to shard the postgres (by client id, perhaps) and also need sharded blob store (by client id as well). We would then not need that `CLIENT_ID` portion of the blob store partition logic. We might need another mechanism, as that is a lot of data.
2. Ingestion latency: To address the metric data being available within 10 seconds, we'd need the workers to have surely picked up and stored the metric within 10s. With the number of workers mentioned earlier, we should have all data processed every second. This is likely overkill, but is definitely sufficient. 
3. Query latency: p95 queries should still be mostly hot data queries. That should easily be hit by the query engine, which is just querying postgres. The cold data scenario is that the query engine hits 2 postgres servers (hot + aggregated). That is larger, but should be done far less frequently.
4. Availability: not addressed.
5. Storage cost: not addressed.
6. Durability: not addressed.

Time elapsed: 63 mins.

## Phase 6: Failure Modes (5 min)

| Failure | Impact | Mitigation |
|---------|--------|------------|
1. Kafka goes down. Cannot process any new metrics, but old metrics are still available. Can mitigate by using an outbox pattern (changing the failure component to the db), or more instances of kafka. I'm not sure a good high-availability model for kafka.
2. hot db goes down. unable to serve metrics queries. can mitigate with read replicas + leader-election.
3. data movement processor goes down. data not being purged from postgres and costs grow. Can mitigate by scaling up the service (it's stateless so this is possible) and using concurrency locks to prevent moving the wrong data

There are other similar failure modes as well. Moving on for time reasons.

---

Time elapsed: 68 mins.

---

## Review

**Score: 3.5/10**

| Criterion | Score | Notes |
|-----------|-------|-------|
| Problem Framing | 4/10 | Queryability is part of it, but missed the pipeline/flow dimension; framing led to Postgres-centric thinking |
| Architecture Quality | 3/10 | Postgres at 5M pts/sec is a fundamental miss; no tag indexing; rollups misunderstood; no cardinality governance sub-system |
| Tradeoff Reasoning | 3/10 | No cost math despite it being explicitly required; good instinct on the 15k-server smell but didn't follow the thread |
| Failure Handling | 3/10 | Surface-level; missed the most critical failure mode (cardinality explosion) and tenant isolation |
| Communication | 5/10 | Honest about gaps; good self-awareness; some phases are stream-of-consciousness rather than structured |
| Operational Concerns | 2/10 | Multi-tenancy, rate limiting, cardinality governance, and cost modeling all absent |

### Framing Assessment

**Their framing:** Storage and retrieval problem — "how metrics and their tags are stored and queried."

**Correct framing:** Streaming/pipeline problem — the core challenge is not *where* data sits but *how it flows*: accepting 5M+ pts/sec, applying inline cardinality governance and tag indexing, routing through write-optimized hot storage, and progressively downsampling into cheaper tiers — all while keeping recent data queryable within 10 seconds.

**Match:** Partially correct.

The "storage and retrieval" framing is the most common trap for this problem. It makes you think about database schemas (Postgres tables, blob store paths) when you should be thinking about data flow, buffering, backpressure, inline processing, and transformation stages. The storage *choices* are important, but they're downstream of getting the pipeline right. If you can't ingest at rate and transform data through tiers efficiently, it doesn't matter what database you picked.

The framing error directly caused the most damaging design choices: defaulting to Postgres (a row-oriented RDBMS) for time-series writes at 5M/sec, and thinking about the query engine as "just querying Postgres" rather than as a fan-out router across heterogeneous storage tiers.

### Phase-by-Phase Feedback

**Phase 1 (Framing):** The one-line summary is reasonable but buries the pipeline nature of the problem. "Core technical challenge" was left blank — this is the most important field in Phase 1, and skipping it cascades into everything else. "The decision that matters most" — "how metrics and their tags are stored and queried" — is too vague to guide design. A sharper answer would be: *"The write path architecture for the hot tier — it must sustain 5M pts/sec with 10-second queryability, and no general-purpose database can do both."* That one sentence would have redirected the entire design.

**Phase 2 (Sub-systems):** The five sub-systems are reasonable as a starting point, but critically missing:
- **Cardinality governance** — explicitly called out in the problem as a first-class concern, discussed in context.md, and then completely absent from the architecture. This is the single most impactful omission.
- **Tag index / inverted index** — tags are the primary query dimension. Finding all series matching `env:prod AND service:checkout` across billions of series requires a dedicated inverted index. Without it, every query is a full scan.
- **Tenant isolation / rate limiting** — the problem says "fair-share resource allocation so one noisy tenant cannot degrade others." No sub-system addresses this.

The "why it's separate" column was self-admittedly hand-wavy ("mostly SRP and coupling"). The better question isn't "why is it separate for clean-code reasons" — it's "what failure or scaling characteristic forces this to be a separate operational boundary?" For example: the ingestion path must be independently scalable from the query path because they have inverse load patterns (ingestion peaks during business hours; dashboards peak during incidents, which are unpredictable).

**Phase 3 (Deep Dive):** Two critical issues here.

*Storage:* Postgres is a row-oriented RDBMS. At 5M pts/sec, you'd need roughly 5,000 INSERT operations per millisecond. Postgres, even with batching, tops out around 50-100k inserts/sec per instance on fast hardware. You'd need 50-100 Postgres instances just for writes — and you'd still have terrible read performance for time-range aggregations because row-oriented storage requires reading entire rows when you only need the value column.

The right answer is a time-series-optimized store: columnar format (data points for the same metric stored contiguously), append-only writes with in-memory buffer that flushes to disk in sorted blocks, and compression that exploits time-series patterns (delta encoding for timestamps, XOR encoding for values — achieving 1-2 bytes per data point instead of ~50). This is what systems like Gorilla (Facebook), M3DB (Uber), and Datadog's internal store do.

The blob store idea for cold storage is directionally correct. But `client-id/date/metrics-store.log` is too coarse. You'd want **columnar Parquet files** partitioned by `tenant/metric_name/time_range/` so that a query for "CPU utilization for tenant X over the last 3 months" can scan only the relevant partition rather than reading every metric for every date.

*Rollups:* The description — "selectively picked (via some probability model) metric entries" — reveals a misunderstanding. Rollups are **deterministic downsampling**, not probabilistic sampling. You take every 10-second data point in a 1-minute window and compute `{min, max, sum, count, avg}`. You keep all five aggregates so any dashboard query (average CPU, max latency, etc.) can be answered exactly from the rollup without touching raw data. Nothing is randomly selected; nothing is lost that a coarser query would need.

*Query engine:* "Just querying Postgres" hand-waves away the entire hard part. A "last 1 hour, p99 latency grouped by service" query must: (1) look up the inverted index to find all series tagged `service:*`, (2) fan out to the relevant hot-store shards, (3) scan time-range-partitioned columnar data, (4) compute percentiles across thousands of series, (5) return in < 200ms. A 30-day query must additionally hit warm/cold tiers and merge results. These are fundamentally different execution plans served by the same API — the query router's job is to figure out which tiers to hit and how to merge.

**Phase 4 (Integration):** The Kafka mention first appears here rather than in the deep dive, which suggests it was an afterthought rather than a load-bearing design choice. "Worker process stores metrics into hot postgres table" at 5M/sec — no discussion of batching, write buffering, or parallelism. No mention of where cardinality checks happen in the flow (they must happen *before* writes to the hot store, not after).

**Phase 5 (NFRs):** The 15k-server calculation is actually the most insightful moment in the design. You correctly smelled that something was wrong. The problem is you didn't follow the thread: the smell should have triggered "Postgres can't be the hot store at this write rate — what storage engine *can*?" Instead, the conclusion was "that's a lot of servers." This is the difference between identifying a symptom and diagnosing the root cause.

Availability, storage cost, and durability were all marked "not addressed." In an interview, leaving storage cost unaddressed for a problem that explicitly says "justify storage tier choices with rough cost math" would be a significant miss. A back-of-envelope: S3 is ~$0.023/GB/month, EBS SSD is ~$0.10/GB/month, NVMe instance storage is ~$0.30/GB/month. At 2 TB/day of logs alone, 15 days of hot storage is 30 TB — that's $9k/month on NVMe vs. $690/month on S3. The 13x cost difference is *why tiering matters*.

**Phase 6 (Failures):** The failures identified are real but generic (any distributed system has "what if the DB goes down"). The failure modes specific to *this* system were missed:
- **Cardinality explosion** — one customer tags by request_id, creates millions of series/hour, saturates indexing and blows out memory for all tenants. This is the #1 operational risk.
- **Agent retry storms** — if the ingestion gateway is slow, thousands of agents retry simultaneously, creating a thundering herd that can cascade into full outage.
- **Rollup lag** — if the rollup job falls behind, hot storage fills up, costs spike, and eventually writes start failing.
- **Tenant isolation failure** — one tenant's 500k-series burst crowds out another tenant's queries.

### Context Questions Assessment

One question was asked (cardinality governance) — it was a good question and the answer was clearly understood. But several critical knowledge gaps went unasked:
- **"What is a time-series database and why can't Postgres handle this workload?"** — This is the knowledge gap that most damaged the design. If you don't know why TSDBs exist, you'll default to what you know (Postgres), and the entire architecture collapses under write load.
- **"What does 'rollup' or 'downsample' actually mean mechanically?"** — The probabilistic-sampling misunderstanding suggests this question should have been asked.
- **"How do inverted indexes work for tag-based queries?"** — Tags are the primary query dimension, and no indexing strategy was discussed.
- **"What are actual AWS storage costs per tier?"** — The problem explicitly requires cost math; asking for the numbers would have enabled it.

The cardinality governance question was asked and well-answered, but then the concept was never incorporated into the architecture. Good context research that doesn't make it into the design is wasted effort.

---

## Ideal System Architecture

```
═══ INGESTION PIPELINE ═══

  Agents (HTTPS/gRPC, batched + compressed)
         │
         ▼
  ┌──────────────────┐
  │ Ingestion Gateway │
  │                   │
  │ • TLS terminate   │
  │ • Authenticate    │
  │ • Decompress      │
  │ • Validate schema │
  │ • Per-tenant rate  │
  │   limit           │
  │ • ACK to agent    │
  └────────┬──────────┘
           │
           ▼
  ┌──────────────────────────────┐
  │ Kafka (partitioned by        │
  │   tenant_id + metric_name    │
  │   hash)                      │
  │                              │
  │ • Durable buffer             │
  │ • Decouples ingestion from   │
  │   processing                 │
  │ • Absorbs burst (15M/sec     │
  │   peak vs 5M sustained)      │
  └────────┬─────────────────────┘
           │
           ▼
  ┌──────────────────────────┐
  │ Stream Processors         │
  │ (Flink or custom workers) │
  │                           │
  │ • Assign series ID        │
  │   (metric_name + sorted   │
  │    tag set → hash)        │
  │ • Update tag inverted     │
  │   index                   │
  │ • Cardinality check ◄──────────── Governance Service
  │   (HLL per metric per     │       (threshold configs,
  │    tenant)                 │        enforcement rules)
  │ • Batch writes             │
  └────┬──────────┬───────────┘
       │          │
  ACCEPT│     REJECT/AGGREGATE
       │          │
       ▼          ▼
  ┌────────┐  ┌───────────┐
  │Hot TSDB│  │Dead Letter│
  │        │  │+ Alert    │
  └────────┘  └───────────┘

═══ HOT STORE (TSDB on NVMe SSDs) ═══

  ┌──────────────────────────────────┐
  │ Write path:                       │
  │   In-memory write buffer (WAL)    │
  │   → Flush to sorted columnar      │
  │     blocks every N seconds         │
  │   → Compressed: delta timestamps,  │
  │     XOR values (~1.4 bytes/point)  │
  │                                    │
  │ Sharded by: series_id hash         │
  │ Retention: 15 days full resolution │
  │ Cost: ~$0.30/GB/month (NVMe)      │
  │ Capacity: ~30 TB (15 days × 2TB)  │
  │ Monthly cost: ~$9,000              │
  └──────────────────────────────────┘
           │
           │ (Rollup job — background, cron-driven)
           │ Reads 10s data → emits {min,max,sum,count,avg}
           │ per 1-minute window per series
           ▼
  ┌──────────────────────────────────┐
  │ Warm Store (Columnar on EBS SSD)  │
  │                                    │
  │ • 1-minute resolution rollups      │
  │ • 13-month retention               │
  │ • Cost: ~$0.10/GB/month            │
  │ • Data is ~60x smaller than raw    │
  │   (60:1 downsample + compression)  │
  └──────────────────────────────────┘
           │
           │ (Rollup job — 1-min → 1-hour)
           ▼
  ┌──────────────────────────────────┐
  │ Cold Store (Parquet on S3)         │
  │                                    │
  │ • 1-hour resolution rollups        │
  │ • 3-year retention                 │
  │ • Partitioned: tenant/metric/      │
  │   year/month/                      │
  │ • Cost: ~$0.023/GB/month           │
  │ • Queryable via Athena or Presto   │
  └──────────────────────────────────┘

═══ QUERY PATH ═══

  Client Query API
         │
         ▼
  ┌──────────────────────────┐
  │ Query Router              │
  │                           │
  │ 1. Parse time range       │
  │ 2. Resolve tags → series  │
  │    IDs via Tag Index      │
  │ 3. Determine tier(s):     │
  │    • < 15 days → Hot      │
  │    • 15d-13mo → Warm      │
  │    • > 13mo → Cold        │
  │ 4. Fan-out to tier shards │
  │ 5. Merge + aggregate      │
  │ 6. Return                 │
  └──┬───────┬────────┬──────┘
     │       │        │
     ▼       ▼        ▼
    Hot     Warm     Cold
   (<200ms) (<1s)   (<5s)

═══ SUPPORTING SERVICES ═══

  ┌─────────────────────────┐
  │ Tag Inverted Index       │
  │                          │
  │ tag_key:tag_value →      │
  │   set of series_ids      │
  │                          │
  │ Stored in: Redis cluster │
  │   or custom on SSD       │
  │ Queried by: Query Router │
  │ Updated by: Stream       │
  │   Processors             │
  └─────────────────────────┘

  ┌─────────────────────────┐
  │ Cardinality Governance   │
  │                          │
  │ • HyperLogLog per metric │
  │   per tenant (in Redis)  │
  │ • Threshold configs in   │
  │   Tenant Config service  │
  │ • Actions: alert, drop,  │
  │   strip tag, aggregate   │
  │ Queried by: Stream       │
  │   Processors (inline)    │
  └─────────────────────────┘

  ┌─────────────────────────┐
  │ Tenant Config            │
  │                          │
  │ • Retention policies     │
  │ • Rate limits            │
  │ • Cardinality budgets    │
  │ • Billing tier           │
  │ Queried by: Gateway,     │
  │   Rollup jobs,           │
  │   Governance, Query      │
  └─────────────────────────┘

  ┌─────────────────────────┐
  │ Rollup Scheduler         │
  │                          │
  │ • Cron-driven jobs       │
  │ • 10s→1min (hot→warm)   │
  │ • 1min→1hr (warm→cold)  │
  │ • Verify target written  │
  │   before deleting source │
  │ Reads: Hot/Warm stores   │
  │ Writes: Warm/Cold stores │
  └─────────────────────────┘
```

---

## Key Differences

| Aspect | Their Design | Ideal Design | Why It Matters |
|--------|--------------|--------------|----------------|
| Hot store | Postgres (row-oriented RDBMS) | Purpose-built TSDB with columnar storage, WAL, and compression on NVMe SSDs | Postgres tops out at ~50-100k inserts/sec; the hot store needs 5M/sec. Row-oriented reads are catastrophic for time-range aggregations. This single choice makes or breaks the system. |
| Tag querying | Not addressed | Inverted index (tag:value → series_id set) in Redis or custom store | Tags are the primary query dimension. Without an inverted index, every query requires a full scan of all series — impossible at this scale. |
| Cardinality governance | Absent from architecture despite being discussed in context | Inline check in stream processors using HyperLogLog, with configurable enforcement actions | A single customer tagging by request_id can create millions of series/hour, saturate indexing, and degrade the entire platform. This must be in the hot path, not an afterthought. |
| Rollup mechanism | "Probabilistic selection" of metric entries | Deterministic downsampling: compute {min, max, sum, count, avg} per time window per series | Probabilistic sampling loses data unpredictably. Deterministic rollups preserve all aggregation-relevant information at lower resolution — dashboards get exact answers, not approximations. |
| Write path buffering | Workers write directly to Postgres | In-memory write buffer with WAL → periodic flush to sorted columnar blocks | Buffering amortizes write cost, enables compression (delta + XOR encoding achieves ~1.4 bytes/point vs ~50 bytes raw), and turns random writes into sequential I/O. |
| Query routing | Single Postgres query, or two Postgres queries for historical | Query router that resolves tags → series IDs, determines tier(s) by time range, fans out to heterogeneous stores, merges results | Recent queries hit in-memory/SSD data (<200ms). Historical queries hit Parquet on S3 via Athena/Presto (<5s). These are fundamentally different execution engines behind one API. |
| Cost model | Not addressed | Explicit: ~$9k/mo for hot (NVMe), warm 60x cheaper per data point (downsample + EBS), cold another 4x cheaper (S3). Tiering is the entire cost strategy. | The problem explicitly requires cost math. At 2TB/day, the difference between keeping everything on NVMe vs. tiering to S3 is hundreds of thousands of dollars per year. |
| Multi-tenancy | Not addressed | Per-tenant rate limits at gateway, per-tenant Kafka partitioning, per-tenant cardinality budgets, per-tenant query quotas | Without tenant isolation, one noisy customer's burst can starve every other customer's ingestion and queries. This is a platform-killing failure mode. |

---

## Core Mental Model

**How they conceptualized the problem:** "Where does the data go?" — leading to a database-schema-first design (Postgres tables for metrics, blob store for archives, another Postgres for aggregates). The architecture was organized around storage locations.

**How they should have conceptualized it:** "How does data flow and transform?" — from raw data points arriving at 5M/sec, through inline processing (cardinality checks, tag indexing, series assignment), into a write-optimized hot store, through background rollup transformations, into progressively cheaper and coarser storage tiers, with a query layer that knows how to fan out across all tiers.

**Why this reframe leads to better architecture:** The "where does it go" framing makes you pick a database first and then try to make the workload fit. The "how does it flow" framing makes you design the pipeline stages first — what happens at each stage, what throughput is needed, what transformations occur — and then pick the storage engine that fits each stage's requirements. This naturally leads to: a write-optimized TSDB for hot (because the write rate demands it), columnar formats for warm (because rollups are aggregation-friendly), and Parquet on S3 for cold (because cost demands it). The storage choices *emerge from* the pipeline design rather than constraining it.

The other major reframe: **rollups are not data loss — they're format transformation.** The design treated tiering as "move data to cheaper place and maybe lose some." In reality, rollups *preserve all queryable information* at lower resolution. A 1-minute rollup with {min, max, sum, count} can answer any dashboard query exactly — avg, max, sum, count, and even approximate percentiles. Understanding this changes tiering from a cost-cutting compromise into a query-optimization strategy: coarser data is actually *faster* to query because there's less of it.
