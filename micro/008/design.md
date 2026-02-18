# Design: Real-Time Fraud Decisioning Pipeline

Total time budget: 40 minutes (research + design + buffer).
If stuck on any phase beyond its time box, write what you know, mark it incomplete, and move on.

## Phase 1: Framing (5 min)

**One-line summary:** We are designing a system that accepts events and classifies them based on a classifier. We are designing the data stores and processes the classifier will use as well.

**Core technical challenge:** Constructing a pipeline of events to both a "hot" and "cold" path for the classifier (online vs offline) features.

**Framing:** Which of these best fits this problem?
- [ ] A storage and retrieval problem
- [X] A streaming/pipeline problem
- [ ] A coordination/consensus problem
- [ ] A scheduling/state machine problem
- [ ] A distributed counting/aggregation problem

**The decision that matters most:**

## Phase 2: Sub-systems (5 min)

Name 3-5 sub-systems. One sentence each. Do NOT describe how they work — that's Phase 3.
If you're writing more than 5 lines, you're over-designing this phase.

- Online feature store - this is what the classifier invokes to make a decision
- Offline feature store - this is what will be eventually used for training of the classifier and other analytics
- Event ingestion engine - where events are processed, and routes to the proper places
- front api - the user hits this to get their classification

Time elapsed: 74 mins

## Phase 3: Key Decisions (15 min)

### Decision 1: How should risk features be produced for sub-120ms p99 decision latency?

- **A.** Synchronous feature reads at decision time from OLTP systems plus cache
- **B.** Stream pre-computation into a materialized online feature store
- **C.** Hybrid: pre-computed velocity features plus synchronous lookup for sparse attributes

**Your choice:** C
**Why:** Option A would be DB-heavy and subject to a thundering herd upon cache invalidation. Option B allows the creation of a low-latency storage solution (an online feature store) that the inference system needs. Option C adds a layer for the infrequently-changing features.
**How it works:**
1. An event comes in.
2. we store the event in a persistent db
3. via CDC or a producer, we send off a kafka message with the event 
4. there is a stream processor job waiting to pick up messages, one job per feature we care about (and potentially many workers per job).
5. when a stream processor job picks up its message, it'll write back our online feature store (e.g redis, key could be `account_id:123:features`)  with the aggregated feature info (e.g `{purchases_5m: 2, last_tx_ts: <time>}`)
6. when the inference component needs the data, it's quickly accessible via a lookup (e.g `account_id:123:features`).

Time elapsed: 111 mins

### Decision 2: What decision execution path should the payment authorization call use?

- **A.** Inline scoring in the payment API process with local rule/model runtime
- **B.** Dedicated fraud decision service with bounded-time RPC and explicit fallback policy
- **C.** Queue-first asynchronous scoring with later reconciliation callback

