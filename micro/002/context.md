# Context — Shopify Multi-Tenant Storefront Backend

## Guided Research (10 min)

Research these before starting your design. For each, write 1-2 sentences on what it is and when you'd use it. If time runs out, move on — don't let research eat your design time.

1. **Architecture Patterns** — What is cell-based architecture, and how does it differ from simple sharding as a strategy for isolating groups of tenants?

Cell-based architecture splits up the service into isolated sub-services (with their own api server(s)) with the goal of isolating failure and latency hits.

2. **Data Modeling and Storage** — What are the tradeoffs between shared-table multi-tenancy (tenant ID column), schema-per-tenant, and database-per-tenant, particularly around partition key design, hot partitions, and online schema migrations?

Shared-table mutli-tenancy is the simplest, but uses the most unneccessary compute. Each tenant only cares about their data, and so all queries will be searching through an unncessessarily long search space. In some ways, it's cheap because it uses the same DB. The expensive aspect is due to its usage. Tenant ID would have to be the primary key. Schema migrations are the easiest.

Schema-per-tenant gets rid of the over-scanning problem, but still query surges between clients will still hit the same DB instance. It's cheaper than multiple instances thuogh. You no longer need a tenant id as the primary key, which will help performance (you can use the next most important key like customer id).

DB-per-tenant is the most isolated but most expensive because you're paying for a linear amount of DBs as customers. 

There is a lot more to research, and I didn't address the core of the question(s).

Total time elapsed: 28 mins.

3. **Reliability and Resilience** — How do bulkhead patterns and load shedding work together to prevent a single overloaded tenant from consuming shared resources?

Moving on for time reasons.

## During Design
[Questions that arise while designing. Note which phase triggered them.]
