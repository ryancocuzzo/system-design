# CI/CD Orchestration Platform

## Context

You're building the core orchestration service for a cloud-hosted CI/CD platform similar to GitHub Actions. The system receives workflow definitions triggered by repository events (push, PR, schedule), schedules jobs onto ephemeral compute runners, and manages the full lifecycle of pipeline execution including artifacts, caching, and secrets.

## Functional Requirements

1. **Workflow Scheduling**: Accept workflow definitions (DAGs of jobs with dependencies) and schedule them onto available runners
2. **Runner Management**: Provision and manage ephemeral compute environments (containers/VMs) that execute jobs, then tear down
3. **Job Queue Management**: Queue jobs when runners are unavailable; support priority, cancellation, and timeout
4. **Artifact Storage**: Store and retrieve build artifacts (binaries, test results, logs) between jobs and after workflow completion
5. **Secrets Management**: Securely inject repository/organization secrets into job environments without exposing them in logs
6. **Build Caching**: Cache dependencies (npm, pip, maven) and intermediate build outputs to accelerate subsequent runs
7. **Quota & Fairness**: Enforce per-organization concurrency limits while ensuring fair scheduling across tenants
8. **Workflow Status**: Provide real-time status updates and streaming logs for running jobs

## Non-Functional Requirements

| Metric | Target |
|--------|--------|
| Organizations | 500,000 active |
| Workflows triggered | 50,000/minute peak |
| Concurrent jobs | 200,000 globally |
| Job start latency (queued → running) | p50 < 10s, p99 < 60s (when quota available) |
| Artifact storage | 5 PB total, 500 MB avg per workflow |
| Log ingestion | 100 GB/minute |
| Secrets access latency | p99 < 50ms |
| Cache hit rate target | > 70% for repeat builds |
| Availability | 99.9% for job scheduling |

## Constraints

- Runners execute untrusted user code and must be fully isolated
- Secrets must never appear in logs, artifacts, or be accessible to other tenants
- Workflows can have up to 100 jobs with complex dependency graphs
- Jobs can run from 10 seconds to 6 hours
- Must support multiple runner types: Linux, Windows, macOS, GPU, ARM
- Organizations can bring their own self-hosted runners
- Artifacts must be retained for 90 days by default
- Must gracefully handle runner failures mid-job (retry vs. fail)
