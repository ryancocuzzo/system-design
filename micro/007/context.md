# Context — Instagram Photo Feed Service

## Guided Research (5 min)

Research these before starting your design. For each, write 1-2 sentences on what it is and when you'd use it. If time runs out, move on.

1. **Fan-out patterns** — What is the difference between fan-out on write and fan-out on read for social feeds, and why does follower count asymmetry make a uniform approach impractical?

Fan-out on write puts the new post on the feed of the users who follow them when the post is made (in the write path).
Fan-out on read grabs the relevent posts, and kind of builds the feed, at query time (on the read path).

The follower count skew is important because fan-out on write is sensible for users with few followers (simply putting a record of the post on their followers' feed inbox) but not for users with huge follower counts because this would be millions of writes at once. And, fan-out on read is not a silver bullet. Fan-out on read would mean running an expensive query constantly, and reads dominate social app activity. And caching will not fix this because the cache will be invalidated constantly (new posts, etc).

Time elapsed: 36 mins.

2. **CQRS** — How does separating the read model from the write model help when reads outnumber writes by 1000:1 and require different data shapes?

It lets us optimize independently for the two different jobs of each. We need to shape the data differently on our two paths to optimize for mostly our user constraints. And the shape of the data then follows that decision (the de-coupling of posts and inbox items means now have different models on read vs write paths).

Time elapsed: 45 mins.

## During Design
[Questions that arise while designing. Note which phase triggered them.]

Q: These are the options for the second design question:

> - **A.** Per-user feed in a wide-column store (Cassandra) — each user's feed is a pre-materialized row of post IDs sorted by rank score
> - **B.** In-memory sorted sets (Redis) backed by persistent storage — hot feeds served from memory, cold feeds reconstructed on demand
> - **C.** Post timeline in a relational database with read replicas — query followed accounts' posts at read time using indexed lookups

Here are my initial thoughts:

For option A, I haven't worked with Cassandra, so I'm not sure what value it brings here would be or what a "wide column store" is. 
For option B, sorting sounds good because of the ranking aspect, but I would expect feeds to be always cold due to how often they'd need to change (when any followed user posts).
Option C sounds good, but doesn't really address the scoring aspect itself.

Could you give me feedback on those thoughts?

### Feedback

**Option A — Cassandra / wide-column store.** A wide-column store is a database where each row has a key and can contain a very large number of columns, and those columns are sorted. Think of it like a distributed sorted map-of-maps. For a feed, the row key is the user ID and each column is a post reference sorted by timestamp or rank score. The value Cassandra brings here is: (1) writes are extremely fast (append-only LSM tree), which matters if you're fan-out-on-write pushing millions of post references, (2) reading a feed is a single sequential range scan on one partition — no joins, and (3) it's designed to scale horizontally by adding nodes with no single bottleneck. The tradeoff is that it has very limited query flexibility — you get your data back in the sort order you defined at write time, and not much else. That rigidity is actually fine here because a feed is always read the same way: "give me the top N posts for user X."

**Option B — the "always cold" intuition is wrong, and it's worth understanding why.** "Hot" and "cold" don't refer to how often the feed *changes* — they refer to how often the feed is *read*. A user who opens Instagram 10 times a day has a hot feed. A user who hasn't opened the app in 3 months has a cold feed. Redis sorted sets handle frequent updates just fine — inserting into a sorted set is O(log n), so pushing a new post reference when someone they follow posts is cheap. The insight behind Option B is that you only need to keep ~500M DAU feeds in memory, not all 2B+ accounts. Users who go inactive get evicted; when they come back, you reconstruct their feed from persistent storage. This is a classic tiered-caching approach.

**Option C — the scoring gap is real, but the bigger issue is cost at scale.** 5B feed reads/day means ~58K feed reads/second average, with peaks much higher. Each feed read would require querying up to 5,000 followed accounts' recent posts, merging, and ranking. Even with read replicas and good indexes, that's an enormous amount of fan-out-on-read work for every single feed load. Relational databases are great at flexible queries, but that flexibility comes at a per-query cost that multiplied by 58K/sec becomes untenable. This option is essentially choosing fan-out on read at the storage layer regardless of what you picked in Decision 1.

**Key question:** Given what you chose in Decision 1 for the feed generation strategy, which storage model complements that choice? If you're pre-computing feeds (fan-out on write), you need fast writes and fast sequential reads per user — that points toward A or B. If you're computing at read time, you need flexible queries — that points toward C but with the scale concerns above.

Q: I still don't understand how Option B would work in practice. It puts a cache in place. So, it's just a cache with pull-through to a relational DB if the query missed?

### Feedback

Not quite — it's more than a simple cache-aside pattern. The backing store doesn't have to be relational, and the Redis layer isn't just a passive cache that mirrors what's in the DB. Here's how it works concretely:

**Write path:** When a post is created and the fan-out service decides user X should see it, it writes the post reference (post ID + rank score) into two places: (1) a Redis sorted set keyed by user X's ID, and (2) a persistent store (could be Cassandra, DynamoDB, or even a relational DB — the backing store choice is secondary). The Redis sorted set keeps the feed ordered by score, and you cap it at, say, the most recent 500 posts via `ZREMRANGEBYRANK` so memory stays bounded.

**Read path:** When user X opens their feed, you call `ZREVRANGE` on their sorted set — one O(log n + k) operation returns the top k post IDs already sorted by rank. You then hydrate those IDs into full post objects (from a separate post cache or DB). This is sub-millisecond for the feed fetch itself.

**The "cold feed" reconstruction:** If user X hasn't opened the app in months, their sorted set was evicted from Redis (LRU or TTL). On their next feed load, you detect the miss, query the persistent store for their feed data, rebuild the sorted set in Redis, and serve from there. This is slower — maybe 50-100ms instead of 1ms — but it only happens once per returning inactive user.

So Redis here is the **primary serving layer**, not a cache. It's the source of truth for feed ordering for active users. The persistent store is the durability backstop, not the thing you're querying on every read. That's the key distinction from a traditional cache-aside pattern where the DB is the source of truth and the cache just saves you a trip.