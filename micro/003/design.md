# Design: Multi-Tenant Search-as-a-Service

Total time budget: 40 minutes (research + design + buffer).
If stuck on any phase beyond its time box, write what you know, mark it incomplete, and move on.

## Phase 1: Framing (5 min)

**One-line summary:** This is a multi-tenant service enabling customers to do full-text searches on their documents that supports uploads and fast queries at huge scale.

**Core technical challenge:** Routing a shard to its persistent location where it can then be queried (fast routing, fast querying)

**Framing:** Which of these best fits this problem?
- [ ] A storage and retrieval problem
- [X] A streaming/pipeline problem
- [ ] A coordination/consensus problem
- [ ] A scheduling/state machine problem
- [ ] A distributed counting/aggregation problem

**The decision that matters most:** Is how the uploaded documents reach their home in a shard.

Time elapsed: 51 mins.

## Phase 2: Sub-systems (5 min)

Name 3-5 sub-systems. One sentence each. Do NOT describe how they work — that's Phase 3.
If you're writing more than 5 lines, you're over-designing this phase.

1. Document upload system => Users upload their documents here
2. Routing system / Gateway => User queries come here, this routes them to the aggregation service
3. query aggregation service => Maps a query to the indices the should be hitting. Then, maps from thoe indices to shards, and from those shards to specific nodes. Then, queries those indices from the nodes and aggregates the findings
4. shards => these are physical stores of json documents (e.g document DBs).

Time elapsed: 58 mins.

## Phase 3: Key Decisions (15 min)

### Decision 1: Index and shard isolation strategy

- **A.** Index-per-tenant — each tenant gets a dedicated Elasticsearch index with its own shards and mappings
- **B.** Shared indices with routing — tenants share large indices, documents routed to tenant-specific shards via custom `_routing`
- **C.** Tiered isolation — small tenants grouped into shared indices, large tenants assigned dedicated indices based on document volume thresholds

**Your choice:** C
**Why:** This balances the isolation of dedicated indices (a large client might have big usage spikes and big upload volume) with the cost-awareness of shared indices with routing.
**How it works:**
The query aggregation service would go through a series of lookups to transform from a customer query to a set of indices and their node information. This would likely be some kind of inverted index that maps different aspects of their query to indices and from indices to nodes.

It's not fully clear to me what this lookup process would look like.

The indices (JSON-like structures) could live in a document DB and be cached in a NoSQL cache (e.g Redis).

The index location could be decided on initialization based on a "Tier" selected by the client to determine if they will get shared or dedicated resources. We don't have a specific requirement on spin-up latency, but if we did, we would need extra dedicated resources sitting idle at all times to meet the scenario of rapid spin-up for dedicated-tier clients.

Time elapsed: 70 mins.

### Decision 2: Indexing pipeline architecture

- **A.** Direct Elasticsearch writes — application tier sends bulk index requests directly to the cluster
- **B.** Queue-buffered pipeline — documents flow through a message queue with consumer workers that batch-write to Elasticsearch
- **C.** Write-ahead to primary store with async sync — documents persist in a durable primary store first, then sync to Elasticsearch asynchronously via change data capture

**Your choice:** B
**Why:** De-couples the incoming request (indexing request) from the internal work to change the document
**How it works:**
The user sends us their json object. We put a message on a messaging system (e.g kafka) with the new document content. we return an OK to the user. We have worker processes who then pick up the job to process the document, apply changes depending on the user's configuration (maybe a condition depending on a config setting the user has set up that we've persisted in a DB), gets the index location using the same lookup logic used in the query aggregation service, and updates the relevant document.

Total time elapsed: 82 mins.

### Decision 3: Reindexing strategy for schema evolution

- **A.** Alias swap with online reindex — create a new index with updated mappings, reindex documents from old to new, atomically swap the alias
- **B.** Dual-write with catch-up — start writing to both old and new index, backfill existing documents into the new index, cut over when caught up
- **C.** Versioned indices with query fan-out — create a new index version for new writes, query across both old and new indices, gradually migrate old documents in the background

**Your choice:** Option C
**Why:** Continuity in the upload process (no paused uploads), And I'd prioritize an interim double-read > double-write.
**How it works:**
Described in `context.md` document by the AI question-answerer. I did not design this.

