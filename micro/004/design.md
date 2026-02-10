# Design: Container Image Registry

Total time budget: 40 minutes (research + design + buffer).
If stuck on any phase beyond its time box, write what you know, mark it incomplete, and move on.

## Phase 1: Framing (5 min)

**One-line summary:** We're designing a image registry that can allow users to efficiently store and query huge amounts of images.

**Core technical challenge:** Efficient storage of the data

**Framing:** Which of these best fits this problem?
- [X] A storage and retrieval problem
- [ ] A streaming/pipeline problem
- [ ] A coordination/consensus problem
- [ ] A scheduling/state machine problem
- [ ] A distributed counting/aggregation problem

Time elapsed: 41 mins.

**The decision that matters most:**

## Phase 2: Sub-systems (5 min)

Name 3-5 sub-systems. One sentence each. Do NOT describe how they work — that's Phase 3.
If you're writing more than 5 lines, you're over-designing this phase.

- Storage & querying engine => A user hits this to store and query their images
- Content addressing system => this takes the user's metadata and generates the content address for it
- Persistence layer => This is where the content-addressed data is stored
- Data movement system => Moves the data from hot storage to colder storage for cost-effectiveness
- Usage-based Garbage collector => Deletes unused image layers
- Retention-based garbage collector => Deletes images who are beyond their retention policy 
- Vulnerability scanner => Scans all layers against a CVE DB

Time elapsed: 51 mins.

## Phase 3: Key Decisions (15 min)

### Decision 1: Layer storage and deduplication model

- **A.** Global content-addressed object store — one flat namespace keyed by digest, all repositories reference the same physical blob
- **B.** Repository-scoped storage with a cross-repo link table — each repo logically "owns" its layers, a reference table tracks sharing for dedup
- **C.** Chunked sub-layer deduplication — break layers into variable-size chunks (like rsync/Borg), dedup at chunk level for finer granularity

**Your choice:**
**Why:**
- We can't do B because then we don't get the duplication property between repos.
- We can't do C because the operations on the layer would be complex and expensive.
- With A, the content-addressing will ensure all instances of the same layer data route to the same place.
- We only do A though with sharging becasue because the raw data size (200TB de-duplicated) is too large for one store.
**How it works:**
This is the "Persistence layer" and "Content addressing system" in the design.

The content addressing system would listen for a pub/sub message that new image layer has arrived. That data is around 40MB so it should exist in a temporary blob store + a cache.

Then, it should then perform the content-addresing hash to create the digest.

We should then have an in-memory mapping from digests to the sharded blob stores that contain the content-addressed data. If none exists, we should pick one at random or via some heuristic (least current usage). We should shard based on the hash of the content address.

Manifests should exist in a sharded document store. A user's image metadata could have a path like `userId/imageName/imageTag/metadata.json` with the metadata in that tag. And for `latest`, we could fetch the imageTag directory with the highest createdAt time.

### Decision 2: Garbage collection strategy for shared layers

- **A.** Mark-and-sweep — periodically scan all manifests to build a live set, delete anything not in it, block deletes during sweep
- **B.** Reference counting — increment on push, decrement on delete/retention, collect at zero with deferred decrement reconciliation
- **C.** Lazy tombstoning — mark layers as "pending deletion" on last known dereference, grace period before physical delete, reconciliation sweep catches missed decrements

**Your choice:** A (mark-and-sweep)
**Why:** The layer usage here resembles a graph (X is used by Y). Reference counting would require traversing for any child layers that depend on a deleted parent, and so on recursively. Lazy tombstoning would require the same.
**How it works:**
Conceptualizing this as a graph traversal, we'll need to create an in-memory graph in this worker node.

We'll add nodes for all layers (we'll need some central store of all layers in the system currently, we can query it in batches).

Then, we can do a "BFS" of layers' child layers and mark them as visited in the in-memory graph.

Then, we can delete the unvisited nodes from the system by deleting them from the central DB and creating pub/sub jobs (to be picked up by workers) to delete batches of unused data.

This can happen as a periodic job.

If the memory footprint, of all these nodes it too large, we might need an additional scaling technique. I'm not sure what that would be. Maybe some kind of map-reduce operation.

### Decision 3: Multi-region layer distribution

