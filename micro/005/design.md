# Design: Infrastructure Run Orchestration Platform

Total time budget: 40 minutes (research + design + buffer).
If stuck on any phase beyond its time box, write what you know, mark it incomplete, and move on.

## Phase 1: Framing (5 min)

**One-line summary:** We're designing a system for running infra-as-code jobs and managing the lifecycle of those jobs.

**Core technical challenge:** Knowing which candidate jobs (pipeline "phases") are now unblocked when one job (pipeline "phase") finishes.

**Framing:** Which of these best fits this problem?
- [ ] A storage and retrieval problem
- [ ] A streaming/pipeline problem
- [ ] A coordination/consensus problem
- [X] A scheduling/state machine problem
- [ ] A distributed counting/aggregation problem

Time elapsed: 42 mins.

**The decision that matters most:**

## Phase 2: Sub-systems (5 min)

- Job creator => user submits pipeline, this breaks it out into different jobs with their dependent jobs and queues them
- Job resolver => listens for when jobs hit a terminal state (completed, failed, canceled) and unblocks/resolves the status of the jobs who were dependent on that job's outcome.
- Queue => pulls jobs (pipeline phases) that are unblocked and assigns a worker to them
- Worker => runs the job. pulls the input artifacts from blob storage, runs the job, and writes the output artifacts to blob storage. Listens for cancel signals.

Time elapsed: 52 mins.

## Phase 3: Key Decisions (15 min)

### Decision 1: How to implement workspace-level run serialization?

- **A.** Distributed lock per workspace — acquire a lock (Redis/etcd) with TTL before entering plan or apply; release on completion or timeout
- **B.** Database-backed queue with row-level locking — each workspace has a run queue; a claim query atomically marks the next eligible run as active
- **C.** Single-writer actor per workspace — assign each workspace to a partition owner that sequences all run transitions for that workspace in-process

**Your choice:** B
**Why:** The primary store for the queued jobs (phases) will be the DB. That will be where we do the querying to find jobs. We'll apply the lock there as well. I am not sure why not Option A, but I do get the feeling that coupling the "canidate" and lock data structures is a good pattern. Option C references partitioning which is good, but has only one worker. That would be an important bottleneck.
**How it works:** 
We have many workers jobs querying for jobs with status = QUEUED. Their query can do a join with a tenant usage table and # of currently running jobs per-tenant to guide fairness and tenant limiting. We also must **allow only 1 run per workspace to be ongoing at the same time** to protect against state corruption. To acquire a lock, the tenant can do a `SELECT * FOR UPDATE` operation while providing updating the status to RUNNING with a TTL for releasing the lock (in the case of worker failure).

Time elapsed: 73 mins.

### Decision 2: How to handle cross-workspace dependency ordering when upstream state changes?

- **A.** Synchronous graph walk — resolve the full dependency graph before enqueuing; block downstream runs until all upstream applies complete
- **B.** Event-driven cascade — upstream apply emits a state-changed event; downstream workspaces independently enqueue runs on receipt
- **C.** Version-gated planning — downstream runs read upstream state version at plan start; if upstream has an in-flight run, wait or re-plan after it lands

**Your choice:** B
**Why:** A would be expensive to do synchronously plus it does a heavy walk (full-graph). B enables the system to work asynchronously and also does a lieghtweight walk (over only the relevant dependents). C looks like it could cause issues or stalling at plan time and lead to stalled resources.
**How it works:**
When a worker job completes, it emits a message with the pipeline phase id. We then go lookup states in our DB that are not yet ready to be picked up becasue they're blocked by (dependent on) the job that just completed. For each of those jobs, if all thier dpenedent jobs have completed successfully, mark them as QUEUED to they are ready to be picked up.

Time elapsed: 93 mins.

### Decision 3: How to distribute runs across the shared worker pool fairly?

- **A.** Weighted fair queue with per-org concurrency caps — global queue with scheduling weights; each org limited to N concurrent runs
- **B.** Per-org partitioned capacity — statically allocate a slice of workers to each org; unused capacity is not shared
- **C.** Global FIFO with priority tiers — single queue ordered by submission time; paid tiers get priority slots, free tiers get best-effort

**Your choice:** A
**Why:** C would be the simplest, but could get monopolized by high-priority users with high job counts, so low-tier users could get starved. B would easily lead to starvation and inefficienty if lots of jobs are coming from one partition and few jobs from another. A addresses weighting for dynamic priority and limits to prevent monopolization from high-job-count users.
**How it works:**
The scheduler would need to pull the currant usage per-tennt (from the jobs table itself or another usage metadata table) and the priority per-tenant (from a metadata table mapping tenants to their assigned priority level) and join that with jobs table when picking a job. This would be complex query with several joins.