Time elapsed: 99 minutes.

## Phase 4: Stress Test (5 min)

Pick the 3-5 most critical NFRs and failure modes. You cannot cover all of them — prioritize.

| Concern | Type | How addressed | Confidence |
|---------|------|---------------|------------|
| | NFR / Failure | | high / medium / low |

1. Network partition between the routing component (to map to indexes/shards) and other components => We would be unable to process queries. We could address this with some kind of failover to a rouding component in another region that reaches the same persistence.
2. Huge spike in document queries for a shared tier user => We must have per-client rate limiting at the gateway and we can also respect backpressure with query worker pools in the document nodes that reject after some concurrency limit + the queue length and additional app logic that queues failed workloads for when the nodes are available again. High confidence this is handled.
3. Low-latency queries. I'm not sure if this is handled. I think this relates linearly to the efficacy of the indexes, fan-out, and performance handling at the node level (limiting concurrent threads via worker pools, limiting search space via efficient sharding). I have medium confidence those considerations are covered in this design.

---

Time elapsed: 126 minutes.

---

## Review

**Score: 4.5/10**

| Criterion | Score | Notes |
|-----------|-------|-------|
| Problem Framing | 4/10 | Wrong framing (streaming/pipeline), wrong core challenge (routing), wrong "most important decision" |
| Key Decisions | 5/10 | Decision 1 and 2 correct choices; Decision 3 not actually designed; mechanics confused throughout |
| Tradeoff Reasoning | 4/10 | Justifications generic, don't reference specific constraints or NFRs, rejected options not explained |
| Stress Testing | 4/10 | One relevant concern (noisy neighbor), two generic/vague; missed the most system-specific risks |
| Communication | 5/10 | Honest about gaps, structure mostly followed, but time 3x over budget and Decision 3 explicitly punted |

### Framing Assessment

**Their framing:** Streaming/pipeline problem — "Routing a shard to its persistent location where it can then be queried (fast routing, fast querying)." Most important decision: the indexing pipeline (how documents reach their shard).

**Correct framing:** Scheduling/state machine problem — managing the placement, lifecycle, and resource allocation of 8,000 tenants with 1000:1 size variance on shared, stateful, shard-constrained infrastructure, where the units of allocation (shards, nodes) are scarce and each operational action (reindex, bulk import, analyzer change) is a multi-step state transition that can destabilize the cluster.

**Match:** Wrong.

The "streaming/pipeline" framing makes you think the hard part is getting documents into ES efficiently. That's a solved problem — Kafka + ES `_bulk` API is well-understood infrastructure. The actual hard parts are:

1. **Shard strategy under constraints:** You have 8,000 tenants with sizes ranging 10,000:1. The shard is the unit of resource allocation, and there's a hard ceiling (~1,000 per node). How you pack tenants onto shards determines whether the cluster is viable at all. This is a bin-packing/scheduling problem.
2. **Noisy neighbor isolation:** One tenant's 8K docs/sec bulk import or expensive aggregation query must not degrade others by more than 20%. ES has no native tenant awareness. Isolation must be built in your orchestration layer above ES.
3. **Schema evolution as a state machine:** Reindexing a 500M-document tenant takes hours. During that time, the system must manage concurrent states (old index receiving queries, new index being built, catch-up phase, atomic cutover, cleanup). This is a multi-step state machine, not a pipeline.

The wrong framing led directly to misidentifying the most important decision. The pipeline (Decision 2) matters, but it's downstream of the shard strategy (Decision 1). If you pack tenants onto shards wrong, it doesn't matter how elegantly you deliver documents — the cluster will either exceed shard limits or create hot shards that violate the noisy neighbor NFR. The shard isolation strategy is the load-bearing decision.

### Phase-by-Phase Feedback

