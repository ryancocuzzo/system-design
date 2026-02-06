# Design

One-line headline: We're designing a network of servers such that our customers (owners of services) can point their users at these servers (with some url we give them), we apply rules to those requests and then either reject the requests or pass them along to their intended destination (the url of the customer's deployed service).

The core architectural challenges appear to be
- the subdomain url generation (to be able to give the customer a url to point their users to) and mapping that to a customer deployment url
- low latency rule matching (a full db lookup will be far too expensive for a middleware service like this. this is enforced in the p50 < 1ms non-functional requirement)
- ensuring all requests hit the closest edge service to that user
- 5 9s of availability
- Rule (change) propagation across edge nodes

A core note to this architecture is the "sub-domain". In `https://xyz.google.com`, `xyz` is the sub-domain. That information is what we provide to the customer for their clients to use as the entry point to their (and, by definition, this, system). All rules and enforcement are done at the sub-domain-level.

Let's break it into sub-systems:

1. The routing mechanism
- Requirement: All user requests should be hitting their nearest edge node (regardless of their subdomain within our system)
- Mechanism:
    - Option A: A really thin, low-latency centralized proxy service
        - user request hits this service, it manages a lookup table for the closest edge node to the user's IP (in the request) and proxies there
        - the lookup table is maintained by a running loop process that fetches a centralized lookup table (e.g a centralized redis) and refreshing, or the server can listen to a messaging stream for messages from a central API to refresh the table (or for changes in the table)
    - Option B: Some kind of DNS configuration that ensures the nearest edge node is the one that the request resolves to
- For high-availability support, this component could do frequent health checks of edge nodes and, if they're down, adjust the routing table to the next closest node.

2. Edge node
- This must support rule constraint satisfaction for 200M rules. 
Assuming each rule consumes 1kb of storage, that makes 200GB of storage required to support the rule set. This won't fit in-memory. This edge node will have an in-memory cache of recently used rules. When that cache doesn't have the required ruleset, default to a centralized distributed persistence layer to evaluate rules.
- For the constraint satisfaction process, a naive way that would work would be to fetch the supported limited rules (HTTP method, headers, etc), their window, their required action, and the current usage counts for that customer all from persistence.
    - In all cases, increment the associated usage counts for the request
    - In a case where a user's usage exceeds a limit rule threshold, reject the request with a 429.
    - Otherwise, we should be proxying the request onto its destination server.
        - We can get that destination server with another lookup into persistence using the subdomain provided. This will give us the customer's desired endpoint and we can proxy to there.
            - This can be done with a centralized NoSQL DB which has the full set of 200M customer comains, but with an in-memory cache.
        - This should proxy in a way that does not couple to the users' service, to avoid needing to handle erroneous or harmful service behaviors.
- We apply the same general process for global firewall rules. We have the current global rules in persistence and evaluate them per each request (naively can loop over each and see if the request violates them).

3. The centralized persistence layer
- Referencing the logic above, this is a NoSQL DB that maps sub-domains to their rules, windows, required actions and usage counts. Naively, it can store its rules as such:
    - {SUBDOMAIN_ID}/
        - CustomerEndpoint: https://..
        - Rules: [ .. ]
        - WindowPolicy: <PolicyType>
        - FirewallRules: [ .. ]
        - ClientUsageOverWIindow: { id1: 99, id2: ...}
- There is an API server that watches this DB and, when there are changes, it publishes a message to a message stream with the change for the edge nodes to update.

4. The API Server, for the customers to interact with their configuration
- This is the interaction point for the user. Supports
 - POST /subdomains => creates a new subdomain
    - Updates the centralized perstance with
        - {SUBDOMAIN_ID}/
            - CustomerEndpoint: https://..
            - Rules: [ .. ]
            - WindowPolicy: <PolicyType>
            - FirewallRules: [ .. ]
            - ClientUsageOverWIindow: { id1: 99, id2: ...}
    - Notifies the edge nodes of the new subdomain, so they can update their mappings
    - Returns http://{SUBDOMAIN_ID}.THIS_SERVICE_URL as the public endpoint for the customer's users.
 - POST /subdomains/:id/rules => create a new rule for their subdomain
    - Updates the centralized persistence layer
 - PATCH /subdomains/:id/rules/:id => edit a rule
    - Same behavior as above, but for modifying the rule
    - Publishes a message for edge nodes to update that cached rule, if they have it
 - GET /subdomains/:id/rules => fetches their rules