- **A.** Full active replication — every layer replicated to every region on push, all regions serve from local storage
- **B.** Pull-through cache — one origin region stores all layers, other regions fetch and cache on first pull with CDN in front
- **C.** Tiered replication — popular base layers pre-replicated to all regions, long-tail layers served via pull-through cache on demand

**Your choice:** B
**Why:** Full active replication gives us important latency gaurantees for popular images, but is far too much storage overhead for all images. Pull-through cache will likely take a performance hit on the very first query of a popular image, but then therafter, the cache will keep that image in-region and it will be high-performance. Our NFRs refer to the p95, and the cache should keep most of the cases within-region. 
**How it works:**
The persistence system is deployed in a few different regions. In each, we also have a cache with an LRU policy. The API doing the querying ("Storage & querying engine") will check the cache, then the regional persistence. Then, if the image is in neither, it will have to check with the routing component (mapping content addresses to where to find them globally) which cache to go check. Then, it'll pull the layer from that other region's cache and store it in the local region's cache.


## Phase 4: Stress Test (5 min)

Pick the 3-5 most critical NFRs and failure modes. You cannot cover all of them — prioritize.

| Concern | Type | How addressed | Confidence |
|---------|------|---------------|------------|
| | NFR / Failure | | high / medium / low |

- A huge layer is uploaded (many GB) => We'll need to accept streams for large layer data. Medium confidence this is handled
- Huge spike in pulls => The persistence layer is sharded, the api pods they hit can be horizontally scaled. High confidence this is tackled.
- Pull latency => high confidence this is addressed through the storage mechanism and horizontal scaling of stateless APIs (routing service).

---

Time elapsed: 99 mins.

---

## Review

**Score: 4/10**

| Criterion | Score | Notes |
|-----------|-------|-------|
| Problem Framing | 4/10 | Correct category, but core challenge too vague, key decision left blank |
| Key Decisions | 4/10 | D1 right choice with wrong mechanics, D2 wrong choice from wrong mental model, D3 acceptable |
| Tradeoff Reasoning | 4/10 | Rejections based on misreads (D1) and incorrect model (D2); D3 reasoning was strongest |
| Stress Testing | 3/10 | Surface-level concerns, missed critical failure modes, ignored table format |
| Communication | 4/10 | 2.5x over time budget, key sections blank, Phase 2 over-scoped |

### Framing Assessment

**Their framing:** "Efficient storage of the data"
**Correct framing:** Lifecycle management of content-addressed blobs with massive cross-repository fan-in — specifically, knowing when it's safe to delete a blob that millions of independent manifests may reference, while not blocking operations and controlling cross-region replication cost.
**Match:** Partially correct

Storage is the right *category*, but "efficient storage" is too vague to drive decisions. Storing blobs in S3 is trivially solved. The hard part is the *lifecycle*: reference tracking across 60M manifests, safe garbage collection without blocking, and cost-controlled global distribution. This vagueness directly led to Phase 2 including a "data movement system" (not in requirements) while missing the reference-tracking metadata store (critical to every decision).

### Phase-by-Phase Feedback

**Phase 1 (Framing):** The framing category (storage and retrieval) is correct. But "efficient storage of the data" doesn't name what makes this hard — it could describe any storage system. The one-line summary ("allow users to efficiently store and query huge amounts of images") is a product description, not a technical framing. "The decision that matters most" was left blank, which is the most important signal of whether you understand the problem. It should be something like: "How to safely reclaim shared layers without corrupting live images."

**Phase 2 (Sub-systems):** Listed 7 sub-systems against a 3–5 guideline. Several issues:
- "Content addressing system" isn't a sub-system — it's a property of the storage model. In OCI, the client computes the SHA-256 digest before upload. The registry verifies it. There's no background worker computing hashes.
- "Data movement system" (hot-to-cold tiering) isn't in the requirements. This is scope creep.
- Two separate garbage collectors (usage-based and retention-based) conflate two things. Retention policies delete *manifests/tags*. When a manifest is deleted, its layer references are decremented. There's one layer GC — the thing that reclaims zero-reference layers. Retention is just one of several triggers for dereference.
- Missing: **metadata store** (the manifest → layer reference graph, tag → manifest pointers), **CDN/distribution layer**, **multi-region replication sub-system**. These are more architecturally important than data tiering.

**Phase 3 (Key Decisions):**