**Phase 1 (Framing):** The one-line summary restates the problem rather than identifying what makes it hard. "Core technical challenge" — "Routing a shard to its persistent location" — inverts the relationship. Shards don't get routed; documents get routed to shards, and queries get routed to indices. But even corrected, the challenge isn't routing — it's *what shards exist, how many, and which tenants are on which ones.* "The decision that matters most" — the pipeline — is Decision 2, not Decision 1. In a real design review, misidentifying the load-bearing decision means you'll spend your design energy on the wrong thing, which is exactly what happened (Decision 1 mechanics were confused; Decision 2 got the clearest treatment).

**Phase 2 (Sub-systems):** Four sub-systems named, discipline broken on #3 (full mechanics described instead of one sentence). More critically:

- Sub-system #3 describes ES's own coordinating-node behavior as though it's a platform-level service you'd build. ES already does query fan-out, shard-level execution, and result merging internally. Your platform's query layer is a thin router: resolve tenant → index alias, add tenant filter if shared index, enforce rate limit, forward to ES. Conflating your system's responsibilities with ES's built-in behavior suggests unclear mental model of the boundary between "what we build" and "what ES does."
- Sub-system #4 ("shards as physical stores of json documents, e.g. document DBs") — shards are ES/Lucene primitives, not a sub-system you design. They're Lucene indices containing inverted indexes, doc values, and stored fields. Calling them "document DBs" mischaracterizes the technology.
- **Missing:** Tenant metadata service (maps tenants → indices, stores configs, manages aliases), per-tenant rate limiter / quota service (the noisy neighbor NFR demands this as a first-class sub-system), reindex orchestrator (schema evolution is a core FR — where does it live?), cluster health monitor (tracks shard counts against ceilings, triggers tenant promotions).

**Phase 3 (Key Decisions):**

*Decision 1 (Tiered isolation):* Correct choice. But the justification — "balances the isolation of dedicated indices with the cost-awareness of shared indices" — is a generic restatement of what "tiered" means. It doesn't say *why* A and B are wrong for *this* problem's specific constraints. The mechanics are confused: "some kind of inverted index that maps different aspects of their query to indices" conflates ES's inverted index (text search) with the platform's metadata lookup (tenant → index routing). "The indices (JSON-like structures) could live in a document DB and be cached in a NoSQL cache" mischaracterizes ES indices — they're cluster-level constructs containing shards, not JSON structures you store in another database. The "Tier selected by the client" idea adds a billing model that doesn't exist in the requirements and misplaces the tiering decision — the platform should tier automatically based on document volume thresholds, not customer self-selection.

*Decision 2 (Queue-buffered pipeline):* Correct choice. The justification — "de-couples the incoming request from the internal work" — is correct but generic. The *specific* reasons B is right for this problem: (1) 40K docs/sec with bursts to 8K/tenant requires buffering that A can't absorb without coupling API availability to ES cluster health; (2) the queue enables per-tenant flow control — you can throttle consumption rate per tenant to prevent a bulk import from saturating ES; (3) the 5-second searchability SLA is achievable (Kafka → batched bulk writes to ES completes in 1-2 seconds under normal load), ruling out C which adds CDC latency. The mechanics description is the strongest of the three decisions, though the mention of "apply changes depending on a config setting" is vague hand-waving.

*Decision 3 (Versioned indices with query fan-out):* Defensible choice, but the reasoning doesn't address the primary cost: **inconsistent search relevance during migration.** If a tenant switches from English stemming to multilingual stemming, old-index documents rank differently from new-index documents. For a *search platform*, relevance consistency is a core product promise. Option A (alias swap), combined with the Kafka queue from Decision 2 (replay messages arriving during reindex for catch-up), gives you a clean atomic cutover with consistent relevance. The mechanics were explicitly not designed — "Described in context.md document by the AI question-answerer. I did not design this." This is the most honest statement in the design, but for grading purposes, an undesigned decision is an incomplete decision.

**Phase 4 (Stress Test):**

Concern 1 (network partition to routing component) — generic failure mode applicable to any distributed system. The mitigation ("failover to a routing component in another region") is reasonable but not specific to this system. A search platform's most critical partition scenario is between the ingestion workers and the ES cluster during a bulk import — messages pile up in Kafka, and when the partition heals you get a thundering herd of bulk writes.

