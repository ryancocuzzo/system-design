# Design: Shopify Multi-Tenant Storefront Backend

Total time budget: 50 minutes (research + design + buffer).
If stuck on any phase beyond its time box, write what you know, mark it incomplete, and move on.

## Phase 1: Framing (5 min)

**One-line summary:** We're building a multi-tenant platform for merchants their stores.

**Core technical challenge:** 1. Failure domain isolation 2. persistence structure

**Framing:** This is mostly a failure isolation and data structure problem.

Total time epapsed: 34 mins.

**The decision that matters most:**

## Phase 2: Sub-systems and Connections (5 min)

List sub-systems, their responsibilities, and how they connect. Bulleted list, not a table.

- Client system for their customers (API with auto-scaling, persistence layer)
    - the system that clients' users interact with
    - these must have a component that pulls API version and db schema changes
    - these must also pull configuration changes (feature flags, rate limits, etc)
    - all "pulling" can be done through a listener for messaging (e.g kafka)
- Central system (API, persistence layer)
    - the system that clients use to spin up new businesses (client systems)
    - uses a publisher when making all configuration changes (adjust feature flags, etc) and to create/delete businesses
- Client system operator
    - a worker listening for messages about creating and deleting businesses
    - will provision the infrastructure and store internal and externally-visible info in respective data stores

Total time elapsed: 48 mins.

## Phase 3: Deep Dive (20 min)

Pick the 2 sub-systems where the real complexity lives. Design those. Skip the straightforward ones.

### Client system

**Approaches:**
In both cases, a centralized gateway.
A. Each client has their own "deployment" of API servers and a DB (with replicas). We route client requests to the client system's load balancer based on the url.
B. There is a central fleet of APIs and DBs that we have client infrastructure on. We shard the DBs by client id and use the hash of the client id to determine the db to route to. Then, we partition at the db level as well based on client id (to get actual data separation).
C. A hybrid model of A and B.

In both cases, a "pull model" of changes to the api or db schema. This is to avoid a huge fan-out of deployment for either. And DB schema changes must be online changes. I'm not certain how that works, but this is how to accomplish the zero-downtime component.

**Choice:** A. I'm choosing this path because it's the simplest and safest.

NOTE: In many cases, shared infrastructure should be sufficient. There are likely some mechanisms for handling temporary peak-time traffic bursts at the DB level. And then, for large-scale or clients with extremely bursty patterns, we'd want dedicated infrastructure to protect other clients on the shared infrastructure.

**Design:**

- A centralized gateway component. All client's users' requests go here. Routes client requests to the client system's load balancer based on the url.
- A load balancer to balance traffic accross API servers
- Horizontally auto-scaling set of API servers supporting all ther clients' operations (purchases, etc)
- A DB with read-only replicas
- A worker listening for DB migrations. This worker would conduct the online schema changes.
- A worker listening for API version changes that does canary-style rollouts. Similar to an ArgoCD.

Total time elapsed: 74 mins.

### Centralized system

Moving past this for time reasons.

## Phase 4: Stress Test (5 min)

Pick the 3-5 most critical NFRs and failure modes. You cannot cover all of them — prioritize.

| Concern | Type | How addressed | Confidence |
|---------|------|---------------|------------|
| | NFR / Failure | | high / medium / low |

Network partition between the client and central systems. This only means platform upgrades and changes will be delayed. I have medium confidence this is addressed. 

One client has a huge peak of usage. The api servers will auto-scale up and no other clients will be affected because their api servers are entirely separate. I have high confidence this is addressed.

NFR of tenant count of 2M, growing by 5% monthly. This is unaddressed and it appears like a non-starter for my choice of A for the client system approach. It would require a linear amount of API servers and DBs (2M).

---

Time elapsed: 86 minutes.

---

## Review

**Score: 4/10**

