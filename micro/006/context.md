# Context — URL Shortener Service

## Guided Research (5 min)

Research these before starting your design. For each, write 1-2 sentences on what it is and when you'd use it. If time runs out, move on.

1. **Hot partitions** — What causes hot partitions in distributed key-value stores, and what strategies exist to distribute load when a small number of keys receive the majority of traffic?

Hot partitions are caused when a small subset of keys get a lot of traffic.

To distribute this, you can store the key on multiple different servers and do a routing calculation (random, or hash the user id) to pick which key-value store server gets picked for interacting with that "hot key".

2. **Cache stampede protection** — What is a cache stampede (thundering herd), and how do techniques like request coalescing, locking, and probabilistic early expiration prevent it?

A cache stampede is when there is a cache miss on a hot key (e.g a TTL expires) and a ton of traffic comes in at the same time, all of which would then hit the DB and cause a massive DB spike.

Request coalescing means, if there is another request who is fetching the same data, to wait on and use that data result by "attaching to the same in-flight promise/future".

Locking is used in distributed systems by acquiring a lock on the key from the key-val store and others waiting for the data need to either wait & retry later or serve stale data.

probabalistic early expiration increases the odds of refresh as the TTL approaches 0, which will make it so it is increasingly likely over time that, by the time the TTL expires, the cache key has already been refreshed.

Time elapsed: 28 minutes.


## During Design
[Questions that arise while designing. Note which phase triggered them.]

Q: Design question 2 asks:

> - **B.** DynamoDB or Cassandra — purpose-built for high-throughput point lookups, native horizontal scaling, limited query flexibility
> - **C.** Redis Cluster as primary store with async disk persistence — sub-millisecond reads, but cost and durability implications at ~18B keys

A Redis Cluster shared across API servers feels like it would add a meaningful latency tax (it's not in-memory). Otherwise, DynamoDB surely is easy because it's a managed service. Those don't feel like apples-to-apples options.

**A:** Two clarifications: (1) Redis *is* in-memory — the data lives in RAM. The latency you're thinking of is the network round-trip to the Redis cluster (typically sub-ms when co-located), not disk. (2) You're right that B and C aren't apples-to-apples. The intent is: B = durable primary store (disk-backed, managed, built for this scale); C = use a cache-tier store *as* your primary. Option C's real problems at 18B keys are **cost** (RAM is ~10–50× more expensive than disk per GB) and **durability** (AOF/RDB are not the same as DynamoDB's replicated writes). So C is a plausible-but-wrong choice: it optimizes for latency while ignoring that at this scale you'd need a massive Redis fleet and still have weaker durability. The "easy because managed" point for DynamoDB is valid — ElastiCache is also managed, but the cost curve at 18B keys makes it a non-starter as primary storage.

Time elapsed: 54 mins.

Q: I don't understand the following portion of the response above.

>  and **durability** (AOF/RDB are not the same as DynamoDB's replicated writes). So C is a plausible-but-wrong choice: it optimizes for latency while ignoring that at this scale you'd need a massive Redis fleet and still have weaker durability.

What is AOF? Are AOF/RDB's writes not sufficient? And it's not clear to me where the optimization is for latency or durability in any option.

**A:** **AOF** = Append-Only File. Redis logs each write to a file on disk. **RDB** = Redis Database Backup — periodic snapshots. Both persist data, but the model differs from DynamoDB: Redis typically uses *async* replication (replica lags behind primary) and *async* or *periodic* disk sync (fsync every second, or every write with a latency hit). If the primary dies before a replica catches up or before the next AOF sync, you lose recent writes. DynamoDB replicates *synchronously* — it doesn't acknowledge the write until it's durable on multiple nodes. So AOF/RDB can be sufficient for many workloads, but for a URL shortener where "lose a few seconds of creates" might mean thousands of lost mappings, the weaker guarantee matters. **Where the tradeoff is:** Option C (Redis) is optimized for *latency* — reads come from RAM, sub-ms. Option B (DynamoDB) is optimized for *durability and scale* — reads are slower (1–5ms) but the write path is built for durable, replicated storage. The "optimization" is what each system is designed for: Redis = cache-tier speed; DynamoDB = durable primary store.