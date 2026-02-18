# Context — Real-Time Fraud Decisioning Pipeline

## Context setting questions

Q: Is this asking us to design the actual scoring/ranking algorithm? I've never designed one. I'm not really sure how that would work. Are there specific things I can research that can quickly triage my understanding?

No. This exercise is about the system architecture around real-time fraud decisioning, not building a novel ML model from scratch.
Treat the model/rules engine as a black box that outputs a risk score, and focus your design on: ingestion, feature freshness, latency budgets, idempotency/dedup, fallback behavior, and policy/model rollout safety.

Quick triage research (high ROI):
1. **Feature pipeline basics**: "online features vs offline features in fraud systems" - understand how velocity features are precomputed and served fast.
2. **Decision path design**: "synchronous fraud decision service timeout and fallback policy" - learn fail-open/fail-closed tradeoffs. 
3. **Streaming correctness**: "event time windowing with late data and watermarking" - learn how out-of-order events affect risk features.
4. **Idempotency in streams**: "deduplication keys in at-least-once processing" - avoid duplicate events causing inconsistent decisions.

My research from the above tells me:
- We'll need both online feature service (in a redis-like cache) for the prediction component and large, queryable relational data for the offline training of that srevice
- We should be failing-open in the approval flow. Trade-off is risking $ vs customer satisfaction. If the fraud system has an issue, we could create a lot of unhappy customers by failing their transactions irrationally.
- The simplest late data strategy seems to be dead-letter. We'll drop all late notifications. The value is simplicity. This means additional risk of missing important out-of-window events.
- We should should do idempotency key checks in some cache (e.g redis) for stream events

Time elapsed: 41 mins.

## Guided Research (5 min)

Research these before starting your design. For each, write 1-2 sentences on what it is and when you'd use it. If time runs out, move on.

1. **At-least-once vs exactly-once** — In a fraud decision pipeline, when do at-least-once guarantees become risky, and what mechanisms keep duplicate events from causing inconsistent decisions?

at-least-once gaurantees are risky if you're double-processing. For example, a user might have a transaction just below a high-risk threshold for total spend per 5 transactions (e.g they bought a car). If we process that transaction twice, they are now above the high-risk threshold to where they will be flagged as a fraud candidate. Since we're using streaming, We can de-duplicate (and safeguard this) with idempotency keys checked against a redis store to verify we havent processed that event yet.

2. **Windowing** — How do tumbling, hopping, and sliding windows change fraud feature quality when behavioral events arrive late or out of order?

- tumbling is simplest to operate, but suffers at the edges (they absorb bursts at the edge into the full window aggregate, which loses information). tumbling requires only 1 update for late data (update that timeframe)
- hopping offers smoother edges than tumbling, which gives us more info on patterns at the edge, but updates to late data mean updating several time windows (fanning-out to the number of time windows that contain that time). In terms of fraud feature quality, these windows still do technically have edges, so they're not perfect.
- sliding windows I don't fully understand, but they appear to be the best fraud feature quality but the most operationally expensive.

Total time elasped: 65 mins.

## During Design
[Questions that arise while designing. Note which phase triggered them.]

Q: What is an OLTP system?

OLTP (Online Transaction Processing) systems are databases optimized for high-volume, low-latency transactional workloads — many small, fast reads/writes with strong consistency guarantees. Examples: PostgreSQL, MySQL, DynamoDB. Contrast with OLAP (Online Analytical Processing) systems optimized for complex analytical queries over large datasets.

In fraud decisioning, OLTP systems store transactional data (account details, recent transactions, device fingerprints) that must be queried quickly during authorization. The latency constraint (<120ms p99) means you can't afford slow analytical queries; you need indexed lookups that return in milliseconds. However, OLTP systems aren't ideal for computing aggregations like "transactions in last 5 minutes" — that's where pre-computed features or stream processors shine.

Q: Help me understand the options in the following design question:

> ### Decision 1: How should risk features be produced for sub-120ms p99 decision latency?

> - **A.** Synchronous feature reads at decision time from OLTP systems plus cache
> - **B.** Stream pre-computation into a materialized online feature store
> - **C.** Hybrid: pre-computed velocity features plus synchronous lookup for sparse attributes

I'm leaning towards option B. Here are my initial thoughts:
(a) would mean doing actual queries on non-aggregated data in real-time. Even with a cache, that feels expensive. It also seems like the cache would be ineffective because it would be either invalidated constantly or it would risk hiding high-risk behavior that wasn't in the cahe.
(b) we would have a worker doing constant "pre-computation" (lookups and aggregations) from a relational db into a redis-type store per-user. That worker set pulls jobs from a queue that is updated with a new message for each user when they do a transaction (so it should likely take effect before their next transaction). I'm not certian how the redis construction would look.
(c) I'm not sure what we mean by "velocity features" vs normal features (which could be deltas e.g number of transactions in past 5 minutes). I'm also not sure what we mean by a sparse attribute here, and what type of store we'd be doing a synchronous lookup in.