| Criterion | Score | Notes |
|-----------|-------|-------|
| Problem Framing | 4/10 | Right instinct on isolation, but vague; most important decision left blank |
| Architecture Quality | 3/10 | Chosen approach violates stated constraint; research not applied |
| Tradeoff Reasoning | 4/10 | Alternatives named but choice poorly justified against requirements |
| Stress Testing | 5/10 | Caught own fatal flaw — didn't pivot |
| Communication | 5/10 | Honest about gaps, but structure broke down and time budget blown |
| Operational Concerns | 4/10 | Canary rollouts and online DDL mentioned; no observability |

### Framing Assessment

**Their framing:** "Failure domain isolation" and "persistence structure" — categorized as a storage and retrieval problem.
**Correct framing:** Resource isolation and lifecycle coordination across millions of co-located tenants — a scheduling/state machine problem.
**Match:** Partially correct.

"Storage and retrieval" is the surface-level reading — it makes you think the hard problem is "which database layout?" The real challenge is: how do you *coordinate shared physical infrastructure* across 2M independent tenants with wildly different load profiles, while managing tenant lifecycle (provisioning, scaling, migrating, deleting) as a continuous process? The database schema is a tool for solving that problem, not the problem itself.

This wrong framing led directly to the design's central mistake: treating each tenant as needing its own infrastructure, rather than asking "what is the right *grouping and coordination* strategy that lets millions of tenants share infrastructure safely?"

### Phase-by-Phase Feedback

**Phase 1 (Framing):** "Failure domain isolation" is in the right neighborhood but too vague to guide architecture. The critical question is *at what granularity* do you draw failure domain boundaries — per-tenant? per-group? per-region? That question is where the real design lives, and it went unasked. "Persistence structure" is the obvious surface challenge. The most important decision was left blank — this is the single highest-value sentence in the entire exercise and it was skipped.

**Phase 2 (Sub-systems + Connections):** Three systems identified: "Client system," "Central system," "Client system operator." The decomposition roughly maps to serving plane / management plane / provisioning worker, which is reasonable. But the naming is confusing — "Client system" apparently means "the infrastructure serving one merchant's customers," which conflates the tenant with its infrastructure. The Kafka-based pull model for config and schema changes is a solid instinct. Missing: no routing/gateway layer (how does a request to `cool-shirts.com` reach the right backend?), no caching layer, no rate limiting or resource quota system.

**Phase 3 (Deep Dive):** This is where the design went off the rails. Three approaches were correctly identified (per-tenant infra, shared fleet with sharding, hybrid). But Approach A was chosen — "simplest and safest" — despite the problem explicitly stating: *"The platform cannot require per-tenant infrastructure provisioning for standard stores — operational cost must scale sub-linearly with tenant count."* The choice directly violates a stated constraint.

The NOTE acknowledging that "shared infrastructure should be sufficient" and that Approach A might not scale undermines the choice while it's being made. This suggests the right answer was known but not committed to. In a design review, hedging like this reads as: "I know this is wrong but I'm going with it anyway."

The deep dive into the chosen approach is internally consistent (gateway → LB → auto-scaling APIs → DB with replicas → migration worker → rollout worker) but it's engineering a solution to the wrong problem.

Only one sub-system got a deep dive. The centralized system was skipped entirely.

**Phase 4 (Stress Test):** The strongest moment in the design: catching that Approach A requires 2M separate API+DB deployments and calling it a "non-starter." This is exactly what stress testing is for. But the exercise ended there — the insight should have triggered a pivot to Approach B or C, even as a sketch. The network partition concern is low-priority (delayed config propagation is not a critical failure mode). Missing: flash sale 100x spike on shared infra, schema migration coordination across 2M separate DBs, GDPR deletion at scale, tenant provisioning in <30s with per-tenant infra.

### Research Assessment

The `context.md` research was partially completed:

1. **Cell-based architecture:** Answered with one sentence — "splits up the service into isolated sub-services (with their own api server(s)) with the goal of isolating failure and latency hits." This is correct at a high level but misses the key differentiator from sharding: cells are *self-contained stacks* (compute + storage + cache), not just database shards. This surface-level understanding is why cells didn't appear in the design — the research didn't go deep enough to see how cells solve the grouping problem.