## Addressing non-functional requirements

1. Proxies requests: All the services mentioned are stateless, with the exception of their disposable caches, and can be horizontally scaled. The system's backpressure is only its lookup speeds, everything after that is proxied away.
2. Edge nodes: This system uses a pull model (listening for messages) for edge nodes staying in-sync with the data source of truth, and the routing component's table can very safely handle a few hundred entries. This will scale well.
3. Customer domains and Active Rate Limit Rules: The central source of truth will be a distributed NoSQL DB with smart routing and several nodes for reading. That should handle the load.
4. Unique client IDs tracked: This could be a bottleneck. The current storage model also will not scale well for rolling-basis usage limits. It would need a time-based refresh because it's not storing each timestamp of each user's requests. I'm not sure of a performant way to do that.
5. Rule eval latency: This shuold be hit, assuming a high-quality im-memory cache for each edge node's rules. The rule matching pattern described here is naive, however. That may tax the latency.
6. Firewall rule propogation and rule config propogation: Very safely hit through the "pull model" that the edge nodes are using. This update shuold be near instantaneous.
7. Availability: This system supports HA through the routing table. No other failover procedure is addressed in this design.

Time elapsed: 105 minutes

---

## Review

**Score: 3/10**

| Criterion | Score | Notes |
|-----------|-------|-------|
| Problem Understanding | 4/10 | Misidentified the core challenge; focused on subdomain routing instead of distributed counting |
| Architecture Quality | 3/10 | Centralized persistence on the hot path breaks at this scale; no viable counter strategy |
| Tradeoff Reasoning | 3/10 | One routing tradeoff explored; nothing on the hard decisions (consistency, counting, partitions) |
| Failure Handling | 2/10 | Explicitly acknowledged as not addressed; no partition strategy despite it being a stated constraint |
| Operational Concerns | 2/10 | No monitoring, deployment, observability, or analytics pipeline |
| Communication | 5/10 | Structured with numbered sub-systems and self-aware NFR section, but subdomain framing adds confusion |

### Feedback

The design treats this as a proxying and routing problem with rate limiting bolted on as a DB lookup. The actual problem is a **distributed counting problem under consistency and latency constraints**. Every major architectural decision flows from that framing, and missing it means the design doesn't address any of the hard parts: how counters stay accurate across 300+ nodes, what happens during partitions, how to handle hot keys, or how to keep rule evaluation under 1ms. The centralized NoSQL DB as the counter store is a non-starter at 30M rps. The 105-minute time also signals the framing problem — time was spent on subdomain mechanics instead of the core challenge.

---

## Detailed Critique

### On Structure and Clarity of Thought

**What worked**: The numbered sub-system decomposition (routing, edge node, persistence, API) is a reasonable starting scaffold. The "Addressing non-functional requirements" section at the end shows good self-awareness — you correctly flagged that client ID tracking and rolling windows were unsolved. That honesty is valuable in an interview.

**What didn't work**: The one-line headline and the architectural challenge list both center on **subdomain URL generation and routing** — problems the question doesn't actually ask about. The problem states customers already have domains proxied through the edge network. This misframing consumed significant design time (routing mechanism, subdomain creation API, subdomain-to-customer mapping) on solved problems while leaving the actual hard problems — distributed counting, consistency, partitions, hot keys — unaddressed.

**How the structure choice affected reasoning**: By leading with routing as sub-system #1, you spent your sharpest thinking on the least important component. The edge node (sub-system #2) is where all the real complexity lives, but it got a surface-level treatment: "fetch rules from persistence, increment counters, proxy or reject." That single sentence is where the entire design needed to go deep.

### Specific Points Examined

**1. "the subdomain url generation... and mapping that to a customer deployment url"**

This is not a requirement of the problem. The problem says "proxies HTTP traffic for 25 million customer domains." The routing is already handled — customers have pointed their DNS at your edge network. Spending design time on subdomain generation, `POST /subdomains`, and subdomain-to-endpoint mapping is solving a problem that doesn't exist here. In an interview, this would signal that the candidate didn't read the problem carefully.