To ballpark the perfromance of that query, we have - 50k orgs (tenants). We'll have a huge job count to scan and we'll be joining this on the tenands table. This is a perforamance bottleneck.

I'm thinknig this is an expensive operation that we could counter by reducing the search space with sharding.
- a sharded tenants table by the hash of the tenand id
- multiple job queues (sharded by a hashed tenant ID perhaps)
- worker pools dedicated to each queue

The bottleneck here is clearly the JOIN, but I'm not sure how to counter it. I'm not sure about how to approach the querying or sharding/parittioning strategy for this data.

## Phase 4: Stress Test (5 min)

Pick the 3-5 most critical NFRs and failure modes. You cannot cover all of them — prioritize.

| Concern | Type | How addressed | Confidence |
|---------|------|---------------|------------|
| | NFR / Failure | | high / medium / low |

Worker pod failure: I have a high confidence this would be addressed by using TTLs in the DB locks.
Scale: I have medium confidence this is addressed. I think the run serialization is addressed, but the querying strategy and data placement / partitioning strategy are generally unaddressed.
Run start latency: The worker pickup model is generally addressed, but no specifics are given. I'm not sure how the calculation should be done for that.

---

Time elapsed: 125 mins.

---

## Review

**Score: 5/10**

| Criterion | Score | Notes |
|-----------|-------|-------|
| Problem Framing | 5/10 | Correct category, but core challenge misses state safety; "decision that matters most" left blank |
| Key Decisions | 6/10 | All three options correct, but mechanics conflate phases with runs and don't solve the scheduling query |
| Tradeoff Reasoning | 5/10 | Present but shallow; "I'm not sure why not A" is honest but not analysis |
| Stress Testing | 3/10 | Missed state durability (the hardest NFR); no table format; mitigations vague |
| Communication | 5/10 | Honest about gaps, good use of context.md for questions, but 125 min vs 40 min budget and Phase 4 incomplete |

### Framing Assessment

**Their framing:** "Knowing which candidate jobs (pipeline 'phases') are now unblocked when one job finishes."
**Correct framing:** Orchestrating multi-phase pipelines with per-workspace serialization and cross-workspace dependency ordering, where the execution substrate (agents) is unreliable and state corruption is catastrophic.
**Match:** Partially correct

The unblocking/scheduling angle is real, but it's only half the problem. The other half — and arguably the harder half — is state safety: ensuring that a crash mid-apply doesn't corrupt the state file, that serialization is never violated even under agent failure, and that state versions are immutable and rollback-safe. The problem says "zero tolerance for data loss or corruption" and "corrupted state file means production infrastructure becomes unmanageable." That's not a footnote — it's the constraint that should drive the entire architecture. This framing gap meant state management was never designed.

### Phase-by-Phase Feedback

**Phase 1 (Framing):** The scheduling/state machine category is correct. The one-line summary ("designing a system for running infra-as-code jobs and managing the lifecycle of those jobs") is too generic — it restates the problem without identifying what makes it hard. "The decision that matters most" was left blank. This should have been something like "how to guarantee state safety when agents can die mid-apply."

**Phase 2 (Sub-systems):** Four sub-systems named, which is a reasonable count. The concern: there's no **State Store** sub-system. For a problem where state durability is the most critical NFR, the system that manages state versions, encryption, rollback, and cross-workspace reads should be a first-class sub-system. The "Job resolver" concept is good — it maps to the event-driven cascade logic. The "Queue" and "Worker" distinction is fine but they're really one sub-system (the execution layer). A better decomposition:

1. Run Orchestrator — state machine that manages run lifecycle transitions
2. Workspace Scheduler — per-workspace serialization + cross-workspace dependency resolution
3. Capacity Scheduler — fair multi-tenant dispatch to the worker pool
4. State Store — versioned, encrypted state management with rollback
5. Worker Pool — ephemeral agents that pull and execute phases

**Phase 3 (Key Decisions):**

**Decision 1 — B (correct).** The intuition about co-locating the queue and lock is right but articulated as a feeling rather than a reason. The concrete reason: with B, the serialization invariant (one active run per workspace) is enforced by the same ACID transaction that claims the run. With A, you have two systems (queue + lock store) that must stay in sync — if the lock is acquired but the queue claim fails, or vice versa, you have inconsistency. The mechanics description conflates workspace serialization with tenant fairness (joining a tenant usage table during the claim query). These are two separate concerns: Decision 1 is about workspace-level mutual exclusion, Decision 3 is about org-level fairness.

