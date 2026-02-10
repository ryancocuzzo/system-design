# Context — Multi-Tenant Search-as-a-Service

## Guided Research (5 min)

## Context-setting questions

### Q: What is a custom field mapping? What is a text analyzer? How are both used in this scenario?

**Field mappings** define the schema of a document in Elasticsearch — what fields exist and how each field is stored and indexed. When a tenant creates a search index, their mapping says things like: "`title` is a `text` field (full-text searchable), `price` is a `float` (range-filterable), `tags` is a `keyword` array (exact-match filterable), `variants` is a `nested` object (supports querying child objects independently)." The mapping determines what queries are possible and how data is physically stored on disk. Unlike a relational schema, you can't easily change a mapping on an existing index — changing a field from `keyword` to `text`, or altering how it's analyzed, requires reindexing all documents.

**Text analyzers** control how text fields are tokenized and normalized before being written to the inverted index. An analyzer is a pipeline with three stages:

1. **Character filters** — transform raw text before tokenization (e.g., strip HTML tags, normalize Unicode)
2. **Tokenizer** — splits text into tokens (e.g., standard tokenizer splits on whitespace/punctuation; n-gram tokenizer produces overlapping substrings for partial matching)
3. **Token filters** — transform individual tokens (e.g., lowercase, stemming `"running"→"run"`, synonym expansion `"sneakers"→["sneakers","shoes"]`, stop word removal)

In this scenario, each tenant configures both because their search needs differ. An e-commerce tenant selling in French needs a French stemmer and synonym list mapping brand aliases. A code documentation platform might use a custom tokenizer that preserves dots and underscores (`"System.IO.File"` should not split into three tokens). A legal search tenant might need exact phrase matching with no stemming at all. The analyzer directly determines search quality — it controls what matches what — so tenants must own this configuration.

The combination matters: the mapping says *which fields are searchable text*, and the analyzer says *how that text is processed*. Both are baked into the index at write time. If a tenant decides their synonym list was wrong or wants to switch from English to multilingual stemming, every existing document must be re-analyzed and rewritten — that's the schema evolution requirement in the problem.

### Q: How are full-text queries done performantly? What does "index" mean here?

**Inverted index** — this is the core data structure. Think of it as the index in the back of a textbook, but for every unique term across every document:

```
Term        → Document IDs (postings list)
─────────────────────────────────────────
"running"   → [doc_3, doc_17, doc_42]
"shoes"     → [doc_3, doc_8, doc_17]
"waterproof"→ [doc_8, doc_42]
```

When a user searches `"running shoes"`, Elasticsearch looks up both terms in the inverted index, gets their postings lists, and intersects them → `[doc_3, doc_17]`. This is an O(n) merge of sorted lists, not a scan of every document. That's why full-text search is fast even over millions of documents — the inverted index turns "find documents containing these words" into sorted list operations.

On top of the inverted index, Elasticsearch stores:
- **Doc values** — column-oriented storage for sorting, aggregations, and facets (e.g., sorting results by price, computing a histogram of ratings)
- **Stored fields** — the original document source, returned in search results
- **Norms** — per-field length normalization factors used in relevance scoring (shorter fields matching a term score higher)

**"Index" in Elasticsearch** has a specific meaning distinct from a database index. An Elasticsearch index is the top-level container for a collection of documents that share the same mapping. It's closer to a "table" in relational terms, except it's also the unit of shard allocation. Each index is split into one or more **shards**, where each shard is a self-contained Lucene index (the actual inverted index + doc values + stored fields on disk). When you query an Elasticsearch index, the query fans out to all its shards in parallel, each shard returns its top results, and a coordinating node merges them.

So the hierarchy is: **Cluster → Index → Shards → Segments (immutable Lucene files on disk)**. The problem's shard constraint (max ~1,000 per node, optimal size 10–50 GB) matters because each shard carries fixed overhead — file handles, memory for segment metadata, thread pool slots for search — regardless of how much data it holds. An index with 1 shard holding 100 MB wastes almost as many node resources as one holding 30 GB.

Time elapsed: 23 mins.

-------------

Research these before starting your design. For each, write 1-2 sentences on what it is and when you'd use it. If time runs out, move on.

1. **Data Modeling and Storage** — What are the tradeoffs between index-per-tenant and shared indices with custom routing in Elasticsearch, particularly around shard count limits, query fan-out, and hot shard risk?