Concern 2 (spike for shared-tier user) — the most relevant concern. The mitigation mentions per-client rate limiting at the gateway and backpressure with worker pools, which are directionally correct. But "query worker pools in the document nodes" conflates the application layer with ES's internal thread pools. The concrete mitigation is: per-tenant rate limiting *above* ES (in your gateway and ingestion workers), combined with ES circuit breakers as a safety net.

Concern 3 (low-latency queries) — important but vaguely addressed. "Relates linearly to the efficacy of the indexes, fan-out, and performance handling at the node level" is a list of factors, not an analysis. The specific concern: a large tenant with 50 shards fires a complex aggregation that consumes all search thread pool slots on the nodes hosting those shards, starving other tenants' simple queries on the same nodes. The mitigation is shard-aware node allocation (large tenants on dedicated nodes) and per-tenant query timeout/complexity limits.

**Missing critical concerns:**
- **Shard count management:** With 8,000 tenants in a tiered model, what's the total shard count? If 7,760 small tenants are grouped ~200 per shared index, that's ~39 shared indices × ~5 shards = ~195 shards. If 240 large tenants average 20 shards each, that's ~4,800 shards. With replicas, ~10,000 total. At 1,000 shards/node, you need 10 data nodes just for shard overhead. This math needed doing.
- **Reindex impact on cluster stability:** Reindexing a 500M-document tenant means reading all docs from old shards and writing to new shards. This saturates disk I/O and network on the hosting nodes. Without throttling the reindex rate, it degrades every other tenant on those nodes.
- **Schema evolution for shared-index tenants:** If a small tenant in a shared index changes their analyzer, you can't reindex the entire shared index (it contains other tenants' data). You'd need to extract that tenant's documents, reindex into a temporary or new location, and route them back. This interaction between Decision 1 and Decision 3 wasn't considered.

### Research Assessment

**Guided research prompt 1 (index-per-tenant vs shared indices):** Good. The three dimensions (shard count, fan-out, hot shard risk) were correctly identified with directionally correct tradeoffs. Minor inaccuracy: "fan-out is low (can always route to just one shard)" — a large tenant with index-per-tenant would still have multiple shards (a 500M-doc tenant at 10-50 GB/shard needs 10-50 shards), so fan-out is proportional to tenant size, not always one. This research DID translate into Decision 1 — the tiered choice reflects understanding that both extremes fail. This is the strongest research-to-decision connection in the exercise.

**Guided research prompt 2 (thread pools, circuit breakers, request cancellation):** Surface-level. Each mechanism described as "protect nodes" without explaining *how* — thread pools don't protect nodes in general; they bound concurrency for specific operation types (search, index, bulk), and when the queue fills, requests are rejected. The key insight missed: **ES's built-in mechanisms protect the cluster from total overload but do NOT provide per-tenant fairness.** Thread pools are shared across all tenants. A tenant doing expensive aggregations consumes the same search thread pool slots as everyone else. Per-tenant isolation must be built *above* ES in your application layer. This research gap shows up in the design: there's no tenant quota sub-system, and the stress test's noisy-neighbor mitigation conflates application-layer rate limiting with ES-internal thread pools.

**Context-setting questions:** Four substantial questions asked and answered (field mappings + analyzers, how queries work, what Decision 2 means mechanically, how reindexing works). These demonstrate genuine engagement and a willingness to build understanding before designing. However, they consumed the vast majority of the time budget. The exercise budget is 40 minutes total (5 min research + 35 min design). Research took 45 minutes (including context questions), leaving the actual design starting at minute 51 — already past the total budget. This time allocation means the design was done under time pressure that the research phase created.

**Anti-pattern present:** Decision 3 was explicitly deferred to the AI-provided context answer. While honest, this means the candidate understood the mechanics (from reading the answer) but didn't apply their own judgment about which option fits *this* problem's constraints. Understanding the options and choosing between them are different skills — the exercise tests the latter.

---

## Ideal System Architecture

