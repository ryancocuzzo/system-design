# Container Image Registry

## Context

You are building a container image registry (like Docker Hub or GitHub Container Registry) that hosts OCI-compliant container images. Images are composed of a manifest (metadata) pointing to an ordered list of layer blobs. Layers are immutable, content-addressed by SHA-256 digest, and frequently shared across images — a base layer like `ubuntu:22.04` may be referenced by millions of images across thousands of repositories. The registry serves a global developer audience pushing and pulling images as part of CI/CD pipelines and local development.

## Functional Requirements

- **Push and pull:** Users push container images (manifest + layers) and pull them by tag (e.g., `latest`) or content digest. Layers already present in the registry are not re-uploaded.
- **Cross-repository deduplication:** Identical layers referenced by images in different repositories are stored exactly once. The registry tracks which manifests reference which layers.
- **Vulnerability scanning:** Pushed images are scanned against a CVE database. Users query scan results per image tag or digest. The CVE database updates daily.
- **Retention policies:** Repository owners configure tag retention rules (e.g., keep last 10 tags, delete untagged manifests older than 30 days). Layers with zero remaining references are reclaimed.

## Non-Functional Requirements

- **Scale:** 500K repositories, 60M image manifests, 200TB of deduplicated layer data. Average layer size: 40MB. Average image: 5 layers.
- **Pull latency:** Manifest resolution < 150ms p95 globally. Layer downloads served at CDN-edge speed from any region.
- **Push throughput:** 8K layer uploads/second aggregate across all regions. Individual CI pipelines may burst to 50 concurrent layer uploads.
- **Garbage collection:** Unreferenced layers reclaimed within 24 hours. GC must not block or degrade concurrent push/pull operations.

## Constraints

- Layer blobs are immutable and content-addressed. A single layer may be referenced by manifests across tens of thousands of repositories. Deleting a layer that still has live references corrupts those images.
- Multi-region: images pushed in one region must be pullable from any region within 60 seconds.
- Object storage (S3/GCS) is the layer backing store. Storage cost is the dominant operating expense — unused layers must be reclaimed, and cross-region replication costs money per GB transferred.

Time elapsed: 8 mins.

## Concepts to Explore

These areas are relevant to this problem. Research unfamiliar ones in `context.md` before starting your design.

- **Data Modeling and Storage** — how content-addressable storage, reference tracking, and garbage collection interact when shared objects have millions of inbound references across tenants
- **Caching** — how immutability and content-addressing change caching and CDN strategy for global distribution of large binary blobs
