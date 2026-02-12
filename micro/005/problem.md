# Infrastructure Run Orchestration Platform

## Context

You are building the core orchestration layer for a platform where teams manage infrastructure-as-code workspaces (similar to Terraform Cloud). Each workspace holds a state file that represents the current deployed infrastructure. Users trigger runs — multi-phase pipelines (plan → policy check → approval → apply) — against workspaces. Runs within a workspace must execute serially to prevent state corruption, but runs across independent workspaces should execute concurrently. Workspaces can declare dependencies on other workspaces' output state, creating cross-workspace ordering constraints during execution.

## Functional Requirements

- **Run lifecycle:** Users trigger runs that progress through plan → policy check → approval → apply. Each phase produces artifacts (plan file, policy results, state file). Runs can be canceled or discarded at any phase.
- **State management:** Each successful apply produces a new immutable state version. Workspaces support state rollback. A workspace can read another workspace's current state as a data source during the plan phase.
- **Workspace-level serialization:** Only one run per workspace can be in the plan or apply phase at a time. Queued runs wait until the active run completes or is discarded.
- **Cross-workspace run triggers:** When a workspace's state changes after an apply, dependent workspaces can be configured to automatically enqueue new runs, respecting dependency ordering.


## Non-Functional Requirements

- **Scale:** 50K organizations, 2M workspaces, 500K runs/day. Peak: 80 concurrent runs per organization during business hours.
- **Run start latency:** A queued run must begin its plan phase within 30 seconds of becoming eligible (no active run ahead of it in the same workspace).
- **State read latency:** Reading another workspace's current state < 200ms p95.
- **Durability:** State files are the source of truth for deployed infrastructure. Zero tolerance for data loss or corruption — a corrupted state file means production infrastructure becomes unmanageable.

Time elapsed: 9 mins.

## Constraints

- Runs execute on ephemeral worker agents provisioned per-run. Agents pull work from the platform; the platform does not push into agents. Agents may crash or become unreachable mid-run.
- State files range from 1KB to 50MB (average: 500KB). They contain sensitive data (resource IDs, connection strings, secrets) and must be encrypted at rest and in transit.
- Organizations share a finite pool of concurrent run capacity. No single organization should be able to monopolize the execution pool during peak hours.

Time elapsed: 10 mins.

## Concepts to Explore

These areas are relevant to this problem. Research unfamiliar ones in `context.md` before starting your design.

- **Reliability and Resilience** — how state locking, run serialization, and failure recovery interact when the execution platform itself can fail mid-run
- **Architecture Patterns** — how scheduling and state machine patterns apply to orchestrating multi-phase pipelines with cross-entity ordering constraints