**Your choice:** B
**Why:** Inline scoring in the payment api process couples the scoring and the api. That means they deploy together and fail together. That's not good. The benefit of the option is lower latnecy. The dedicated fraud inference service is a good option because it de-couples the api and detection, but will be higher-latency than inline because we're making an RPC call. Option C is not deterministic within the 200ms, so we can't accept that option.
**How it works:**
We will make an RPC call with the transaction event info to the fraud inference service, store the result in a relational DB for analytics, and return the result.
It seems like the better path is to fail-open on this fraud detection service (if it's down or the call times out after some reasonable TTL) and allow all transactions while it's down. We can have a reconiliation mechanism (a one-off worker job) to back-assess transactions that occurred while the fraud detection service was down.

= 129 mins

### Decision 3: Where should short-lived risk state and deduplication state live?

- **A.** In-memory key-value cluster with TTL-based risk state and idempotency keys
- **B.** Stream processor managed state with changelog-backed recovery
- **C.** Wide-column store keyed by account/device with time-bucketed rows

**Your choice:** Option A
**Why:** Option B feels to be assuming features from the stream processor that we cannot assume. I dont understand the wide column store enough to weigh it as an option. Option A leverages a simple, low-latency path via in-memory storage.
**How it works:** We use the same redis cluster as is hosting the feature store. Before running the fraud check, we check if the event has already been processed (de-dup) by searching the event id in redis (e.g `fraud_result:event_id`) and, if so, we return the result. Otherwise, after the fraud check, we store the result in that redis key. That key (the event id) can be re-used as an idempotency check as well.

= 163 mins

## Phase 4: Stress Test (5 min)

Pick the 3-5 most critical NFRs and failure modes. You cannot cover all of them — prioritize.

| Concern | Type | How addressed | Confidence |
|---------|------|---------------|------------|
| | NFR / Failure | | high / medium / low |

- 150k events / second. We can definitely handly that safely with horizontally-scaled api servers and a set of sharded DBs, perhaps sharded by the hash of the event id. Then a set of CDC listeners per DB. All-in-all, I feel the request path itself is simple enough as it just makes one DB transaction. At that point, it's just about horizontal scaling.
- 99.99% availability. With DB, Redis and Kafka clusters that each support horizontal scaling and replication among themselves, this infrastructure should be highly available. Medium confidence.
- Catch at least 92% of confirmed fraud. I have no clue. 

---

Time elapsed: 170 mins.

---

## Review

**Score: 4.5/10**

| Criterion | Score | Notes |
|-----------|-------|-------|
| Problem Framing | 5/10 | Correct category, but one-line summary is generic, core challenge is partially wrong, and "decision that matters most" is blank |
| Key Decisions | 6/10 | All three choices are defensible; D2 mechanics are solid; D1 mechanics miss the hybrid component; D3 reasoning is thin |
| Tradeoff Reasoning | 5/10 | D2 reasoning is the strongest in the design; D1 rejects A correctly but never explains why C beats B; D3 explicitly admits ignorance |
| Stress Testing | 3/10 | Three items in prose below an empty table; 92% catch rate is abandoned; no failure mode analysis |
| Communication | 4/10 | "Decision that matters most" blank, stress test table empty, Phase 2 descriptions are thin, 4x time overrun |

### Framing Assessment

**Their framing:** A streaming/pipeline problem — constructing a pipeline of events to a "hot" and "cold" path for the classifier.
**Correct framing:** A streaming/pipeline problem — maintaining reliable, low-latency feature freshness under out-of-order event arrival while enforcing a hard latency budget across a synchronous decision path.
**Match:** Partially correct.

The category is right — this is a streaming/pipeline problem. But the identified core challenge misses the mark. "Hot and cold path for the classifier" is an implementation detail, not the fundamental tension. The actual challenge is: **behavioral signals arrive 2 seconds late and out of order, yet the decision must complete within 120ms p99**. You can't wait for events to settle; you have to decide on stale features. That tension — late data vs bounded decision latency — is what forces every architectural choice in this design. Calling it "hot/cold feature pipeline construction" frames it as a data routing problem rather than a correctness-under-latency problem.

The "decision that matters most" being left blank is also a real miss. In an interview, this is the moment you signal what you understand about the problem. The answer here is: **how do you produce risk features fast enough that a synchronous decision path can hit <120ms p99 when upstream signals may not have arrived yet?**

### Phase-by-Phase Feedback

**Phase 1 (Framing):** Correct category, partially correct core challenge, blank critical field. The one-line summary — "a system that accepts events and classifies them based on a classifier" — is product-level, not technical. Compare: "A streaming feature pipeline that pre-computes risk aggregates from out-of-order events to serve sub-120ms synchronous fraud decisions." The framing didn't hurt Phase 3 decisions, but that's largely because the decision scaffolds constrained the choices — not because the framing was load-bearing.

**Phase 2 (Sub-systems):** Four sub-systems named, but three critical ones are missing:
- **No fraud decision service** — the component that makes the actual approve/review/decline call is not named as a sub-system, yet it's the entire subject of Decision 2.
- **No stream processor** — named in the mechanics of Decision 1 but not as a sub-system.
- **No deduplication / idempotency layer** — the entire subject of Decision 3.
- **No policy/rules store** — FR3 requires dynamic policy updates without stopping traffic; that's a sub-system.

The descriptions are also thin: "this is what the classifier invokes to make a decision" describes a usage, not a responsibility. What does the online feature store own?

**Phase 3 (Key Decisions):**

*Decision 1 (C — hybrid):* Correct choice. The rejection of A is sound ("DB-heavy, thundering herd on cache invalidation"). But the rationale for C over B is absent — you described what B does without saying why C beats it. The distinction that makes C better than B: sparse attributes like account country, known device set membership, or account age don't change with every transaction. There's no need to stream-compute these; a direct OLTP or Redis lookup is cheaper and simpler. B would stream-compute everything, including fields that don't benefit from it. The mechanics have one significant error: "one job per feature we care about." Stream processors are keyed by entity, not by feature. You run one job (or a small set of jobs per entity type) that computes all features for that entity in a single pass. One-job-per-feature means hundreds of parallel jobs all writing to the same Redis key, fighting over update ordering and wasting resources.

*Decision 2 (B — dedicated fraud service):* Best reasoning in the design. Correctly rejects A (coupling and joint failure), correctly rejects C (non-deterministic within 200ms constraint), references the specific constraint. The fail-open decision is right and the reconciliation mechanism is a concrete addition. What's missing: the RPC must itself have a bounded timeout. The problem says 200ms hard deadline. If the fraud service takes 150ms and the feature lookups take 50ms, you've already missed it. The design should name the budget allocation (e.g., 80ms for feature fetch + inference, leaving margin for payment API overhead) and explicitly state what "explicit fallback policy" means — which you do cover ("approve, then reconcile"), but it should be in the mechanics, not just implied.

*Decision 3 (A — Redis + TTL):* Correct choice. The rejection of B is weak ("assuming features from the stream processor that we cannot assume" is vague and inaccurate — B is fully viable; you just chose not to add that complexity). The rejection of C should be: wide-column is for durable, time-range-queryable history — it has higher read/write latency than Redis and is over-engineered for sub-second idempotency checks. The mechanics are sound: Redis key per event_id, check before scoring, write after. The one gap: what's the TTL on the dedup key? Without a TTL, you'd eventually fill memory. And what happens if Redis is down during the check — do you skip dedup (risk double-processing) or fail the scoring (risk dropping events)?

**Phase 4 (Stress Test):** The table is empty; three concerns are described in prose below it. The prose items are:
- 150K/sec → horizontal scaling and sharded DBs. Correct directional answer, but the interesting bottleneck isn't the API — it's whether Kafka and the stream processor can keep pace at peak (2x burst = 300K/sec). The lag accumulation during a burst is the real failure mode.
- 99.99% availability → horizontal scaling and replication. The two-region requirement in NFR3 is ignored entirely. Active-active across two regions with Redis and Kafka is a non-trivial problem.
- 92% catch rate → "I have no clue." This gets no credit, but the honest answer is also that this is mostly an ML/model problem, not a systems architecture problem. The systems architecture contribution is: feature freshness (stale features hurt recall), late event handling (missed events = missed signals), and avoiding dedup false positives.

Missing critical failure modes:
- Redis cluster failure: both the feature store and dedup store are gone simultaneously.
- Kafka consumer lag under 2x burst: stream processor falls behind, features go stale, recall degrades.
- Out-of-order events landing after the decision: a fraud signal for an event arrives 2 seconds late — do you re-score? What's the policy?
- Stream processor job restart during a window boundary: partial aggregates can produce incorrect features.

### Research Assessment

**At-least-once vs exactly-once:** Well answered. Identified the risk (double-counting pushes accounts over thresholds) and the mechanism (Redis idempotency keys). Translated directly into Decision 3 mechanics. Good.

**Windowing:** Correctly characterized all three types (tumbling/hopping/sliding tradeoffs). However, the research never translated into Phase 3. Decision 1's mechanics don't specify a windowing strategy, don't mention watermarks, and don't address how out-of-order events (the 2-second constraint) are handled. The research answered the question but was left on the table — the design behaves as if all events arrive in order.

**Late data (from triage research):** "Dead-letter late events" was noted as the simplest strategy. This is a real tradeoff that deserved mention in Phase 4 — dropping 2-second-late events means missing fraud signals for a known class of events. It also contradicts the research finding on windowing (why research windowing strategies if you're dead-lettering late events?).

**FR3 (dynamic policy updates) and FR4 (decision/outcome events):** Neither addressed anywhere in the design. FR3 is a system design question with real answers (hot-reloadable config, feature flags, model version routing). FR4 is a named output requirement.

---

## Ideal System Architecture

```
═══ FEATURE PIPELINE (async) ═══

  Auth Events ───▶ Kafka ──▶ Stream Processor (Flink)
  Device Signals ──▶       (dedup by event_id in      │
  Merchant Signals ──▶      processor state;           │
                            window aggregations per    │
                            account / device /         │
                            merchant entity;           │
                            watermark for 2s late      │
                            event tolerance)           │
                                                       ▼
                                             ┌──────────────────┐
                                             │  Online Feature  │
                                             │  Store (Redis)   │
                                             │  account:{id}:   │
                                             │  features        │
                                             │  device:{id}:    │
                                             │  features        │
                                             └──────────────────┘

  Stream Processor also emits to:
  ┌──────────────────┐
  │  Offline Store   │  Queried by: model training,
  │  (data warehouse)│  analyst investigation,
  └──────────────────┘  post-transaction learning (FR4)

═══ DECISION PATH (synchronous, per auth request) ═══

  Issuer ──▶ Payment API ──RPC (80ms timeout)──▶ Fraud Decision Service
                │                                       │
                │ fallback: approve if timeout/down     │
                ◀──────────────────────────────────────-┘
                                                        │
                                          ┌─────────────┼─────────────────┐
                                          │             │                 │
                                          ▼             ▼                 ▼
                                   Redis dedup    Redis feature      OLTP (sparse
                                   check          store lookup       attr lookup:
                                   (event_id      (account,          account country,
                                    TTL: 1h)       device,           known devices)
                                                   merchant)
                                          │
                                          ▼
                                   Rules Engine + Model
                                   (policy loaded from
                                    Policy Store; hot
                                    reloadable)
                                          │
                                          ▼
                                   Decision (approve / review / decline)
                                          │
                          ┌───────────────┼───────────────────┐
                          ▼               ▼                   ▼
                   Redis dedup      Decision DB           Outcome event
                   (write result    (analytics,           ──▶ Kafka ──▶
                    for idempotency) audit log)           Analyst store

═══ SUPPORTING SERVICES ═══

  ┌──────────────────┐
  │  Policy Store    │  Holds rule versions and model endpoints;
  │  (config DB /    │  Fraud Decision Service subscribes or polls;
  │   feature flags) │  enables FR3 (hot rule/model swaps) without restarts
  └──────────────────┘

  ┌──────────────────┐
  │  Redis Cluster   │  Roles: feature store (stream processor writes),
  │  (shared)        │  dedup + idempotency (decision service reads/writes),
  │                  │  TTL-bounded (features: 24h, dedup keys: 1h)
  └──────────────────┘
```

---

## Decision Analysis

### Decision 1: Feature production for <120ms p99

**Correct option: C (hybrid)** — though B is also fully defensible.

**Why not A:** At 150K/sec, synchronous OLTP aggregations ("SELECT COUNT(*) WHERE account_id=? AND ts > NOW()-5min") are untenable. Even with caching, cache invalidation on every transaction makes hit rates low for active accounts, and cold misses pay full aggregation cost. Thundering herd on cache invalidation is a real concern at this scale.

**Why C over B:** B pre-computes everything via stream processor. C adds a distinction: sparse attributes (account country, account age, whether device is in known-device set) are infrequently-changing, non-aggregate fields. They don't need a stream processor — a direct Redis or OLTP lookup is cheaper and simpler. Streaming them adds job complexity without improving freshness. C reserves the stream processor for what it's actually good at: time-windowed aggregations (velocity, spend totals, distinct merchant counts).

**Correct mechanics:** Stream processor (Flink) consumes auth and behavioral events from Kafka, keyed by entity (account, device, merchant). Maintains window state with a 2-second watermark to tolerate late events. Emits aggregated features to Redis on window close (or on configurable trigger). Sparse attribute lookups (account metadata) bypass the stream processor and hit OLTP or a separate Redis key directly at decision time. All lookups are pipelined in the fraud decision service.

**Effect on Decision 2:** The feature store being Redis means the RPC from payment API to fraud service needs a tight timeout (≤80ms) to leave margin for Redis lookups plus inference.

### Decision 2: Decision execution path

**Correct option: B (dedicated fraud decision service with bounded RPC and explicit fallback).**

**Why not A:** Inlining couples deployment and failure domains. A model artifact deployment or scoring library bug can take down the payment API. At 150K/sec, the payment API is on the critical path for revenue; the fraud service should not be on the same failure domain.

**Why not C:** The issuer timeout window requires a synchronous decision. "Queue first, reconcile later" is an async pattern that only works if the business accepts approving first and reversing later (chargeback path). The problem states the system must return a deterministic fallback within 200ms — this is a synchronous contract.

**Correct mechanics:** Payment API sets an RPC timeout of ~80ms to the fraud service (leaving budget for network overhead and the payment API's own processing). Fraud service fetches features from Redis (pipelined, ~5ms), runs rules + model (~20-30ms), returns decision. If the RPC times out or the fraud service is unreachable, payment API applies the explicit fallback: fail-open (approve) and emit the event to a reconciliation queue for back-assessment. Policy store holds the fallback rule so it can be changed without a deploy.

**Effect on Decision 3:** The fraud service needs a dedup check before scoring, and must cache decision results for idempotent retries — this is what Decision 3 is solving.

### Decision 3: Short-lived risk state and deduplication

**Correct option: A (Redis + TTL).**

**Why not B:** Stream processor managed state works well for aggregation state, but using it as the deduplication authority for the fraud decision service tightly couples the decision service to the streaming job's availability and recovery cycle. When the stream processor restarts (checkpoint recovery), the dedup state gap could allow duplicates through during the recovery window. Keeping dedup in Redis isolates the concern.

**Why not C:** Wide-column stores (Cassandra, HBase) are optimized for durable, queryable time-range history. They have higher write and read latency than Redis (single-digit ms vs sub-ms), and their data model (time-bucketed rows with range scans) is over-engineered for what dedup needs: a point lookup on an event ID to see if it's been processed. TTL management in Cassandra is also less predictable than Redis `EXPIRE`.

**Correct mechanics:** Before scoring, fraud service calls `GET dedup:event:{id}`. If exists: return cached result. If not: score the event, then `SET dedup:event:{id} {result} EX 3600` (TTL = window large enough to outlast any retry window, e.g. 1 hour). Feature store keys live in the same Redis cluster under different key prefixes (`account:{id}:features`). Both use TTL — features expire after 24h (inactive entities), dedup keys after 1h. If Redis is unreachable, decision service falls back: skip dedup check and score anyway (accept potential double-scoring as the lesser failure mode vs dropping the request).

---

## Key Differences

| Aspect | Their Design | Ideal Design | Why It Matters |
|--------|--------------|--------------|----------------|
| Core technical challenge | "Hot/cold path construction" | Feature freshness under out-of-order arrival within a hard latency budget | Wrong framing frames it as a routing problem rather than a correctness-under-latency problem |
| "Decision that matters most" | Left blank | Feature production strategy — pre-computing the right signals fast enough to hit <120ms p99 | This is the most important field in Phase 1; leaving it blank signals no clear mental model |
| Stream processor architecture | One job per feature | One job (or one job per entity type) computing all features for that entity | One-job-per-feature creates write contention on Redis and hundreds of redundant jobs |
| Hybrid element in Decision 1 | Described B's mechanics without the sparse-lookup component of C | Velocity features via stream processor; sparse/static attributes via direct OLTP/Redis lookup | Without the sparse-lookup component, the choice of C over B is unexplained and the design is just B |
| Windowing and late events | Not addressed despite researching windowing | Watermark of 2s in stream processor to tolerate late events; define late-arrival policy explicitly | Constraint 1 explicitly says events arrive up to 2s late — ignoring this means features can be incorrect |
| RPC timeout budget | Not defined | Explicit budget allocation (e.g. 80ms for fraud service) within 200ms hard deadline | Without a named budget, the design can't be evaluated for feasibility against the p99 requirement |
| Dynamic policy updates (FR3) | Not addressed | Policy store with hot-reloadable rules/model versions; fraud service subscribes or polls | FR3 is a named functional requirement; omitting it means the design fails a stated requirement |
| Decision/outcome events (FR4) | Not addressed | Decision results emitted to Kafka → analyst store / offline training pipeline | FR4 is a named functional requirement; also closes the feedback loop for model improvement |
| Stress test table | Empty | Filled with 3-5 concerns, types, mitigations, and confidence | Concerns in prose below an empty table is structurally incoherent |
| Two-region availability (NFR3) | Not addressed | Active-active or active-passive across two regions with Redis replication and Kafka geo-replication | 99.99% across two active regions is a specific, hard constraint that shapes the entire deployment model |

---

## Core Mental Model

**How they conceptualized the problem:** As a data routing problem — get events to the right stores (hot/cold), then the inference service reads from them. The mental model is "build the pipelines and the classifier will work."

**How they should have conceptualized it:** As a **correctness-under-latency problem** — specifically, how do you make a high-confidence decision in 120ms when the signals you need may not have arrived yet? The core tension is: fraud detection quality improves with more complete behavioral context, but the latency budget shrinks the window for that context. Every architectural decision follows from managing that tension: pre-compute features so the decision path doesn't wait, use watermarks so the stream processor tolerates late events without blocking, use a dedup layer so retries don't corrupt feature state, use fail-open so the latency budget is never violated.

**Why this reframe leads to better decisions:** Thinking "routing" leads you to focus on where data goes. Thinking "correctness under latency" leads you to ask: what state must be fresh at decision time? What happens if it isn't? What's the fallback? How do retries affect correctness? Those questions directly map to all three decision scaffolds and to the missing elements (watermarks, policy store, outcome events, RPC timeout budgeting).
