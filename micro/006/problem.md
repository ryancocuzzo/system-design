# URL Shortener Service

## Context
A platform needs a URL shortening service that creates compact, shareable links and redirects users to the original URLs. The service handles tens of millions of new URLs daily and billions of redirect lookups, with extreme popularity skew — a small fraction of URLs account for the vast majority of traffic.

## Functional Requirements
1. Given a long URL, generate a unique short URL (e.g., `sho.rt/Ab3xK9`)
2. Redirect users from a short URL to the original URL with minimal latency
3. Support optional custom short codes chosen by the user
4. Short URLs expire after a configurable TTL (default 5 years)

## Non-Functional Requirements
1. 10M new URLs created per day (~120 writes/sec average, ~500/sec peak)
2. 1B redirects per day (~12K reads/sec average, ~50K/sec peak)
3. p99 redirect latency < 10ms
4. 99.99% availability for the redirect path

## Constraints
1. Short codes: max 7 characters, base62 encoding (~3.5 trillion possible keys)
2. Short codes must not be sequentially enumerable
3. 5-year default retention — system must store up to ~18B URL mappings

## Concepts to Explore

These areas are relevant to this problem. Research unfamiliar ones in `context.md` before starting your design.

- **Hot partitions** — extreme read skew means a small number of keys dominate traffic, affecting both storage and cache design
- **Cache stampede protection** — at peak redirect volume, a cache miss on a popular key can cascade into database overload