Index-per-tenant:
- shard count too high (linear to # customers)
- fan-out is low (can always route to just one shard)
- hot shard risk is low (a customer would only impact the performance of their shard)

Shared indices
- shard count is better (we decide it depending on NFRs, sub-linear)
- fan-out is variable, depending on strategy (customer documents live across several or many shards)
- hot shard risk is high (a high volume surge for one customer impacts others on their shared shard)

2. **Reliability and Resilience** — How do Elasticsearch's thread pool queues, circuit breakers, and shard-level request cancellation work, and how could they be leveraged to enforce per-tenant resource isolation on a shared cluster?

- Thread pools can be used to protect nodes themselves (by ensuring we're only running so many queries at a time)
- circuit breakers can be used also to protect the nodes (by ensuring requests too large that will likely fry the system are not sent)
- request cancellation can be used in the event of requests that are stalling or problematic for some reason. Again, this would protect at the node-level.

Total time elapsed: 45 mins.

## During Design

Q: The design doc template shows the following:
```
### Decision 2: Indexing pipeline architecture

- **A.** Direct Elasticsearch writes — application tier sends bulk index requests directly to the cluster
- **B.** Queue-buffered pipeline — documents flow through a message queue with consumer workers that batch-write to Elasticsearch
- **C.** Write-ahead to primary store with async sync — documents persist in a durable primary store first, then sync to Elasticsearch asynchronously via change data capture
```

What is this system supposed to be doing, that is being described? What would an index request look like? What does it mean to be "writing to" or "syncing to" elasticsearch?

**A:** This decision is about the **write path** — how does a tenant's document get from their API call into Elasticsearch so it becomes searchable?

The concrete scenario: a tenant (say, an e-commerce platform) calls your platform's API to add or update a product in their search index. Something like:

```
POST /v1/indices/my-products/documents
{
  "id": "shoe-42",
  "title": "Waterproof Running Shoes",
  "price": 89.99,
  "tags": ["running", "waterproof"]
}
```

Your platform must get this JSON document into Elasticsearch. Under the hood, that means making an **index request** to ES — literally an HTTP call to the ES cluster:

```
PUT /my-products-index/_doc/shoe-42
{
  "title": "Waterproof Running Shoes",
  "price": 89.99,
  "tags": ["running", "waterproof"]
}
```

When ES receives this, it runs the document through the analyzer (tokenize "Waterproof Running Shoes" → `["waterproof", "running", "shoes"]`), writes the terms into the inverted index, stores doc values for `price`, stores the original source — and now queries can find it. That's what "writing to Elasticsearch" means: sending documents so ES processes and indexes them. "Syncing to" means the same thing, but implies it happens asynchronously from the original write — the document was already persisted somewhere else, and ES gets updated afterwards.

In bulk, ES supports a `_bulk` API that batches hundreds or thousands of index operations into a single HTTP request, which is far more efficient than one-at-a-time.

The three options differ in **what sits between the tenant's API call and that ES index request:**

- **A (Direct):** Your API server receives the tenant's request and immediately makes the ES bulk index call. Simple, low latency, but if ES is slow or overloaded, the tenant's API call blocks or fails. Your platform's write availability is coupled to ES's health.
- **B (Queue-buffered):** Your API server accepts the document, writes it to a message queue (e.g., Kafka), and immediately ACKs to the tenant. Separate worker processes consume from the queue and batch-write to ES. The tenant gets fast acknowledgment. ES pressure doesn't block the API. Workers can batch efficiently. But there's a lag between ACK and searchability.
- **C (Write-ahead + CDC):** The document is first persisted in a primary database (e.g., Postgres, DynamoDB) — that's the source of truth. Then a change data capture (CDC) process detects the new/updated row and writes it to ES. ES is treated as a secondary derived index, not the primary store. Most durable, but most complex and highest latency to searchability.

Q: Decision 3 asks the following:
```

### Decision 3: Reindexing strategy for schema evolution

- **A.** Alias swap with online reindex — create a new index with updated mappings, reindex documents from old to new, atomically swap the alias
- **B.** Dual-write with catch-up — start writing to both old and new index, backfill existing documents into the new index, cut over when caught up
- **C.** Versioned indices with query fan-out — create a new index version for new writes, query across both old and new indices, gradually migrate old documents in the background
```

Q: How do we know what indices to be updating in the first place, from the user request? I referenced a mapping in my architecture, but that process feels ambiguous to me.

Q: How would any of these be done mechanically, in practice? How would we create an index, update mappings, reindex documents and so on?

**A (resolving tenant requests to ES indices):**

Your platform maintains a metadata layer — a mapping from tenant + tenant's logical index name to the actual underlying ES index name(s). When a tenant creates a "products" index through your API, your platform:

1. Records the mapping: `{tenant: "acme", index: "products"} → ES index "acme_products_v1"`
2. Creates the actual ES index `acme_products_v1` with the tenant's mappings/analyzers
3. Creates an ES **alias** `acme_products` pointing to `acme_products_v1`

From then on, all document writes and search queries for tenant "acme", index "products" route to the alias `acme_products`. The alias is the indirection layer — it lets you swap what's behind it without the tenant knowing.

This metadata lives in your platform's own database (Postgres, etc.), not in ES. When a request comes in, your API layer looks up the tenant + index name → resolves to the ES alias → sends the ES request. If you chose shared indices (Decision 1B/C), this mapping would instead resolve to a shared index name plus a routing value, but the lookup is the same idea.

**A (mechanical operations — how reindexing actually works):**

Here's what each option does concretely, using the alias setup above:

**Option A — Alias swap:**
```
Current state: alias "acme_products" → "acme_products_v1" (old analyzer)

1. Create new ES index:
   PUT /acme_products_v2  { mappings: {...}, settings: { analysis: {new analyzer} } }

2. Reindex from old to new (ES does this server-side):
   POST /_reindex
   { "source": { "index": "acme_products_v1" },
     "dest":   { "index": "acme_products_v2" } }
   
   ES reads every doc from v1, runs it through v2's new analyzer, writes it to v2.
   This can take minutes to hours depending on doc count.

3. During reindex, searches still hit the alias → v1 (old data, old analyzer). 
   Writes also go to v1.

4. When reindex completes, swap the alias atomically:
   POST /_aliases
   { "actions": [
       { "remove": { "index": "acme_products_v1", "alias": "acme_products" } },
       { "add":    { "index": "acme_products_v2", "alias": "acme_products" } }
   ]}

5. Delete v1 when safe.

Problem: Documents written to v1 *during* the reindex are lost from v2.
You need to either pause writes, or do a second catch-up pass, or accept
that some recent docs use the old analyzer until they're naturally updated.
```

**Option B — Dual-write with catch-up:**
```
1. Create acme_products_v2 with new mappings.

2. Update your write path: every new document write goes to BOTH v1 and v2.
   v1 uses old analyzer, v2 uses new analyzer. Searches still hit v1 only.

3. Run a background backfill: read all existing docs from v1, write them to v2.
   Because dual-write is already active, new docs are covered.

4. When backfill completes, v2 has everything. Swap the alias to v2.

5. Stop dual-writing to v1. Delete v1.

Advantage: No window where new writes are missed.
Cost: Your write path is more complex during the migration, and you're 
temporarily writing twice (doubles indexing load for that tenant).
```

**Option C — Versioned indices with query fan-out:**
```
1. Create acme_products_v2 with new mappings.

2. Point new writes to v2 only. Stop writing to v1.

3. Update the search path: queries now fan out to BOTH v1 and v2, merge results.
   (ES supports searching across multiple indices in one request natively.)

4. Background migration: gradually read docs from v1, reindex into v2, delete from v1.

5. When v1 is empty, remove it from the query fan-out.

Advantage: No dual-write complexity, no big-bang reindex. Migration is gradual.
Cost: During migration, every query hits two indices (higher latency, more fan-out).
Old documents in v1 still use the old analyzer — search relevance is inconsistent 
until migration completes. A search for "running shoes" might rank v1 docs 
(old stemming) differently from v2 docs (new stemming).
```

The key tradeoff across all three: you're choosing between complexity in the write path (B), complexity in the read path (C), or a simpler model with a brief consistency gap (A). All of them use the alias indirection to achieve zero downtime — the tenant's queries never fail, they just temporarily hit old or mixed data.

Q: What is a query in this system? How does it interact with indices?

**A:** A query is a search request from a tenant's end-user — "find me products matching 'waterproof running shoes' under $100, sorted by relevance, with a facet breakdown by brand." Your platform receives it as an API call, translates it into an ES query, and sends it to the tenant's index.

Concretely, the tenant's app calls your platform:

```
POST /v1/indices/products/search
{
  "query": "waterproof running shoes",
  "filters": { "price": { "lt": 100 } },
  "facets": ["brand"],
  "sort": "relevance",
  "size": 20
}
```

Your platform resolves `(tenant, "products")` → ES alias `acme_products`, then sends an ES query:

```
POST /acme_products/_search
{
  "query": {
    "bool": {
      "must": {
        "match": { "title": "waterproof running shoes" }
      },
      "filter": {
        "range": { "price": { "lt": 100 } }
      }
    }
  },
  "aggs": {
    "brands": { "terms": { "field": "brand" } }
  },
  "size": 20
}
```

Here's what happens inside ES when this arrives:

1. **Coordinating node** receives the request. It knows the alias `acme_products` resolves to index `acme_products_v1`, which has (say) 5 shards spread across 3 data nodes.

2. **Fan-out:** The coordinating node sends the query to all 5 shards in parallel. Each shard is a self-contained Lucene index, so each one independently:
   - Looks up `"waterproof"`, `"running"`, `"shoes"` in its inverted index → gets postings lists
   - Intersects/scores them using BM25 (relevance algorithm — considers term frequency, document length, rarity of term across corpus)
   - Applies the price range filter using doc values (column-oriented storage, no inverted index needed)
   - Computes a local top-20 by score
   - Computes a local facet count for each brand

3. **Merge:** The coordinating node collects results from all 5 shards, merges the top-20 lists into a global top-20, merges the brand facet counts, and returns the response.

The performance characteristics that matter for the problem:

- **Fan-out = shard count.** More shards per index = more parallel work per query = more cluster resources consumed. If a large tenant has 50 shards, every single query from that tenant touches 50 shards. This is why shard count is both a performance lever and a resource cost.
- **Expensive queries** are ones with complex aggregations (nested facets, cardinality counts), large result sets, or queries that match a huge number of documents (wildcard/regex queries). These consume thread pool slots and heap on every shard they touch.
- **Per-tenant isolation** isn't built into ES natively. ES doesn't know about tenants — it just processes queries against indices. If tenant A fires an expensive aggregation across 50 shards and tenant B fires a simple keyword search on 2 shards, they compete for the same thread pools and heap on any node that hosts shards from both. That's the noisy neighbor problem in this context.