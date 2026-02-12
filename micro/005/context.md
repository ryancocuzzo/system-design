# Context — Infrastructure Run Orchestration Platform

## Guided Research (5 min)

Research these before starting your design. For each, write 1-2 sentences on what it is and when you'd use it. If time runs out, move on.

1. **State machine orchestration** — What are the tradeoffs between modeling a multi-phase pipeline as an explicit state machine (with transitions and guards) versus a simple queue-and-status approach, particularly when phases can fail, be retried, or be canceled?

It seems like the tradeoffs are a queue-and-status approach is simple, but requires scattered logic, potentially ambiguity over time because rules are less explicit and the logic will get messy for states like retires and cancels. This is good for very simple systems.

An explicit state machine will require more design upfront (allowed states, allowed transitions) similar to a finite automaton, but then you will have more clarity and no invalid state errors which will be valuable long-term. This is good for systems that need to handle more complex scenarios, like the one we're building.

2. **Fair scheduling under shared capacity** — How do systems like weighted fair queuing or per-tenant concurrency limits prevent resource starvation in multi-tenant environments where tenants share a global worker pool?

Both systems help prevent resource starvation.

per-tentant concurrency limits are good because they ensure no tenant monopolizes the worker pool by capping each tenant to some # of jobs

weighted fair queuing is benefitial because you can then adjust the caps in response to a variety of factors (customer tier levels, actual usage patterns, etc) to ensure that your resources all have consistent usage.

Time elapsed: 36 mins.

## During Design
[Questions that arise while designing. Note which phase triggered them.]

Q: Design decision 1 says this:

> - **A.** Distributed lock per workspace — acquire a lock (Redis/etcd) with TTL before entering plan or apply; release on completion or timeout
> - **B.** Database-backed queue with row-level locking — each workspace has a run queue; a claim query atomically marks the next eligible run as active
> - **C.** Single-writer actor per workspace — assign each workspace to a partition owner that sequences all run transitions for that workspace in-process

I don't really understand the idea behind A or C. I'm only somewhat confident I understand B. Could you explain the ideas of each to me?

**A (Distributed lock):** Worker grabs a lock keyed by workspace ID before starting plan/apply. No lock = wait or move on. TTL ensures a crashed worker doesn't hold the lock forever. Simple, but workers must coordinate with an external store (Redis/etcd) on every run start/end.

**B (Queue + claim):** Runs are rows in a DB. Worker runs `UPDATE runs SET status='active' WHERE workspace_id=X AND status='queued' ORDER BY created_at LIMIT 1 RETURNING id` (or equivalent). The DB's ACID guarantees only one worker can claim the next run. The queue *is* the source of truth; no separate lock.

**C (Actor per workspace):** One process owns all runs for a set of workspaces. It maintains an in-memory queue and hands work to its own workers. No cross-process coordination — sequencing is local. Scales by sharding workspaces across actors; each actor is single-threaded for its partitions.

Q: In the answer above,

> Worker grabs a lock keyed by workspace ID before starting plan/apply. No lock = wait or move on.

This doesn't make sense to me. Where is this assuming the queued jobs data would be?

**A:** Option A doesn't specify — the queue is separate. You'd have runs stored somewhere (DB, Redis list, message queue) and workers would: 1) poll or receive a run for workspace X, 2) try to acquire the lock for X, 3) if lock acquired → execute; if not → re-queue or retry later. The lock only gates *who can run*; it doesn't define where runs live. B combines queue + lock in one place (the DB); A keeps them separate.

Time elapsed: 62 mins.

Q: Here is a snippet from the problem statement:

> ### Decision 2: How to handle cross-workspace dependency ordering when upstream state changes?

> - **A.** Synchronous graph walk — resolve the full dependency graph before enqueuing; block downstream runs until all upstream applies complete
> - **B.** Event-driven cascade — upstream apply emits a state-changed event; downstream workspaces independently enqueue runs on receipt
> - **C.** Version-gated planning — downstream runs read upstream state version at plan start; if upstream has an in-flight run, wait or re-plan after it lands

What are the "stream direction" they're referring to in the options? And, could you clarify the intent of each option? I am leaning towards modeling this dependency tree as a DAG and would like a lightweight way to do graph-based unblocking of dependencies.

**Stream direction:** If B depends on A's state (B reads A's output), A is **upstream** and B is **downstream**. Data flows A → B.

**Intent of each:**

- **A (Synchronous graph walk):** Before enqueuing any run, compute the full DAG and block until all ancestors have completed. E.g. B depends on A and C → don't enqueue B until both A and C are done. Central orchestrator holds the graph and blocks eagerly.

- **B (Event-driven cascade):** When an apply completes, emit an event. Each downstream listens and enqueues on receipt. Naively: A completes → B enqueues; B completes → C enqueues. For DAG unblocking (B depends on A and C), each downstream must track "how many upstreams have completed" — only enqueue when all ancestors are done. Events drive the check; the graph is in a dependency table.

- **C (Version-gated planning):** Don't block enqueue. Downstream runs start planning and read upstream state at plan time. If upstream has an in-flight run, either wait or plan with stale data and re-plan when upstream lands. The "gate" is at plan execution, not at enqueue.

**DAG + lightweight unblocking:** B fits best. Store edges in a `workspace_dependencies` table. On apply complete, emit event; a handler looks up dependents and for each checks "all my upstreams have a completed apply after my last run's plan" — if yes, enqueue. No central graph walk; each event triggers a local check. Option A also uses the DAG but blocks synchronously (heavier). C avoids graph logic at enqueue but pushes complexity into plan-time consistency.

Q: The one thing I'm not understanding during this design is the constant presence of the "workspace". I feel like that's a simple entity, and I'm not even spending much time modeling it. Am I oversimplifying the implications and/or constraints that workspaces provide?

**A:** Workspace is the unit of serialization, the node in the dependency graph, and the scope for state isolation. You don't need a heavy model — an `id`, `org_id`, and dependency edges are enough. The main implication: every decision (lock scope, queue scope, which events to emit) is keyed by workspace. If you have that, you're not oversimplifying.