Help guide my thoughts.

**Guidance:**

**Option A (sync OLTP + cache):** Your intuition is right. At decision time you'd run queries like "SELECT COUNT(*), SUM(amount) FROM transactions WHERE account_id = ? AND created_at > NOW() - 5min" — even with indexes, that's multiple round-trips and aggregation work. Cache helps for repeat lookups of the same account, but (1) cache invalidation is hard — a new transaction changes the aggregates, so you'd need to invalidate on every write, and (2) cold cache on first transaction of a session means you still pay the OLTP cost. At 150K auth/sec, OLTP would be hammered. This option is viable only for low volume or when features are mostly simple key lookups, not aggregations.

**Option B (stream pre-computation):** The key correction: B is not "a worker triggered per user when they transact." It's a **continuous stream processor** (e.g. Flink, Kafka Streams, Spark Streaming) that consumes authorization and behavioral events as they arrive and maintains per-entity feature state. When a new auth event comes in, the processor has already updated features for that account/device/merchant from prior events. The feature store (Redis, DynamoDB, etc.) is written by the stream job, not by workers pulling from a relational DB. Flow: events → stream processor (windowing, aggregations) → feature store. At decision time you do a simple key lookup: `GET account:{id}:features` — no aggregation, no OLTP query. Redis layout: key = `account:{id}` or `device:{id}`, value = JSON/hash of pre-computed features (e.g. `tx_count_5m`, `spend_24h`, `distinct_merchants_1h`).

**Option C (hybrid):** Velocity features = aggregations over time windows (tx count in last 5 min, spend in last 24h, distinct merchants in last hour). These are expensive to compute on the fly — hence pre-compute. Sparse attributes = things that are simple key lookups with no aggregation: "is this device ID in the set of known devices for this account?", "what is the account's country of registration?". These change infrequently and don't need stream processing. You'd store them in OLTP or a cache and do a synchronous lookup at decision time. C says: pre-compute the expensive aggregations (velocity) in a feature store, and do cheap sync lookups for the sparse stuff. Fewer moving parts than full stream-everything, but you still need the stream pipeline for velocity.

**Which fits this problem?** Given 150K/sec, <120ms p99, and aggregations over behavioral events (constraint: events can arrive 2s late, out of order), B or C are the realistic options. B is "everything pre-computed"; C is "pre-compute the hard stuff, sync lookup the easy stuff." Your lean toward B is sound — the problem explicitly calls out streaming signals and aggregations. The main design question is how the stream processor handles late/out-of-order data (windowing + watermarks) and how the feature store is keyed and updated.

Q: We mention "maintains per-entity feature state". My key lack of understanding is how we can manage this without accessing the db. So, as an example, let's say we want a feature to be transactions in past 30 minutes, and let's say we're using a windowing strategy somewhere somehow of hopping 5-minute windows that begin every minute. Would this storage system mean we're always storing exactly 6 5-minute windows that allow us access to the full 30-minute timeframe, and each query for result would fetch those windows and compute a decision for this event?

**Answer:**

**Where the state lives (no DB):** The stream processor (e.g. Flink) has its own **state backend** — typically RocksDB or in-memory — keyed by `(entity_id, window_id)`. Events come from Kafka (or similar); the processor reads them, updates local state, and optionally writes results to the feature store. It never touches the OLTP DB. The DB is the source of raw transactions for the *event stream* (via CDC or an event producer), but the processor does not query it for aggregations.

**What gets stored:** You usually do *not* store 6 separate 5-minute windows and sum them at query time. For a feature like "transactions in past 30 minutes," you use a **single 30-minute window** (tumbling or sliding). The processor maintains one aggregate per entity (e.g. `account_123` → `{count: 7, sum_amount: 450}`) for that window. When it emits or flushes to the feature store, it writes the **final value** — e.g. `tx_count_30m: 7`. At decision time you do `GET account:123:features` and get `tx_count_30m: 7` directly. No fetching 6 windows, no summing at query time.

**If you did use hopping 5-minute windows:** You could store 6 window aggregates and sum at decision time, but that adds read amplification (6 keys) and a small compute step. The more common pattern is to define the feature as "30-minute count" and let the processor use one 30-minute window internally. The hopping/sliding choice affects how the processor computes that single value (e.g. sliding gives smoother updates), but the feature store still holds one number per feature.