```
═══ TENANT API LAYER ═══

    Tenant Requests (ingest docs, search, update schema)
                │
       ┌────────▼──────────┐
       │   API Gateway      │
       │                    │
       │ • Authenticate     │
       │ • Resolve tenant   │
       │ • Per-tenant rate  │
       │   limit (queries/s,│
       │   docs/s, query    │
       │   complexity)      │
       │ • Route to service │
       └──┬─────┬───────┬──┘
          │     │       │
      INGEST  SEARCH  SCHEMA
          │     │     CHANGE
          ▼     │       │
                │       ▼
                │  ┌──────────────────┐
                │  │ Reindex          │
                │  │ Orchestrator     │  (see state machine below)
                │  └──────────────────┘
                │
                ▼
       ┌────────────────┐
       │  Search Router  │
       │                 │
       │ • Resolve tenant│
       │   → ES alias    │
       │ • Add tenant_id │
       │   filter (shared│
       │   indices)      │
       │ • Query timeout │
       │   enforcement   │
       │ • Forward to ES │
       └────────┬────────┘
                │
                ▼
            ES Cluster
        (handles fan-out,
         scoring, merging)

═══ INGESTION PIPELINE ═══

    API Gateway
         │ (validated doc + tenant context)
         ▼
    ┌──────────────────┐
    │ Kafka             │
    │                   │
    │ Partitioned by    │
    │ tenant_id         │
    │                   │
    │ Enables:          │
    │ • Burst absorption│
    │ • Per-tenant      │
    │   throttling      │
    │ • Replay for      │
    │   reindex catch-up│
    └────────┬──────────┘
             │
             ▼
    ┌──────────────────────────┐
    │ Ingestion Workers         │
    │                           │
    │ • Consume from Kafka      │
    │ • Batch by target index   │
    │ • Per-tenant rate cap     │
    │   (e.g., max 2K docs/sec │
    │    to ES per tenant)      │
    │ • Resolve tenant → index  │
    │   via Metadata Service    │
    │ • ES _bulk API writes     │
    └────────┬──────────────────┘
             │
             ▼
         ES Cluster

═══ ELASTICSEARCH CLUSTER (TIERED ISOLATION) ═══

    ┌──────────────────────────────────────────────────┐
    │                                                  │
    │  DEDICATED NODE POOL (hot)                       │
    │  ┌─────────────┐  ┌─────────────┐               │
    │  │ tenant_A    │  │ tenant_B    │  ~240 large    │
    │  │ (dedicated  │  │ (dedicated  │  tenants       │
    │  │  index,     │  │  index,     │  (top 3%)      │
    │  │  own shards │  │  own shards │                │
    │  │  10-50 each)│  │  10-50 each)│  Own indices,  │
    │  └─────────────┘  └─────────────┘  own aliases   │
    │                                                  │
    │  SHARED NODE POOL (warm)                         │
    │  ┌──────────────────────────────────────┐        │
    │  │ shared_index_001  (200 tenants)      │        │
    │  │ shared_index_002  (200 tenants)      │ ~7,760 │
    │  │ ...                                  │ small  │
    │  │ shared_index_039  (160 tenants)      │ tenants│
    │  │                                      │        │
    │  │ _routing = tenant_id                 │        │
    │  │ ~5 shards per shared index           │        │
    │  │ tenant_id filter on every query      │        │
    │  └──────────────────────────────────────┘        │
    │                                                  │
    └──────────────────────────────────────────────────┘

═══ REINDEX STATE MACHINE (per-tenant) ═══

    IDLE
     │  tenant requests analyzer/mapping change
     ▼
    CREATE_NEW_INDEX
     │  create v2 with new mappings + alias setup
     ▼
    REINDEXING
     │  ES _reindex API (throttled to limit I/O impact)
     │  searches continue hitting v1 via alias
     │  new writes go to Kafka (buffered)
     ▼
    CATCH_UP
     │  replay Kafka messages from reindex-start offset
     │  into v2 (docs written during reindex)
     ▼
    SWAP
     │  atomic alias swap: alias → v2
     │  verify queries hitting v2
     ▼
    CLEANUP
     │  delete v1 after grace period
     ▼
    IDLE

    For SHARED-INDEX tenants: extract tenant's docs
    into a temporary dedicated index for reindex,
    then reinsert into shared index (or promote to
    dedicated if tenant has grown)

═══ SUPPORTING SERVICES ═══

    ┌─────────────────────────┐
    │ Tenant Metadata Service  │
    │                          │
    │ • (tenant, index_name)   │
    │   → ES alias, tier,      │
    │   shard count             │
    │ • Analyzer/mapping       │
    │   configs per tenant     │
    │ • Quota configs (rate    │
    │   limits, max doc count) │
    │                          │
    │ Queried by: Gateway,     │
    │   Search Router,         │
    │   Ingestion Workers,     │
    │   Reindex Orchestrator   │
    └─────────────────────────┘

    ┌─────────────────────────┐
    │ Cluster Health Monitor   │
    │                          │
    │ • Track shard count vs   │
    │   ceiling per node       │
    │ • Track per-tenant doc   │
    │   count growth           │
    │ • Auto-promote: when     │
    │   shared tenant exceeds  │
    │   threshold, migrate to  │
    │   dedicated index        │
    │ • Alert on approaching   │
    │   shard/node limits      │
    │                          │
    │ Reads: ES cluster stats, │
    │   Tenant Metadata        │
    └─────────────────────────┘
```