**Decision 2 — B (correct).** The reasoning against A (expensive full walk) and C (stalling) is directionally right but shallow. The stronger argument for B: it's the only option that decouples workspace lifecycles. A requires a central coordinator that must block and track the full graph — a single point of failure and a scaling bottleneck. C pushes the problem to plan time, which means runs are enqueued and claim a worker only to discover they need to wait — wasting capacity. B lets each workspace react independently to upstream state changes.

The mechanics description is where the conflation problem shows up most clearly. They write: "lookup states in our DB that are not yet ready because they're blocked by the job that just completed." But Decision 2 is about **cross-workspace** dependencies (workspace B depends on workspace A's state output), not about intra-run phase ordering (plan → policy → approval → apply). The design treats pipeline phases and cross-workspace runs as the same concept. They're not: phases within a run are sequential by definition (the state machine handles that); cross-workspace dependencies are a separate graph.

**Decision 3 — A (correct).** Good reasoning for rejecting B (waste from static partitioning) and C (monopolization by high-priority users). The mechanics section is the most honest part of the design — correctly identifies the complex join as a performance bottleneck but can't solve it. The solution is simpler than they think: you don't need a join at query time. Keep an `org_running_count` counter (in Redis or an atomic DB column). When a worker requests work, the scheduler picks the next eligible run from the global queue, checks `org_running_count < org_limit`, and dispatches. The counter is incremented on dispatch, decremented on completion. No joins.

**Concrete weighted fairness:** Assign each org a weight (e.g., paid=2, free=1). Track `last_served_at` per org. When picking the next run to dispatch, among orgs with queued runs and under their cap, choose the one with the smallest `last_served_at / weight` — that's the org most "behind" its fair share. Update `last_served_at = now` when you dispatch. This is deficit round-robin: orgs with higher weight get proportionally more slots over time, but no org is starved as long as it has work. Alternative: if you only need tiered priority (paid before free, not strictly proportional), use a simple priority queue — paid-tier runs first, then free — with caps still enforced. That's simpler but doesn't give you fine-grained proportionality.

**Phase 4 (Stress Test):** Three concerns listed in prose instead of the table. The most critical issue: **state durability is completely absent.** This is the #1 NFR ("zero tolerance for data loss or corruption") and the design never addresses how state files are stored, versioned, or protected during apply. What happens if an agent crashes mid-apply after writing a partial state file? How do you rollback? How are state files encrypted? Worker failure via TTLs is fine but obvious. Run start latency is mentioned but unanalyzed — a back-of-the-envelope calculation (500K runs/day ÷ 86400 sec ≈ 6 runs/sec average, 30-second SLO means queue depth per workspace rarely exceeds 1) would have shown this is achievable with simple polling.

### Research Assessment

The state machine research was solid — genuine understanding of explicit FSM vs. queue-and-status tradeoffs, and it clearly informed the framing choice. The fair scheduling research was surface-level but adequate. The research *did* translate into decisions: the FSM insight supports the scheduling/state machine framing, and the per-tenant concurrency concept directly maps to Decision 3.

The context.md questions were excellent. Asking for clarification on options you don't understand is exactly right. The question about upstream/downstream terminology led to a deeper understanding that visibly improved Decision 2. The question about whether workspaces are being oversimplified was a good instinct — but the answer ("workspace is the partition key") should have prompted more attention to state management, and it didn't.

The biggest gap: the research never engaged with the **Reliability and Resilience** concept hint from the problem. That hint specifically called out "how state locking, run serialization, and failure recovery interact when the execution platform itself can fail mid-run." If that had been researched, state safety likely would have appeared in the design.

---

## Ideal System Architecture

```
═══ RUN LIFECYCLE (per workspace) ═══

  User trigger / Auto-trigger
        │
        ▼
  ┌─────────────┐    workspace has     ┌─────────────────┐
  │  Run Queue   │◄── active run? ────▶│  Wait (QUEUED)   │
  │  (per-wksp)  │    no               │  yes → park      │
  └──────┬───────┘                     └─────────────────┘
         │ claim (DB row lock,
         │ WHERE workspace has no active run)
         ▼
  ┌──────────────┐
  │ Run State    │  plan → policy_check → awaiting_approval → apply → completed
  │ Machine      │         │                                    │
  │ (DB-backed)  │     FAIL/CANCEL ──────────────────────► DISCARDED
  └──────┬───────┘
         │ on phase ready
         ▼
  ┌──────────────────────┐
  │  Capacity Scheduler  │  per-org running count < cap?
  │  (global dispatcher) │  yes → assign to agent
  │                      │  no  → wait for slot
  └──────────┬───────────┘
             │ agent polls, receives assignment
             ▼
  ┌──────────────────────┐
  │  Ephemeral Agent     │  pulls plan/apply task
  │                      │  reads input state from State Store
  │  heartbeat ──────────│──▶ orchestrator monitors liveness
  │                      │
  │  on apply success ───│──▶ write new state version to State Store
  │  on failure/crash ───│──▶ TTL expires → orchestrator reclaims run
  └──────────────────────┘

═══ CROSS-WORKSPACE CASCADE ═══

  Apply completes for workspace A
        │
        ▼
  ┌──────────────────────────┐
  │  State-Changed Event     │  (workspace_id, new_state_version)
  └──────────┬───────────────┘
             │
             ▼
  ┌──────────────────────────┐
  │  Dependency Resolver     │  SELECT dependents FROM workspace_deps
  │                          │  WHERE upstream = A
  │                          │  For each: all upstreams done? → enqueue run
  └──────────────────────────┘

═══ SUPPORTING SERVICES ═══

  ┌──────────────────────┐
  │  State Store          │  - Versioned, immutable state files in encrypted blob storage
  │                       │  - Current-version pointer per workspace (DB row)
  │                       │  - Reads: plan phase reads upstream state (< 200ms via pointer + cache)
  │                       │  - Writes: only on successful apply, atomic pointer swap
  │                       │  - Rollback: point to previous version
  │                       │  Queried by: agents (read), orchestrator (version check)
  └───────────────────────┘
  ┌──────────────────────┐
  │  Capacity Tracker     │  - per-org atomic counter of active runs
  │                       │  - increment on dispatch, decrement on completion/timeout
  │                       │  Queried by: Capacity Scheduler
  └───────────────────────┘
```

---

## Decision Analysis

### Decision 1: Workspace-level run serialization → B (DB-backed queue with row-level locking)

**Why B is correct:** The serialization invariant is "at most one run in plan or apply per workspace." With B, the claim query enforces this atomically: `UPDATE runs SET status='active' WHERE workspace_id=X AND status='queued' AND NOT EXISTS (SELECT 1 FROM runs WHERE workspace_id=X AND status='active') LIMIT 1`. One transaction, one source of truth. No split-brain possible.

**Why not A:** Distributed locks (Redis/etcd) are a separate system from wherever runs are stored. Two failure scenarios: (1) lock acquired, claim fails → lock is held with no run executing; (2) claim succeeds, lock write fails → two runs execute concurrently. You need distributed transactions across two systems to avoid this, which defeats the simplicity argument. Also, TTL tuning is tricky: too short and long-running applies lose the lock; too long and crashed agents block the workspace.

**Why not C:** 2M workspaces partitioned across actors means each actor owns thousands of workspaces. If an actor crashes, all its workspaces are frozen until the partition is reassigned. At 500K runs/day, this creates large blast radius. Also requires a partition assignment system (consistent hashing or a coordination service), adding operational complexity. B achieves the same serialization with no stateful coordinator.

**Cascade effect on Decision 2:** Because B stores runs in the DB with their status, the Dependency Resolver (Decision 2) can check upstream completion by querying the same DB — no need for a separate event bus if you use DB-level change notifications or polling. But an event-driven approach (Decision 2, Option B) is still better because it decouples the cascade from the serialization path.

### Decision 2: Cross-workspace dependency ordering → B (Event-driven cascade)

**Why B is correct:** Each workspace reacts independently to upstream changes. When workspace A's apply completes, the event handler checks A's dependents in the `workspace_deps` table, and for each dependent, verifies all upstreams are done before enqueuing. This is O(edges from A), not O(full graph). Workspaces that don't participate in the current cascade are unaffected.

**Why not A:** Synchronous graph walk requires a central coordinator that computes the full transitive closure before any run starts. At 2M workspaces with potentially deep dependency chains, this is a bottleneck and a single point of failure. It also blocks eagerly — downstream workspaces that could start (because their specific upstreams are done) must wait for unrelated branches to finish.

**Why not C:** Downstream runs start planning and then discover their upstream is stale. This wastes a worker slot (the agent is occupied re-planning or waiting). With 500K runs/day and a finite worker pool, this waste compounds. It also introduces plan-time non-determinism: the same run might produce different plans depending on when it happens to execute relative to its upstreams.

**Cascade effect on Decision 3:** The event-driven cascade can produce bursts of enqueued runs (one upstream apply triggers N downstream runs simultaneously). The Capacity Scheduler (Decision 3) must handle these bursts without letting one org's cascade monopolize the worker pool — reinforcing why per-org concurrency caps are needed.

### Decision 3: Fair scheduling → A (Weighted fair queue with per-org concurrency caps)

**Why A is correct:** The constraint says "no single organization should monopolize the execution pool." Per-org caps directly enforce this. Weighted fairness allows differentiation (paid vs. free tiers) without starving anyone. Unused capacity from orgs under their cap is available to others.

**Implementation (simpler than the design suggests):** Maintain an `active_run_count` per org (atomic counter in Redis or a DB column updated on dispatch/completion). Scheduler loop: pick next eligible run from global queue → check `org_active < org_cap` → if yes, dispatch and increment counter; if no, skip to next run from a different org. No joins needed. Weighted priority: multiply the org's queue position by a weight factor based on tier.

**Why not B:** Static partitioning wastes capacity. If org X is idle, its allocated workers sit empty while org Y's runs queue up. With 50K orgs and variable usage, this is extremely inefficient.

**Why not C:** Global FIFO with priority tiers lets a large org flood the queue with high-priority runs. "Priority" doesn't prevent monopolization — it enables it. Caps are the mechanism that prevents it; FIFO ordering doesn't help when one org submits 10x more runs.

---

## Key Differences

| Aspect | Their Design | Ideal Design | Why It Matters |
|--------|--------------|--------------|----------------|
| State management | Not designed | First-class sub-system: versioned blobs, atomic pointer swap, encryption, rollback | State durability is the #1 NFR; missing it means the design can't protect production infrastructure |
| Phase vs. run vs. workspace | Conflated — phases treated as independent jobs in a single global queue | Distinct layers: FSM sequences phases within a run, workspace queue serializes runs, dependency graph orders workspaces | Conflation means the serialization constraint ("one run per workspace") can't be enforced correctly |
| Agent failure handling | TTL on lock | Heartbeat + TTL: agent heartbeats to orchestrator; missed heartbeats trigger reclamation; state file only committed on confirmed success | TTL alone can't distinguish "slow apply" from "dead agent" — too-short TTL kills valid runs, too-long blocks the workspace |
| Scheduling mechanics | Complex multi-join query at claim time | Atomic per-org counter checked on dispatch; no joins | Their approach is a real bottleneck; the counter approach is O(1) per dispatch |
| Cross-workspace vs. intra-run | Same mechanism for both | FSM handles intra-run (plan→policy→approval→apply); event-driven cascade handles cross-workspace | Different ordering guarantees require different mechanisms |
| State reads for cross-workspace deps | Not mentioned | Versioned pointer per workspace; plan phase reads latest committed version by pointer lookup + optional cache | < 200ms p95 NFR requires this to be designed, not assumed |
| Stress testing | 3 concerns, missed durability | Durability, cascade storms, agent failure, workspace starvation, state encryption | Missing the most critical failure mode means the design is untested where it matters most |

---

## Core Mental Model

**How they conceptualized the problem:** A job scheduling system. Pipelines produce jobs, jobs get queued, workers pick them up, completed jobs unblock other jobs. The workspace is just a grouping label.

**How they should have conceptualized it:** A **state-safe orchestration system**. The workspace is the unit of isolation — it owns a state file that represents real infrastructure. Every design decision must answer: "can this corrupt state?" The run is a state machine that transitions through phases; the workspace enforces serialization so two runs never touch state concurrently; the agent is untrusted and ephemeral, so state writes must be confirmed atomically. Scheduling and dependency resolution are important but secondary to the guarantee that state is never lost or corrupted.

**Why this reframe matters:** The job-scheduling framing leads to a design where the hard problem is "how to pick the next job efficiently" — which is why they spent most of their time on query optimization for Decision 3. The state-safe orchestration framing leads to a design where the hard problem is "how to guarantee correctness under failure" — which would have surfaced state versioning, atomic commits, agent heartbeats, and rollback as first-class design elements. The scheduling problem is real but solvable with a simple counter; the state safety problem is the one that requires architectural thinking.
