# Context — Container Image Registry


## Context-setting questions

Q: I understand content-addressing data to basically be like generating a known-length unique hash for that data via some mechanism. Is that correct?

A: Directionally correct, but the defining property isn't the hash — it's that the content determines the address. You hash the bytes and the resulting digest (e.g., `sha256:a3b8f...`) *becomes* the storage key. This is the opposite of location-addressed storage where you pick an arbitrary path. Three properties fall out for free: (1) deduplication is automatic — identical content always produces the same digest, so two independent pushes resolve to one stored blob, (2) integrity is built-in — re-hash on pull and compare to the address to detect corruption, (3) immutability is enforced — changing content changes the hash, so it's a different object. In OCI registries: layers are stored at `sha256:<hex>`, a manifest is a JSON doc listing layer digests, and a tag (like `latest`) is the only mutable pointer — everything below the tag is immutable and content-addressed.

Q: I suspect that we'll need to be actually scanning layers themselves. Does that sound directionally correct?

A: Yes. Scanners (Trivy, Clair, Grype) decompose an image into layers, extract the filesystem from each, identify installed packages (dpkg, rpm, apk, pip, npm), and match against CVE databases. One nuance: while extraction happens per-layer, assessment needs the composed image context because a later layer can delete a vulnerable binary from an earlier layer — so scanners build the union filesystem view. The architectural win: since layers are immutable and content-addressed, scan results for a given digest are cacheable until the CVE database updates. You never re-scan `sha256:abc123` — you only re-evaluate when the CVE feed changes. For a registry where millions of images share base layers, this turns O(images) scanning into roughly O(unique layers × CVE update frequency).

Time elapsed: 17 mins.

## Guided Research (5 min)

Research these before starting your design. For each, write 1-2 sentences on what it is and when you'd use it. If time runs out, move on.

1. **Content-addressable storage and reference counting** — What happens when you need to garbage-collect objects in a content-addressable store where the same object is referenced by millions of independent parents? What makes this harder than traditional garbage collection?

Content-addressable garbage collection (removing items no longer referenced) is different than normal carbage collection because the problem fundamentally represents a graph. It would be inefficient to use a centralized tracking system because every time a reference is deleted, you would have to do a traversal of the node's children.

2. **Pull-through caching vs active replication** — When distributing immutable blobs across regions, what are the cost and latency tradeoffs between replicating everything proactively versus caching on first access? How does content-addressability affect cache correctness?

Active replication
- low-latency
- high egress and storage cost (linear to the total amount of data)
- high operational cost (replication mechanism)

Pull-thruogh caching
- latency depends on cache success in the region
- a cache miss is expensive (egress + cross-region time)
- requires an SLO that tolerates a bad worst-case query time

Content addressability just makes sure that the same data, no matter by whom, is hitting the same cache position, which is a large benefit.

Time elapsed: 38 mins.

## During Design
[Questions that arise while designing. Note which phase triggered them.]

Q: The design options reference this:

> - **C.** Chunked sub-layer deduplication — break layers into variable-size chunks (like rsync/Borg), dedup at chunk level for finer granularity

It looks like this means breaking down the data further than the layer-level? Does this post any benefits to content-addressing?

A: Yes, it means breaking each layer blob into variable-size chunks using content-defined chunking (CDC) — a rolling hash (Rabin, Buzhash) finds split boundaries based on content, not fixed offsets — and content-addressing each chunk independently. The benefit: it catches *near-duplicate* layers that layer-level content-addressing misses entirely. If you rebuild an image and one file changes, the entire layer gets a new SHA-256 digest even though 99% of the bytes are identical. With sub-layer chunking, only the chunks covering the changed bytes are new; the rest dedup against the old layer. This matters most in CI-heavy environments where images are rebuilt frequently with small diffs.

The costs are significant though: (1) your metadata index explodes from millions of layer digests to potentially billions of chunk digests, (2) serving a pull now requires reassembling a layer from chunks on the fly — adding latency and compute, (3) the OCI distribution spec operates at layer granularity so clients are unaware of chunks — it's purely a storage-side optimization, and (4) garbage collection gets harder because reference fan-in increases by orders of magnitude. For most registries, layer-level dedup already captures the big wins (shared base images), and the incremental savings from sub-layer chunking rarely justify the complexity.