---

## Decision Analysis

### Decision 1: Index and shard isolation strategy

**Correct answer: C (Tiered isolation).** The candidate chose correctly.

**Why A (index-per-tenant) is wrong:** Do the shard math. 8,000 tenants × minimum 1 primary + 1 replica shard = 16,000 shards. But that's unrealistically low — the median tenant has 2M docs (~2 GB), fine at 1 shard, but large tenants with 500M docs (~500 GB) need 10-50 shards. With the top 3% (240 tenants) averaging ~30 shards each, that's ~7,200 shards just for large tenants + ~15,520 for small tenants = ~22,720 total, doubled with replicas = ~45,000 shards. At 1,000 shards/node, you need 45 data nodes, and most of those shards are tiny and wasteful (a 50K-doc tenant's shard holds ~50 MB against an optimal 10-50 GB). The per-shard overhead in heap, file handles, and thread pool slots makes this untenable.

**Why B (shared indices with routing) is wrong:** The top 3% hold 70% of all documents. With custom routing by tenant_id, a large tenant's documents concentrate on 1-2 shards of the shared index. When that tenant does a bulk import at 8K docs/sec, those shards become hot — indexing pressure degrades query performance for every other tenant whose documents happen to be on the same shard or the same node. The noisy neighbor NFR (no more than 20% latency degradation) is nearly impossible to meet when a single tenant can dominate a shared shard's I/O and CPU.

**Why C (tiered) is right:** Small tenants (97%, median 2M docs) are grouped into shared indices — ~200 tenants per shared index with ~5 shards each. Total: ~39 shared indices × 5 shards × 2 (replicas) = ~390 shards. Small tenants don't burst hard enough to create hot shards. Large tenants (3%, 50M-500M docs) get dedicated indices with appropriately sized shards. Total: ~240 tenants × ~20 shards average × 2 = ~9,600 shards. Grand total: ~10,000 shards, needing ~10 data nodes. This is 4.5x more shard-efficient than index-per-tenant, and hot-shard risk is contained because large tenants (the ones that actually burst) are isolated. The tiering threshold should be automated — the cluster health monitor watches doc count growth and promotes tenants when they cross a threshold (e.g., 10M docs).

**How it connects to Decisions 2 and 3:** Tiered isolation means the ingestion pipeline (Decision 2) must be tenant-aware: workers resolve each document to the correct ES index (shared or dedicated) and apply different rate limits per tier. For schema evolution (Decision 3), tiered isolation creates two different reindex scenarios: dedicated-index tenants can use alias swap cleanly, but shared-index tenants need their documents extracted, reindexed separately, and reinserted — a more complex state machine.

### Decision 2: Indexing pipeline architecture

**Correct answer: B (Queue-buffered pipeline).** The candidate chose correctly.

**Why A (direct writes) is wrong:** At 40K docs/sec aggregate with individual tenants bursting to 8K, the API layer would need to manage bulk request batching, retry logic, and backpressure directly against ES. If the ES cluster slows down (e.g., during a merge storm or a large tenant's reindex), API latency spikes and the tenant's ingest calls start timing out. Your platform's write availability becomes coupled to ES's moment-to-moment health. There's also no natural point to enforce per-tenant rate limiting — you'd need to build it into the API server itself, mixing traffic shaping with request handling.

**Why C (write-ahead + CDC) is wrong:** The 5-second searchability SLA is the binding constraint. CDC pipelines (Debezium, DynamoDB Streams, etc.) typically have 2-10 seconds of propagation latency under normal load, and under burst conditions can lag further. You'd be spending your entire latency budget on the replication pipeline before ES even starts indexing. Additionally, this introduces a primary store that must handle 40K writes/sec — a significant infrastructure addition for a system where ES *is* the primary store of search data.

**Why B is right:** Kafka absorbs bursts (8K docs/sec from a single tenant doesn't hit ES all at once), workers batch efficiently into `_bulk` requests, and the queue provides a natural point for per-tenant rate limiting (consume at most X messages/sec per tenant). The 5-second SLA is easily met: Kafka publish latency is ~5ms, worker consumption + batching + ES bulk write takes 1-3 seconds. Kafka's durability (replicated log) means acknowledged documents won't be lost even if ES is temporarily down. And critically for Decision 3: Kafka's offset-based replay enables reindex catch-up — after an alias-swap reindex completes, you replay messages from the reindex-start offset to close the gap.

**How it connects to Decision 3:** The queue-buffered pipeline directly enables the best reindex strategy. With Kafka, you know the offset at which a reindex started. After ES's `_reindex` API completes, you replay from that offset into the new index to capture documents that arrived during the reindex. This closes the catch-up gap that makes alias-swap (3A) clean, without needing dual-write complexity (3B) or query fan-out with inconsistent relevance (3C).

### Decision 3: Reindexing strategy for schema evolution

**Correct answer: A (Alias swap with online reindex), enhanced by Kafka replay from Decision 2.**

The candidate chose C (versioned indices with query fan-out). This is defensible but not optimal.

**Why A is correct (with Kafka enhancement):** The alias swap is the simplest operational model: one old index, one new index, one atomic cutover. The traditional weakness — documents written during the reindex are missing from the new index — is solved by the Kafka queue from Decision 2. The procedure:

1. Record the current Kafka offset for this tenant's partition
2. Create new index with updated mappings/analyzer
3. Run `_reindex` from old → new (throttled via `requests_per_second` to limit cluster I/O impact)
4. When `_reindex` completes, replay Kafka messages from the recorded offset into the new index
5. Atomic alias swap
6. Delete old index

This gives you: zero search downtime (queries hit old index throughout, then new index after swap), consistent relevance (all documents in the new index use the new analyzer), no dual-write complexity, and no query fan-out overhead. The Kafka replay makes what would be A's fatal flaw into a non-issue.

**Why B (dual-write) is worse:** Doubles indexing load for the tenant during migration. For a large tenant with 500M documents, the reindex might take hours — during which every new document is indexed twice. At 8K docs/sec burst rate, that's 16K docs/sec hitting ES for one tenant. This is the opposite of noisy neighbor isolation.

**Why C (query fan-out) is worse:** During migration, every query hits two indices and merges results. This adds latency (more shard fan-out) and, more importantly, produces **inconsistent relevance scoring.** If the tenant changed their analyzer from English to multilingual stemming, documents in v1 are analyzed differently from documents in v2. BM25 scores are computed per-index, so the same term has different IDF values in v1 vs v2. Merging these results produces rankings that are neither correct under the old analyzer nor the new one — they're an incoherent blend. For a search platform, relevance quality is the product. Option C degrades it during every migration.

**How this connects back:** The three decisions form a coherent stack: tiered isolation (C) → right-sized indices per tenant class; queue-buffered pipeline (B) → burst absorption + per-tenant rate control + Kafka replay capability; alias swap with replay (A) → clean reindex leveraging Kafka's offset tracking. Each decision enables or improves the next.

---

## Key Differences

| Aspect | Their Design | Ideal Design | Why It Matters |
|--------|--------------|--------------|----------------|
| Core challenge | "Routing a shard to its persistent location" (write path focus) | Managing tenant placement and lifecycle on shard-constrained shared infrastructure | Misidentifying the core challenge means optimizing the wrong thing — the write path is well-understood; the shard strategy is where this system lives or dies |
| Sub-system decomposition | ES internals described as platform sub-systems (query aggregation = coordinating node, shards = sub-system) | Clear boundary: platform services (gateway, metadata, ingestion workers, reindex orchestrator) vs. ES internals (shard fan-out, scoring, merging) | Conflating your system with ES's internals means you'll either re-implement what ES already does or miss what ES doesn't do (tenant isolation, schema lifecycle) |
| Tenant isolation mechanism | No dedicated sub-system; mentioned only in stress test | Per-tenant rate limiting at gateway + per-tenant ingestion throttling in workers + dedicated node pools for large tenants | The noisy neighbor NFR is the hardest constraint in the problem. Without a first-class isolation mechanism, one tenant's bulk import or expensive query degrades everyone |
| Decision 1 mechanics | "Inverted index mapping query aspects to indices" + "indices as JSON-like structures in a document DB" | Metadata service mapping (tenant, index_name) → ES alias; ES aliases as indirection layer | Confused mechanics make it impossible to evaluate whether the design actually works. The metadata lookup is a simple DB query + cache, not an inverted index |
| Decision 3 choice | Versioned indices with fan-out (inconsistent relevance during migration) | Alias swap with Kafka replay for catch-up (consistent relevance, clean cutover) | For a search platform, relevance consistency is the product promise. Mixed-analyzer results during migration is a search quality regression, not just a technical tradeoff |
| Decision 3 mechanics | "Described in context.md by the AI. I did not design this." | Full state machine: create v2 → reindex (throttled) → Kafka replay catch-up → alias swap → cleanup | An undesigned decision is an incomplete design. The mechanics of reindex are where the real complexity lives — especially for shared-index tenants |
| Shard math | No calculation of total shard count or node requirements | ~10,000 shards with tiered model → ~10 data nodes; vs ~45,000 shards with index-per-tenant → ~45 nodes | The shard constraint (1,000/node) is the binding physical limit. Without doing the math, you can't validate whether your isolation strategy is viable |
| Reindex orchestration | No sub-system for managing schema evolution | Reindex orchestrator with explicit state machine (IDLE → CREATE → REINDEX → CATCH_UP → SWAP → CLEANUP) | Schema evolution is a core FR. For shared-index tenants, it's particularly complex: extract docs → reindex separately → reinsert. Without an orchestrator, this is an unmanaged manual process |

---

## Core Mental Model

**How they conceptualized the problem:** "This is a pipeline problem — the challenge is getting documents from tenants into their shards efficiently." The design energy went into the ingestion path (Decision 2 got the clearest treatment), and the shard strategy (Decision 1) was treated as a supporting concern with confused mechanics. The sub-systems were organized around "what touches a document on its journey" — upload system, routing, query aggregation, shards.

**How they should have conceptualized it:** "This is a resource management problem — the challenge is packing 8,000 tenants with 1000:1 size variance onto shared, shard-constrained clusters while maintaining per-tenant guarantees." The organizing principle isn't "how does a document flow" — it's "how are tenants allocated, isolated, monitored, and migrated on shared infrastructure."

**Why this reframe leads to better decisions:**

The pipeline framing makes Decision 2 (indexing architecture) feel like the most important choice, because it's the core of the pipeline. The resource management framing makes Decision 1 (shard isolation strategy) feel like the most important choice, because it determines how resources are allocated — and it is.

The pipeline framing makes you think of ES as the end of a conveyor belt. The resource management framing makes you think of ES as a constrained pool of resources (shards, nodes, thread pools, heap) that your system must manage on behalf of tenants. This shift surfaces the sub-systems that were missing from the design: tenant metadata service (who goes where), cluster health monitor (are we approaching limits), reindex orchestrator (how do we safely evolve tenant schemas without destabilizing the pool), and per-tenant rate limiter (how do we enforce fair sharing).

The pipeline is infrastructure. The resource management is the product. A search-as-a-service platform's core value proposition is: "you get search without operating a cluster." That means your system's primary job is operating the cluster on behalf of tenants — placement, isolation, lifecycle, capacity. The pipeline is a means to that end, not the end itself.