*Decision 1:* Correct choice (A), but the reasoning and mechanics have significant issues. The rejection of B ("we don't get the duplication property between repos") is a misread — option B explicitly says "a reference table tracks sharing for dedup." B still deduplicates physical storage; it just scopes logical ownership. The mechanics describe sharding S3 because "200TB is too large for one store" — S3/GCS scale to exabytes without sharding. 200TB is a single bucket. The pub/sub workflow for content-addressing doesn't match reality — OCI clients push layers via HTTP PUT with the digest already computed. The registry verifies, it doesn't compute.

*Decision 2:* Wrong choice. Mark-and-sweep is the worst option for this problem. The stated reason — "reference counting would require traversing for any child layers that depend on a deleted parent, and so on recursively" — reveals a misunderstanding of the data model. Layers don't have child layers. The graph is shallow: tag → manifest → layers. Two levels. Reference counting means: when a manifest is deleted, decrement the ref count on each of its ~5 layers. No recursion. The chosen option (mark-and-sweep) explicitly says "block deletes during sweep," which directly violates the NFR that GC must not block push/pull operations.

*Decision 3:* Acceptable choice (B). The reasoning about p95 and cache hit rates is sound. However, option C (tiered replication) is stronger because base layers like `ubuntu:22.04` represent a tiny fraction of storage but serve a huge fraction of pulls — pre-replicating just those eliminates first-pull latency for the common case while keeping long-tail costs low. The mechanics describe a reasonable flow but introduce a "routing component" that is really just the metadata store.

**Phase 4 (Stress Test):** Only 3 concerns listed, none in the requested table format. All are surface-level:
- "Huge layer uploaded" — Valid but handled by OCI chunked upload protocol. Not the most critical concern.
- "Huge spike in pulls" — Handwaved with "horizontal scaling." The real answer is CDN absorbs read spikes for immutable content-addressed blobs.
- "Pull latency" — Restates an NFR without identifying a specific failure mode.

Missing critical concerns: GC race condition (a push references a layer that GC is about to delete), reference count drift over time, cross-region replication lag vs the 60-second requirement, CVE database update triggering re-evaluation of millions of cached scan results, retention policy execution at scale (bulk manifest deletions cascading into layer dereferences).

### Research Assessment

The context-setting questions showed genuine curiosity and were good instincts. The guided research had mixed quality:

- **Content-addressable GC:** Described it as "a graph" requiring "traversal of child layers" — this is wrong for OCI's model. The graph is shallow (manifests → layers), not a deep recursive structure. This incorrect mental model *directly* caused the wrong GC decision in Phase 3. This is the worst kind of research outcome: a plausible-sounding conclusion that's wrong for this specific domain, confidently applied to the decision.
- **Pull-through vs active replication:** Good comparison of cost/latency tradeoffs. The note about content-addressability helping cache correctness is correct and insightful. This research *did* translate into Decision 3's reasoning, which was the strongest of the three.

---

## Ideal System Architecture

```
═══ PUSH FLOW ═══

Client (computes digest)
  │
  ▼
API Gateway ───▶ Auth / ACL Check
  │
  ├── Blob exists? (HEAD by digest)
  │     YES → skip upload (dedup)
  │     NO ──▼
  │         Chunked upload → Object Store (S3)
  │         key: sha256:<hex>
  │
  ├── Store manifest
  │     ▼
  │   Metadata DB
  │   ┌──────────────────────────────┐
  │   │ repos(repo_id, name, owner)  │
  │   │ tags(repo_id, tag, manifest) │
  │   │ manifests(digest, layers[])  │
  │   │ layer_refs(layer_digest,     │
  │   │   manifest_digest, refcount) │
  │   └──────────────────────────────┘
  │
  └── Emit event: "manifest pushed"
        │
        ▼
      Scan Queue ───▶ Vulnerability Scanner
                      (per-layer, results cached by
                       layer_digest + cve_db_version)


═══ PULL FLOW ═══

Client
  │
  ▼
CDN Edge (cache by digest — immutable, infinite TTL)
  │ MISS
  ▼
Regional Cache (pull-through from origin)
  │ MISS
  ▼
Origin Object Store (S3)


═══ GARBAGE COLLECTION ═══

Retention Worker (periodic)
  │  evaluates tag retention rules per repo
  │  deletes expired tags/manifests
  │  decrements layer refcounts
  ▼
Layer Refcount → 0?
  │
  YES → Tombstone (mark pending-delete, start grace period)
  │
  ▼ (grace period elapsed, e.g. 6h)
Deletion Worker
  │  verify refcount still 0 (safety check)
  │  delete from Object Store
  │
Reconciliation Sweep (weekly)
  │  scan all manifests → build live layer set
  │  compare against layer index
  │  catch any leaked layers (missed decrements)


═══ MULTI-REGION ═══

                   ┌──────────┐
                   │ Origin   │ ◄── all pushes land here
                   │ Region   │     (metadata DB is here)
                   └────┬─────┘
                        │
            ┌───────────┼───────────┐
            ▼           ▼           ▼
     ┌──────────┐ ┌──────────┐ ┌──────────┐
     │ Region A │ │ Region B │ │ Region C │
     │ CDN edge │ │ CDN edge │ │ CDN edge │
     │ + cache  │ │ + cache  │ │ + cache  │
     └──────────┘ └──────────┘ └──────────┘

Popular base layers (ubuntu, alpine, node)
  → pre-replicated to all regions on push
Long-tail app layers
  → pull-through cached on first access
```