**2. "Option A: A really thin, low-latency centralized proxy service"**

A centralized proxy in front of a globally distributed edge network defeats the purpose of having edge nodes. All 30M rps would funnel through a single point, adding latency and creating a single point of failure. This is the opposite of what an edge architecture provides. The correct answer is Anycast DNS (your Option B), which is how every real CDN/edge network works — but you listed it as an option rather than recognizing it as the obvious and only viable choice.

**3. "Assuming each rule consumes 1kb of storage, that makes 200GB of storage required to support the rule set. This won't fit in-memory."**

Good instinct to do the math. But the conclusion needs refining. You're correct that 200GB won't fit in 64GB, and you're correct that every node must be capable of serving all 25M domains (Anycast means any user can hit any node for any domain). However, traffic follows a power-law distribution — at any given moment, a node's **active working set** is far smaller than the full 25M domains. The design should be: hot rules (active working set) in-memory with LRU eviction, cold rules loaded on-demand from a **regional cache** (not the centralized DB) on first request for a cold domain. A cold-rule cache miss costing 1-5ms from a regional store is a tolerable rarity. The real problem in your design wasn't the cache miss concept — it was falling back to a centralized DB and coupling it with synchronous counter reads on every request.

**4. "fetch the supported limited rules... their window, their required action, and the current usage counts for that customer all from persistence"**

This is the critical flaw. "Fetch current usage counts from persistence" means making a **synchronous remote database call on every single request**. At 30M rps globally, with a p50 < 1ms latency budget, this is physically impossible. A round trip to even a regional database is 1-10ms. To a centralized one, 50-200ms. The entire design depends on counters being **local to the edge node** with background synchronization. This is the core architectural insight the problem is testing.

**5. "In all cases, increment the associated usage counts for the request"**

Where? In the centralized NoSQL DB? That's 30M writes/second to a single counter store. No NoSQL database handles that without extreme sharding, and even then the read-after-write consistency you'd need makes this impractical. The increment must happen locally, in-memory, on the edge node.

**6. "This system uses a pull model (listening for messages) for edge nodes staying in-sync"**

Pull model for rule configuration updates is reasonable. But the design conflates rule updates (infrequent, can tolerate 30s delay) with counter synchronization (must happen continuously, is the core challenge). The pull model is never described for counters, because the design doesn't have a counter synchronization strategy at all.

**7. "This could be a bottleneck. The current storage model also will not scale well for rolling-basis usage limits."**

Credit for identifying this, but in an interview, identifying a fatal flaw in your own design without proposing a fix is worse than not mentioning it. It signals you know the approach is broken but continued with it anyway. The correct response when you realize your counter strategy doesn't work is to **stop and redesign it**, not to note it as a future concern.

**8. "This system supports HA through the routing table. No other failover procedure is addressed in this design."**

The problem states network partitions happen **multiple times per week** and edge nodes **must continue enforcing limits during partitions**. This isn't an edge case — it's a primary design constraint. Not addressing it means the design doesn't satisfy the problem requirements. During a partition, your centralized DB becomes unreachable from some edge nodes, and with no local counter state, those nodes can't enforce any limits at all.

**9. No mention of: hot keys, asymmetric attacks, clock skew, sliding windows, analytics pipeline, counter accuracy guarantees**

These are all explicitly stated in the requirements or constraints. The absence of any discussion of them means most of the problem surface was left untouched.

**10. "105 minutes"**

Nearly 3x the 30-40 minute target. This is diagnostic — when a design takes this long, it usually means the candidate is stuck in details that don't matter (subdomain mechanics, API endpoint definitions) rather than attacking the core problem. In an interview, you'd have been stopped at 40 minutes with most of the hard problems unaddressed.

---

## Ideal System Architecture

