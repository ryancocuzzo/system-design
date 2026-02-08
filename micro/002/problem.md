# Shopify Multi-Tenant Storefront Backend

## Context

You are building the backend platform that powers millions of independent online stores, similar to Shopify. Each store is its own business with its own products, customers, orders, and theme configuration, but all stores run on shared infrastructure managed by your team. Traffic distribution is extremely skewed — the vast majority of stores see modest traffic, but a small percentage drive enormous volume during flash sales, product drops, and seasonal events. The platform ships schema and feature changes weekly across all tenants, and merchants expect zero downtime regardless of what's happening on the platform side.

## Before You Design

Answer these before writing any architecture:

1. What is the core technical challenge this system must solve?
   A. modeling the flow and persistence solution of shared data (compute, persistence) across many tenants => avoid noisy neighbors
   B. zero downtime when anything changes (app changes, infra changes, etc)

Time elapsed; 7 mins.

2. Which of these framings best fits this problem?
   - [X] A storage and retrieval problem
   - [ ] A streaming/pipeline problem
   - [ ] A coordination/consensus problem
   - [ ] A scheduling/state machine problem
   - [ ] A distributed counting/aggregation problem

3. What is the one design decision that, if wrong, makes everything else irrelevant?

Time elapsed: 10 mins.

## Functional Requirements

- **Storefront serving:** Render product listings, product detail pages, collections, and cart for any store given its custom domain or subdomain. Each store has its own catalog, pricing, and theme.
- **Order processing:** Accept checkout submissions, validate inventory, process payment authorization, and create orders — all scoped to the originating store.
- **Catalog management:** Merchants create, update, and delete products, variants, and collections. Changes must be visible on the storefront within 5 seconds.
- **Tenant provisioning:** New stores are created on-demand and must be serving traffic within 30 seconds of creation.
- **Schema evolution:** Platform-wide schema changes (new columns, new tables, index changes) are deployed weekly without downtime for any tenant.
- **Tenant-scoped configuration:** Each store has its own settings, feature flags, and rate limits. Merchants can install third-party apps that extend their store's data model.
- **Data export:** Merchants can export their full store data (products, orders, customers) on demand for portability.

## Non-Functional Requirements

- **Traffic:** 8,000 QPS average, 40,000 QPS peak. Top 1% of stores generate 40% of total traffic.
- **Storefront read latency:** < 100ms p95 for product and collection pages.
- **Order write latency:** < 500ms p95 for checkout submission.
- **Tenant count:** 2 million active stores, growing 5% monthly.
- **Data volume:** 15 TB total across all tenants; individual stores range from 500 KB to 50 GB. Growing 500 GB/month.
- **Availability:** 99.95% per individual store — one store's outage must not affect others.
- **Migration safety:** Schema changes must complete across all tenants within 24 hours with zero read/write downtime.

## Constraints

- Stores are accessed via custom domains (e.g., `cool-shirts.com`) and platform subdomains (e.g., `cool-shirts.myplatform.com`). Routing must resolve to the correct tenant in under 5ms.
- Merchants have legal ownership of their data. Tenant data must be fully deletable within 72 hours of account closure (GDPR compliance).
- The platform cannot require per-tenant infrastructure provisioning for standard stores — operational cost must scale sub-linearly with tenant count.
- Flash sale traffic for a single store can spike 100x its normal baseline within seconds.
- Third-party app installs can add custom fields and tables scoped to a single tenant; the platform must support this without affecting other tenants.

## Concepts to Explore

These areas are relevant to this problem. Research unfamiliar ones in `context.md` before starting your design.

- **Architecture Patterns** — how you isolate tenants at the infrastructure level determines whether failures and load are contained or shared
- **Data Modeling and Storage** — how data is partitioned across tenants controls migration complexity, query routing, and per-tenant scaling
- **Reliability and Resilience** — mechanisms for preventing one tenant's traffic from degrading the experience of others on shared infrastructure