---

## Decision Analysis

### Decision 1: Layer storage and deduplication model

**Correct answer: A — Global content-addressed object store.**

Option A is correct because the OCI spec already defines content-addressing at the layer level. The digest IS the storage key. One S3 bucket, key = `sha256:<hex>`. Deduplication is automatic: two pushes of the same layer produce the same key and the second upload is a no-op (check with HEAD request before upload).

Why B is worse: B adds a layer of indirection — each repo "owns" layers with a cross-repo link table for dedup. This is the model you'd use if layers were mutable or if you needed per-repo access control on individual blobs. But layers are immutable and content-addressed, so the link table is pure overhead. Access control belongs at the manifest/repo level, not the blob level — if you know a layer's digest, you've already been authorized to see a manifest that references it.

Why C is worse: Sub-layer chunking catches near-duplicate layers but at enormous cost: metadata index grows 100x+, pulls require reassembly, OCI clients don't understand chunks. Layer-level dedup already captures the big wins (shared base images). The marginal storage savings don't justify the complexity at 200TB scale.

**Cascading effect:** Getting A right means your GC system operates on a flat set of blobs with a refcount per blob. Getting this wrong (e.g., choosing B with per-repo ownership) would make GC far more complex — you'd need to reason about cross-repo links.

### Decision 2: Garbage collection strategy

**Correct answer: C — Lazy tombstoning.**

This is the most consequential decision in the design and the one most directly constrained by the NFRs. The requirements say: (1) reclaim within 24 hours, (2) must not block push/pull, (3) deleting a live-referenced layer corrupts images.