```
═══════════════════════════════════════════════════════════════════════
                         CONTROL PLANE (off hot path)
═══════════════════════════════════════════════════════════════════════

  Customer ───▶ [ Config API ] ───writes───▶ [ Rule Store (DB) ]
                                                     │
                                              compiles rules
                                                     │
                                                     ▼
                                          [ Rule Compiler ]
                                          (radix trie, regex,
                                           bloom filters)
                                                     │
                                              pushes compiled
                                              rule bundles
                                                     │
                                                     ▼
                                          [ Distribution Bus ]
                                          (pub/sub per region)
                                                     │
                          ┌──────────────────────────┼────────────────┐
                          ▼                          ▼                ▼
                    Region A edges           Region B edges     Region N edges


═══════════════════════════════════════════════════════════════════════
                    EDGE NODE (the hot path — per node)
═══════════════════════════════════════════════════════════════════════

  Inbound        ┌──────────────────────────────────────────────┐
  Request ──────▶│  1. Firewall Check (bloom filter + hash set) │
  (Anycast)      │     BLOCK ──▶ 403 response                  │
                 │     PASS  ──▶                                │
                 │                                              │
                 │  2. Rule Match (compiled radix trie)         │
                 │     Extract: rule_id + client_key            │
                 │     (IP, API-key header, custom header)      │
                 │                                              │
                 │  3. Counter Check (in-memory)                │
                 │     local_count + last_known_global ──▶      │
                 │       OVER BUDGET ──▶ 429 response           │
                 │       UNDER BUDGET ──▶                       │
                 │                                              │
                 │  4. Proxy to Origin                          │
                 └──────────────────────────────────────────────┘
                           │
                    (async, not on hot path)
                           │
                    ┌──────▼──────┐
                    │ Counter     │───── every 1-5s ────▶ Regional
                    │ Sync Agent  │◀── updated budgets ── Aggregator
                    └─────────────┘
                           │
                    emit snapshots
                           │
                           ▼
                    [ Analytics Stream ] ──▶ Kafka ──▶ TSDB


═══════════════════════════════════════════════════════════════════════
               REGIONAL COUNTER AGGREGATION (per region, ~40)
═══════════════════════════════════════════════════════════════════════

  Edge Node A ──┐                          ┌── Region B Aggregator
  Edge Node B ──┼── push deltas ──▶ [ Regional  ] ◀── gossip ──▶ │
  Edge Node C ──┘                  [ Aggregator ]               ...
                                        │
                                  maintains per-key:
                                  - global_estimate (sum of all regions)
                                  - budget_allocation per edge node
                                  - hot key detection (rate threshold)
                                        │
                              ┌─────────┴──────────┐
                              ▼                    ▼
                        Normal keys:          Hot keys:
                        budget = limit/N      tighter sync interval
                        sync every 5s         sync every 100-500ms
                                              or coordinate via
                                              aggregator before allow


═══════════════════════════════════════════════════════════════════════
                    PARTITION BEHAVIOR
═══════════════════════════════════════════════════════════════════════

  [Normal]     edge ◀──sync──▶ regional_agg ◀──gossip──▶ other regions
               budget refreshed continuously, ~5% accuracy

  [Intra-region partition]
               edge loses contact with regional_agg
               → freeze current budget, enforce locally
               → fail-open if budget exhausted and no refresh in 30s
               → resume sync when connectivity restores

  [Inter-region partition]
               regional_agg loses contact with peers
               → enforce with region-local counts only
               → over-admission bounded by: limit × (partitioned_regions / total)
               → acceptable under 5% accuracy target for normal traffic
               → detected by analytics; alert if drift > threshold
```

### Key Design Details

**Counter Budget Algorithm (Token Partition)**:
Each edge node is assigned a **budget** — a fraction of the global rate limit. For a rule "100 req/min per IP," if traffic for that IP hits 5 nodes, each starts with a budget of 20. The regional aggregator rebalances budgets based on observed traffic distribution. Nodes that see more traffic get larger budgets. This eliminates synchronous coordination on the hot path.

**Sliding Window via Approximate Counting**:
Use a combination of fixed windows with interpolation (the "sliding window log" approximation). Each counter tracks the current and previous window counts. The effective count is: `prev_window × overlap_fraction + current_window`. This requires only two integers per counter — no per-request timestamps.

**Memory Management (64GB constraint)**:
- Hot counters: exact in-memory hash map (top ~10M keys per node)
- Warm counters: Count-Min Sketch (~100MB supports billions of keys with <1% error)
- Cold rules: loaded from regional cache on first hit, LRU eviction
- Firewall rules: bloom filter for fast negative lookup (a few MB), exact hash set for confirmed matches