**Summary:** State lives in the stream processor's state backend. The feature store holds **pre-aggregated values** (e.g. `tx_count_30m`, `spend_24h`), not raw window fragments. Decision path does a simple key lookup and gets the final feature vector — no aggregation at query time.

Q: From this discussion, I'm getting the feeling a stream processor is a critical component. I don't know how we could set a stream processor up to write patterns it recognizes to some redis path in realtime. I haven't used a stream processor at this point.

**Answer:**

The stream processor isn't doing "pattern recognition" (that's the model's job). It's doing **aggregations** — count, sum, distinct — over keyed windows. The output is numeric features, not detected patterns.

**Concrete flow:**

1. **Source:** Transaction and behavioral events are written to Kafka topics (e.g. `auth-events`, `device-signals`). Some service (payment API, device SDK) publishes these as they occur.

2. **Stream job:** A Flink or Kafka Streams job subscribes to those topics. The job defines:
   - A key (e.g. `account_id` or `device_id`)
   - A window (e.g. 30 minutes tumbling)
   - An aggregation (e.g. `COUNT(*)`, `SUM(amount)`)

3. **Processing:** Each event is assigned to a key and window. The processor updates its internal state (RocksDB). When a window closes (or on a configurable trigger), it emits a record like `{account_id: "123", tx_count_30m: 7, spend_30m: 450}`.

4. **Sink:** The job has a **Redis sink** — a connector that takes each emitted record and writes it to Redis. For example: `HSET account:123:features tx_count_30m 7 spend_30m 450`. Flink has Redis sinks (via connectors); Kafka Streams typically uses a custom producer or Kafka Connect to push to Redis. The sink runs inside the same job; no separate "writer" process.

**In practice:** You deploy a Flink cluster (or Kafka Streams app). The job runs 24/7, consuming from Kafka and writing to Redis. Latency from event ingestion to Redis update is typically seconds (depending on window emit policy and checkpointing). For sub-120ms decision latency, the feature store is already populated by the time an auth request arrives — the auth event that triggered the decision may not yet be reflected in the store, but prior events are, which is what you need for velocity features.

Time elapsed: 108 mins

Q: I'm still not fully confident how a redis online feature store would look and how it'd be queries.

**Answer:**

**Key structure:** One key per entity, with features as a hash. Entities are often account, device, and merchant — so you have multiple keys per decision.

```
account:acc_abc123:features     → hash of account-level features
device:dev_xyz789:features      → hash of device-level features
merchant:mer_456:features       → hash of merchant-level features (optional, if you have enough volume)
```

**Value structure (Redis hash):**

```
HGETALL account:acc_abc123:features
→ tx_count_5m      "3"
  tx_count_30m     "12"
  spend_24h        "450.00"
  distinct_merchants_1h "2"
  last_tx_ts       "1707890123"
```

Each field is a feature name; each value is the pre-computed aggregate (string in Redis). The stream processor writes via `HSET` or `HMSET`; it can overwrite the whole hash or update individual fields.

**Query at decision time:** The fraud service receives an auth request with `account_id`, `device_id`, `merchant_id`. It issues parallel lookups:

```
HGETALL account:acc_abc123:features
HGETALL device:dev_xyz789:features
HGETALL merchant:mer_456:features
```

Or use `HMGET` if you only need specific features. Pipeline these for lower latency. You get back a flat map of feature names → values, which you pass to the model/rules engine.

**TTL:** Set `EXPIRE account:acc_abc123:features 86400` (e.g. 24 hours) so inactive entities don't bloat memory. The stream processor can refresh TTL on each write.

**Summary:** Key = `{entity_type}:{entity_id}:features`, value = hash of feature name → value. Decision path = `HGETALL` (or `HMGET`) per entity, then feed the combined feature vector to the scorer.

Q: For the 2nd system design question, I don't really understand the question or the options:

> ### Decision 2: What decision execution path should the payment authorization call use?

> - **A.** Inline scoring in the payment API process with local rule/model runtime
> - **B.** Dedicated fraud decision service with bounded-time RPC and explicit fallback policy
> - **C.** Queue-first asynchronous scoring with later reconciliation callback

I was under the impression we just had some inference component we could lean on, and that it would be reasonable to call it synchronously. Is that not the case? Explain to me the question, and what the options are saying.

**Answer:**

**What the question is asking:** The payment authorization flow must return approve/review/decline to the issuer within a timeout (e.g. 200ms). The question is: *where* does the fraud scoring run, and *how* does the payment flow get that result? The options differ in coupling, failure isolation, and whether the auth response blocks on fraud at all.