Why A is wrong: Mark-and-sweep requires scanning all 60M manifests to build the live set. During the sweep, you must freeze deletes (or risk a push adding a reference to a layer you're about to delete). This directly violates "must not block push/pull." Even if you allow concurrent operations, you have a race condition: a push completes between your sweep snapshot and the delete phase, referencing a layer you've marked as dead. At 60M manifests × 5 layers, the sweep itself takes significant time, widening this race window.

Why B is fragile: Pure reference counting works if you never lose a decrement. But in a distributed system with 8K pushes/sec, network partitions, retries, and partial failures, lost decrements are inevitable. A lost decrement means a layer's refcount never reaches zero — it leaks storage forever. Without a reconciliation mechanism, leaked storage accumulates silently. B *could* work with reconciliation bolted on, but at that point you've reinvented C.

Why C is right: When a layer's refcount hits zero, you don't delete immediately — you tombstone it with a timestamp. After a grace period (e.g., 6 hours), a deletion worker verifies the refcount is *still* zero and only then deletes. This handles the race condition: if a push references the layer during the grace period, the refcount goes back above zero and the tombstone is cleared. A periodic reconciliation sweep (weekly) scans all manifests, builds the live set, and catches any layers where a decrement was lost — these get tombstoned and go through the same grace period. The 24-hour reclaim SLA is easily met with a 6-hour grace period. Nothing is ever blocked.

**Cascading effect:** Decision 1 (global namespace) means each layer has exactly one refcount to manage. If you'd chosen B (per-repo ownership), you'd need refcounts per repo AND globally, making tombstoning much harder to reason about.

### Decision 3: Multi-region distribution

**Correct answer: C — Tiered replication.**

The constraints create a tension: "pullable from any region within 60 seconds" and "CDN-edge speed" (NFR) vs "cross-region replication costs money per GB transferred" (constraint). Neither A nor B resolves this tension well.

Why A is wrong: Full replication of 200TB to every region is enormously expensive in both storage and egress. Most of that 200TB is long-tail app layers that may only be pulled from the region they were pushed to. You're paying to replicate data nobody in that region will ever request.

Why B is suboptimal: Pull-through cache works but guarantees a cold-start latency hit for every first pull in every region. For base layers (`ubuntu`, `alpine`, `node`) that are pulled thousands of times per day in every region, the first-pull penalty is a real latency issue — and these layers are a tiny fraction of total storage. You'd pay cross-region egress once per layer per region regardless, but with B you also eat the latency on first pull for layers you *know* will be popular.

Why C is right: Base images and common platform layers (maybe the top 1% by pull frequency — a few TB at most) are pre-replicated to all regions on push. These serve the vast majority of pulls at CDN speed with zero cold-start. Everything else goes through pull-through caching. Content-addressability makes this clean: the CDN and regional cache can use infinite TTL because a digest's content never changes. Cache invalidation is a non-problem. The result: low latency for the common case, low cost for the long tail, and the 60-second SLA is met by regional replication lag for hot layers and pull-through latency for cold ones.

**Cascading effect:** Decision 1 (global namespace) means the CDN cache key is just the digest — no repo-scoping needed. Decision 2 (tombstoning with grace period) means you can safely cache aggressively — a layer won't be deleted while CDN edges still serve it, because the grace period exceeds CDN TTLs.

---

## Key Differences

| Aspect | Their Design | Ideal Design | Why It Matters |
|--------|--------------|--------------|----------------|
| Core challenge | "Efficient storage of the data" | GC safety for shared content-addressed blobs with massive fan-in | Vague framing led to sub-systems that don't address the actual hard problem |
| Object store model | Sharded blob stores with routing table | Single S3 bucket, key = digest | S3 scales to exabytes; sharding adds unnecessary complexity and a routing layer |
| Push protocol | Pub/sub message triggers background hash computation | Client computes digest, HTTP PUT, registry verifies | OCI spec defines this; misunderstanding it means the push pipeline is wrong |
| Manifest storage | Document store with filesystem-like paths | Relational metadata DB (repos, tags, manifests, layer_refs) | Tag → manifest → layers is relational; reference counting requires relational joins |
| GC strategy | Mark-and-sweep with in-memory graph | Lazy tombstoning with grace period + reconciliation sweep | Mark-and-sweep blocks operations (violates NFR) and has race conditions at scale |
| Data model mental model | Deep graph with recursive child layers | Shallow two-level: manifest → layers | Wrong model directly caused wrong GC decision |
| Multi-region | Pull-through cache only | Tiered: pre-replicate popular base layers, pull-through for long tail | Misses the insight that a tiny fraction of layers serve most pulls |
| CDN strategy | Not mentioned | CDN with infinite TTL on content-addressed blobs | Immutability means cache invalidation is a non-problem — this is a massive architectural advantage left on the table |

---

## Core Mental Model

**How they conceptualized the problem:** As a data storage system — how to put large blobs into stores efficiently, how to shard them, how to move them between tiers.

**How they should have conceptualized it:** As a **reference-management problem with storage economics constraints**. The blobs themselves are trivially stored (S3 handles that). The hard part is the *metadata graph* — millions of manifests pointing to shared layers — and the lifecycle operations on that graph: safely determining when a blob has zero live references, handling race conditions between concurrent pushes and GC, and choosing which blobs to replicate where based on access patterns to minimize cost while meeting latency SLAs.

**Why this reframe matters:** When you think "storage problem," you design sharding, tiering, and routing — none of which are needed here (S3 already solves them). When you think "reference management problem," you design the metadata schema first (what points to what), then GC safety (how to delete without corruption), then distribution economics (what to replicate based on the reference/access graph). The metadata store — which was absent from their sub-system list — becomes the centerpiece of the architecture. Every decision flows from how you model and maintain the manifest → layer reference graph.