**Hot Key Handling**:
When a local counter exceeds a configurable rate (e.g., >1000 req/s on a single node), the sync agent switches to a faster sync interval and the regional aggregator may designate that key as "coordinated" — meaning nodes must check with the aggregator before allowing requests above a tighter local threshold.

**Asymmetric Attack Detection**:
Regional aggregators detect when the sum of per-node counts for a key significantly exceeds any single node's count. This pattern — low per-node, high globally — triggers a coordinated response: tighten all node budgets for that key and increase sync frequency.

---

## Key Differences

| Aspect | Their Design | Ideal Design | Why It Matters |
|--------|--------------|--------------|----------------|
| Core framing | Proxying/routing problem with rate limits as a DB lookup | Distributed counting problem with proxying as the easy part | Every architectural decision flows from this framing. Getting it wrong means none of the hard problems get addressed. |
| Counter location | Centralized NoSQL DB, synchronous read/write per request | In-memory on each edge node, async sync to regional aggregator | A remote DB call per request at 30M rps breaks p50 < 1ms. Local counters are the only viable approach. |
| Consistency model | Implicit strong consistency (always read from DB) | Explicit eventual consistency with bounded error via budget partitioning | Strong consistency requires synchronous coordination, which is incompatible with the latency requirement. The 5% accuracy target is the problem telling you to use eventual consistency. |
| Partition handling | Not addressed | Budget freeze + fail-open with bounded over-admission | Partitions happen weekly per the problem. Without a strategy, the system either blocks all traffic (fail-closed) or allows unlimited traffic (no enforcement). |
| Rule evaluation | Naive loop over rules fetched from DB | Compiled radix trie + bloom filters, entirely in-memory | 200M rules with p50 < 1ms means you need O(log n) or O(1) matching, not O(n) scanning. Compilation happens offline; hot path is pure memory lookup. |
| Hot key strategy | Not addressed | Adaptive sync frequency + aggregator-coordinated budgets | 500K rps on one key melts any fixed-interval sync. Hot key detection and tighter coordination is essential for accuracy under skewed load. |
| Memory management | "Won't fit in memory" → fall back to DB | Sharded by traffic locality + Count-Min Sketch for long tail + LRU eviction | Each node only needs rules for its traffic. Approximate data structures handle the long tail within 64GB. |
| Firewall propagation | Same pull model as rules (vague) | Dedicated high-priority push channel with bloom filter on node | Firewall rules must propagate in <10s and be evaluated before rate limits. They deserve a separate, faster distribution path. |
| Analytics | Not addressed | Edge emits counter snapshots → Kafka → TSDB → customer dashboard | Requirement #5 (rate limit analytics) is a stated functional requirement, not optional. |
| Attack resilience | Not addressed | Aggregator detects cross-node attack patterns, tightens budgets globally | The asymmetric attack constraint exists specifically to test whether candidates think about adversarial workloads. |

---

## Core Mental Model

**How you conceptualized the problem**: A request-routing system — "give customers a URL, route to the right edge node, look up rules, check counts, proxy or reject." The rate limiter is a middleware that queries a database.

**How you should have conceptualized it**: A **distributed counting system under adversarial conditions** — "how do 300+ independent nodes agree on how many times something has happened, fast enough to make per-request decisions, accurately enough to enforce limits, and resiliently enough to survive network partitions?" The proxy is trivial; the counter is the system.

**Why this reframe changes everything**: When you see it as a routing problem, you naturally reach for centralized storage (the DB knows the truth) and treat the edge as a thin client. When you see it as a counting problem, you immediately confront the CAP tradeoffs: you can't have strong consistency and sub-millisecond latency and partition tolerance simultaneously. That forces you toward **local-first counting with budget partitioning** — which is the insight that unlocks the entire design. Every other decision (sync intervals, hot key detection, partition behavior, memory management) follows naturally from "counters must be local, coordination must be async."

The 5% accuracy target in the NFRs was the problem **telling you** this. It's saying: "we accept bounded inaccuracy in exchange for latency and availability." That's the tradeoff the design needed to build around.