2. **Multi-tenancy DB tradeoffs:** The most thorough research answer, covering shared-table, schema-per-tenant, and DB-per-tenant. Some good observations (shared-table scans, schema-per-tenant eliminates tenant_id as PK). But the claim that shared-table has "unnecessarily long search space" is wrong — with a tenant_id partition key or index, queries are scoped efficiently. The research also didn't address online schema migrations, which was the core of the question. Despite researching these tradeoffs, the design chose the most expensive option (per-tenant infra) without connecting the research to the decision.

3. **Bulkhead patterns and load shedding:** Skipped entirely. This is the research prompt most critical to noisy neighbor mitigation — the #1 concern in the problem. Skipping it likely contributed to having no rate limiting, resource quotas, or load shedding in the design.

**Anti-pattern present:** Research done on multi-tenancy tradeoffs but then ignored in architecture. The research identified the cost problems with DB-per-tenant, and then the design chose per-tenant infrastructure anyway.

---

## Ideal System Architecture

```
═══ REQUEST FLOW ═══

              custom domains / subdomains
                         │
                ┌────────▼─────────┐
                │    CDN / Edge    │  static assets, theme CSS/JS
                └────────┬─────────┘
                         │
                ┌────────▼─────────┐
                │  Routing Gateway │  domain → tenant_id → cell_id
                │  + Rate Limiter  │  per-tenant request quotas
                │                  │  lookup via cached mapping (<5ms)
                └────────┬─────────┘
                         │ routes to assigned cell
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
    ┌───────────┐  ┌───────────┐  ┌───────────┐
    │  Cell A   │  │  Cell B   │  │  Cell N   │
    │           │  │           │  │           │
    │ API Fleet │  │ API Fleet │  │ API Fleet │  (~10K-50K tenants/cell)
    │ Redis     │  │ Redis     │  │ Redis     │
    │ DB primary│  │ DB primary│  │ DB primary│  shared tables,
    │ DB replica│  │ DB replica│  │ DB replica│  partitioned by tenant_id
    └───────────┘  └───────────┘  └───────────┘
          │              │              │
          └──────────────┼──────────────┘
                         │ events (order placed, product updated, etc.)
                ┌────────▼─────────┐
                │    Event Bus     │  cross-cell coordination
                │    (Kafka)       │
                └──────────────────┘

═══ CONTROL PLANE (SUPPORTING SERVICES) ═══

    ┌───────────────────────┐
    │  Tenant Placement     │  Assigns tenants to cells at creation.
    │  Service              │  Rebalances hot tenants to larger/dedicated cells.
    │                       │  Queried by: Routing Gateway, Provisioning
    └───────────────────────┘

    ┌───────────────────────┐
    │  Migration            │  Rolls out schema changes cell-by-cell.
    │  Coordinator          │  Online DDL (gh-ost / pt-osc) per cell.
    │                       │  40-200 cells, not 2M tenants.
    └───────────────────────┘

    ┌───────────────────────┐
    │  Config / Feature     │  Tenant settings, feature flags, app configs.
    │  Flag Service         │  Propagates to cells via event bus.
    │                       │  Queried by: Merchant Admin, API Fleet
    └───────────────────────┘

    ┌───────────────────────┐
    │  Provisioning         │  New tenant = DB rows + cache warm + DNS.
    │  Service              │  No infra provisioning. Completes in <30s.
    └───────────────────────┘

    ┌───────────────────────┐
    │  Data Lifecycle       │  GDPR deletion (full tenant purge in <72h).
    │  Service              │  Data export on demand.
    └───────────────────────┘

═══ PER-TENANT EXTENSION MODEL ═══

    Third-party apps add custom fields/tables scoped to tenant_id
    within the cell's DB. Schema: `app_{app_id}_tenant_{tenant_id}`.
    Isolated by naming convention + row-level access policies.
    Migrations for custom schemas are app-scoped, not platform-wide.
```

### Key Design Decisions in the Ideal Architecture

**Cell-based isolation:** Tenants are grouped into cells of ~10K-50K. Each cell is a self-contained stack (API servers, cache, database). The cell is the blast radius boundary — if Cell A's DB goes down, only Cell A's tenants are affected. This gives you per-group isolation without per-tenant cost.

