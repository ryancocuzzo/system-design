# Instagram Photo Feed Service

## Context
You're designing the core feed infrastructure for a photo-sharing platform at Instagram's scale. Users post photos, follow other accounts, and scroll a personalized home feed. The fundamental tension is that a tiny fraction of accounts (celebrities, brands) generate a disproportionate share of the content that must appear in hundreds of millions of feeds, while the vast majority of users post infrequently to small audiences.

## Functional Requirements
1. Users upload photos with captions; photos appear in followers' feeds within seconds
2. Users follow/unfollow other accounts and see a home feed of posts from accounts they follow
3. Feed is ranked by relevance signals (recency, engagement, relationship strength), not purely chronological
4. Users can like and comment on posts; engagement counts are visible on each post

## Non-Functional Requirements
1. 500M DAU; average user loads their feed 10 times/day (5B feed reads/day)
2. Feed load latency < 200ms at p99
3. New post visible in followers' feeds within 5 seconds for 99th percentile
4. System must handle accounts with up to 100M followers without degrading feed latency for other users

## Constraints
1. Celebrity accounts (<0.1% of users, >1M followers) produce ~30% of content that appears in feeds — naive uniform fan-out is cost-prohibitive
2. Feed ranking requires per-user signals, so feeds cannot be fully pre-computed as static lists
3. Users follow up to 5,000 accounts on average, with a hard cap of 7,500

Time elapsed: 3 mins.

## Concepts to Explore

These areas are relevant to this problem. Research unfamiliar ones in `context.md` before starting your design.

- [Fan-out patterns (fan-out on write vs fan-out on read)](../concepts.md) — the core tradeoff in how post content reaches followers' feeds at asymmetric scale
- [CQRS (Command Query Responsibility Segregation)](../concepts.md) — separating read and write models when read and write paths have fundamentally different scaling characteristics