**Option A — Inline scoring:** The fraud rules and model run *inside* the same process as the payment API. No RPC, no network hop. You fetch features from Redis, run the model, apply rules — all in-process. The payment flow blocks until scoring completes. Pros: lowest latency, no extra service. Cons: fraud logic is coupled to the payment service; model deployment means redeploying payment; if the model or feature fetch hangs, the whole payment process can stall; scaling is tied to payment traffic.

**Option B — Dedicated fraud service:** The payment API makes a synchronous RPC to a separate fraud service. Payment blocks until fraud returns a decision. The RPC has a bounded timeout (e.g. 80ms). If the fraud service times out, fails, or is unavailable, the payment API applies an explicit fallback (e.g. approve, or send to review). Pros: fraud is isolated; you can deploy models independently; payment can fail fast and fall back. Cons: extra network hop. This is what you're describing — "call inference synchronously" — and it's a valid choice.

**Option C — Queue-first async:** The payment API does *not* wait for fraud. It enqueues the auth request for scoring and returns immediately — typically with a default (e.g. approve or review). A worker scores it later and reconciles (e.g. via callback to issuer, chargeback, or manual review). Pros: payment latency is decoupled from fraud latency. Cons: the issuer may not accept a delayed decision; you're approving first and potentially reversing later. This only works if the business accepts "approve now, reconcile later" — many card flows require an immediate decision.

**Your impression:** "Call inference synchronously" is Option B. The question isn't "can we do that?" — we can. The question is whether B is better than A (inline) or C (async) for this system. Given the constraint (must return a deterministic fallback within 200ms if dependencies fail), the payment flow is synchronous — the issuer is waiting. So the real choice is A vs B. C is only viable if the business accepts deferred reconciliation.

**Why B often wins:** A dedicated fraud service gives you isolation (fraud outages don't take down payment), bounded timeouts (payment can enforce 80ms and fall back), and independent scaling. Inlining (A) is simpler but couples everything and makes fallback harder — if the payment process hangs on a feature fetch, you can't easily "fail open" without process-level timeouts.

Q: Question 3 asks the below:

> ### Decision 3: Where should short-lived risk state and deduplication state live?

> - **A.** In-memory key-value cluster with TTL-based risk state and idempotency keys
> - **B.** Stream processor managed state with changelog-backed recovery
> - **C.** Wide-column store keyed by account/device with time-bucketed rows

I think I partially understand the question, but I am definitely unsure on how to reason about the design. I'm assuming that features are being stored in an online feature store (e.g redis). Upon assessment, we could cache the per-feature risk assessment with a TTL in redis. We could also have cache our final event risk result for that event in redis (also, with an ETL) where the event id is the key and treated as an Idempotency key. Beyond that, I'm not sure how to approach the ask here. I don't know much about wide-column stores nor how they'd apply here. I don't know how a stream processor's state could be leveraged here.
My core gap in understanding is in what the stream processor can do and also how wide-column stores work.
Explain simple and briefly.

**Answer:**

**What the question is asking:** "Short-lived risk state" = intermediate data used to compute or serve risk decisions: window aggregates in flight, cached decisions for recent events, or similar. "Deduplication state" = have we already processed this event? (idempotency). The question is where to store these — not the final feature store (Redis for features), but the supporting state that prevents double-counting and enables correct aggregations.

**Option A (Redis + TTL):** Idempotency: `SET event_id:{id} "processed" EX 3600` — before processing, check; if exists, skip. Risk state: cache recent decisions or intermediate values with TTL. Simple, fast, but Redis is a separate system; if it's down, you lose dedup. TTL handles expiry.

**Option B (Stream processor state):** The stream processor (Flink, etc.) has built-in state — keyed by entity, holding window aggregates and "seen" event IDs. Changelog-backed = state is persisted to Kafka so the processor can recover after failure. Dedup: processor checks its state for `event_id` before applying; if seen, skip. Risk state = the window state (counts, sums) the processor already maintains. No separate Redis for dedup; it's inside the processor. The processor still writes *output* (features) to Redis; the question is where *internal* state lives.

**Option C (Wide-column store):** Cassandra/HBase. Row key = `account_id` or `(account_id, time_bucket)`. Columns = `event_id`, `count`, `amount`, etc. Time-bucketed = each row is a 5-min or 1-hour slice; you range-scan over rows to get "last 30 min." Good for durable, queryable history. Heavier than Redis; typically used when you need range scans over time and strong durability, not just fast key lookups.

**Stream processor in one sentence:** It consumes events, maintains keyed state (aggregates, seen IDs), and emits results — all without a separate DB for that state.

**Wide-column in one sentence:** Rows have many columns; you partition by key (e.g. account) and optionally by time bucket; good for time-range queries over large datasets.