**Shared tables within cells:** Inside each cell, all tenants share the same tables with `tenant_id` as the partition key. This makes provisioning instant (INSERT a row, not spin up a database), migrations tractable (one DDL per cell, not per tenant), and queries efficient (partition pruning on tenant_id).

**Tenant placement as a first-class service:** A placement service decides which cell each tenant lives in, using heuristics like current cell load, tenant size, and traffic history. Hot tenants (the top 1% driving 40% of traffic) can be moved to dedicated or oversized cells. This is how you handle 100x flash sale spikes — the placement service preemptively isolates known-hot tenants.

**Cell-by-cell migration:** Schema changes roll out to one cell at a time using online DDL tools (gh-ost, pt-online-schema-change). If a migration causes problems in Cell A, you stop. 40-200 sequential migrations, each on a manageable database, completing within 24 hours total. Not 2M simultaneous migrations.

**Per-tenant rate limiting at the gateway:** Before a request even reaches a cell, the gateway enforces per-tenant rate limits. This is the first line of noisy-neighbor defense. Inside the cell, bulkheads (connection pool limits per tenant, query timeout enforcement) provide the second line.

---

## Key Differences

| Aspect | Their Design | Ideal Design | Why It Matters |
|--------|--------------|--------------|----------------|
| Isolation granularity | Per-tenant (2M separate deployments) | Per-cell (~40-200 cells, each serving ~10K-50K tenants) | Per-tenant infra at 2M tenants is operationally impossible and violates the sub-linear cost constraint |
| Database strategy | DB-per-tenant | Shared tables with tenant_id partition key within each cell | Shared tables make provisioning instant, migrations tractable, and cost sub-linear |
| Routing | "Routes to client system's load balancer" (no design) | Cached domain→tenant→cell lookup at gateway (<5ms) | Domain resolution is on the critical path for every request; without it, nothing works |
| Migration approach | Worker listening for DDL changes (per-tenant) | Cell-by-cell online DDL with rollback capability | 2M separate migrations is infeasible in 24h; 40-200 cell migrations is routine |
| Noisy neighbor defense | Solved by full isolation (brute force) | Per-tenant rate limits at gateway + cell-level bulkheads + hot-tenant placement | Brute force works mechanically but costs 1000x more than necessary |
| Provisioning | Worker provisions infrastructure per tenant | DB INSERT + cache warm + DNS entry (~seconds) | Per-tenant infra provisioning cannot hit the 30-second requirement at scale |
| Flash sale handling | Auto-scale per-tenant API servers | Pre-position hot tenants in dedicated cells + CDN + rate limiting | Auto-scaling per-tenant works but requires 2M independent scaling decisions |
| GDPR / data deletion | Not addressed | Data Lifecycle Service: DELETE WHERE tenant_id = X across all tables in the cell | Legal requirement; must be designed in, not bolted on |

---

## Core Mental Model

**How they conceptualized the problem:** Each tenant is a miniature deployment that needs its own isolated infrastructure. The system is 2 million small systems.

**How they should have conceptualized it:** Tenants are logical boundaries within shared physical infrastructure. The question is not "how do I give each tenant its own stuff?" but "what is the right *grouping granularity* that balances isolation against operational cost?" The answer is the **cell** — a self-contained stack serving thousands of tenants, where the cell is the failure domain boundary and the unit of operational management (deployment, migration, scaling).

This reframe changes everything:
- Provisioning goes from "spin up infrastructure" to "insert a row and warm a cache"
- Migration goes from "coordinate 2M independent changes" to "roll through 40-200 cells sequentially"
- Noisy neighbor defense goes from "irrelevant because everyone is isolated" to "rate limit at the gateway, bulkhead within the cell, and move hot tenants to bigger cells"
- Cost goes from O(n) in tenants to O(n) in cells, where cells grow logarithmically with tenant count

The self-identified "non-starter" in Phase 4 was the design trying to tell you it needed this reframe. The instinct to catch it was correct — the next step is to listen to it.
