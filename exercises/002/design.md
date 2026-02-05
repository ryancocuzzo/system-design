# CI/CD Orchestration Platform - Design

One-line summary: We're building a Job orchestration service for compute jobs. A job is triggered by an event (e.g PR, push, schedule) => triggers a workflow that is set to trigger upon that event should then be scheduled (for either now or later, depending on the event) => scheduling a workflow means scheduling the jobs which the workflow consists of in-order.

NOTE: Let's split this out into sub-systems.

## Sub-systems (high-level)

1. We can recieve repository events and, through a matching process, generate Workflow requests (an ingestion + workflow generator component)
    => Tells us what which workflows we should be executing and when
2. We can recieve workflow requests and, through a parsing process, generate a set of compute jobs, the commands and environment variables for the jobs, and their dependent job IDs. (a job generator component)
    => We now have a set of runnable compute jobs
3. A job queue, which gets the next job
    - this must respect fairness quotas and rate limits
4. Job runners, which are each provided the commands and environment variables to be run.
    => With these, we can now run our compute jobs.
5. Job runner manager, which should maintain the expected number of job runners. This component creates and tears down individual jobs. This gets the next job from the queuing system.
6. Blob storage for log files and build artifacts
7. A shared cache across workers for
- common build dependencies (e.g npm)
- intermediate build outputs
8. An API that
- lets users access their jobs (metadata, inputs, outputs)
- supports streaming for real-time status updates (workflow jobs completing, log file updates)
9. A secrets vault that is partitioned to give a multi-tenant experience (e.g /orgId/repoId/SECRET_NAME)

## Sub-systems (break-down)

1. Event ingestion engine
 - stateless api that listens to a kafka stream for Event messages (e.g push to commit in a repo)
 - 50k workflows triggered per minute means these must be horizontally scaled 
 - for each of the workflows in this repo, if they match this event metadata (e.g type == PUSH, branch == main) => We queue this workload onto our Workload Requests queue

2. Job generator
 - stateless api that listens to kafka stream for Workload Requests messages (message data will be the workload spec and the user/org info)
 - should scale similarly to event ingestion engine
 - parse the workload spec into into the following (using some pre-defined parsing process):
    - a set of jobs and their dependent jobs
    - for each job,
        - the base image
        - the code to be run
        - the environment variables and where they will be pulled from
        - the interim results generated from the job and those required for the job
 - We send these jobs and the user/org info to the Job Requests queue

3. Queueing system
 - stateless api that listens to stream for Job Requests messages (message data will be the job info and user/org info) and reads some DB to pull org and user usage and capacity info => stores the info into the DB
 - supports 1 endpoint: get next job (runs a query to pull the next job with the best priority score)
  - this query will calculate a `priority` score where factors that affect priority are:
   - job age
   - job's tenant's recent usage (usage over past 24 hours)
   - number of current jobs the tenant has running

4. Job execution system

4a. The job runners
 - stateful processes (e.g containers) that are provided code to run, the env vars to use, artifact info and user/org info.
 - Will try to "run"
    - grab the environment vars. If they are in a secret somewhere, grab the secret from the secrets store `/ORG_ID/REPO_ID/SECRET_NAME`
    - if there are intermediate outputs of other processes required, grab them from the shared cache `/ORG_ID/REPO_ID/RUN_ID/OUTPUT_NAME`
    - if this run will generate outputs that will be used in other processes, store them in that shared cache
    - if there are code dependencies that are used that are explictly mentioned in the job metadata (e.g npm), pull their binary to a local file and make any pre-defined caching adjustments possible (e.g set the PATH variable for the process). This might just be a set of pre-defined matching rules (e.g IF workflow contains step `npm`, execute these steps).
    - Create a log file in the blob store that we stream the program outputs to
        - to protect the secrets value, run a middleware process catching all logs before they happen. Match the log lines to our secrets and redact any matched sensitive content before propagating to the log file.
    - if any artifacts are created as a result of the job that are explicitly named, store them in the blob store
    - if cloning the repo is a "default" behavior for this product, this is where we would do that.
 - connected to kafka and will emit
  - Job.<jobId>.RUNNING upon the code process start
  - Job.<jobId>.FAILED upon failing a regular health check (check the process status) or exit with code 1
  - Job.<jobId>.DONE upon completing

4b. The job manager
 - stateless api that looks to maintain some constant amount of workers at all times (= "Capacity")
 - On a loop,
    - Checks the current amount of workers running (query DB for jobs with STATUS=READY)
    - if there is capacity (# jobs with STATUS=RUNNING < Capacity), ask the queue for the next job and create a new job runner with the job info
 - listens for kafka events coming from job runners and updates the job's status in the DB. 

 5. The user-facing REST API
 - stateless API that lets users access their jobs and supports streaming for real-time status updates
  - GET /workflows returns all workflows for this user (Postgres lookup, returning only a few important fields)
  - GET /jobs returns all jobs for this user (Postgres lookup, returning only a few important fields)
  - GET /jobs/:id is just another simple Postgres lookup but returns all this job's info (all metadata such as env vars, intermediary variables)

6. The user-facing websockets API
 - stateful API that listens for content changes for specific jobs' log files or for kafka events for their status changing.

## Addressing non-functional requirements

- For # of organizations, 500k active is an easy number to exist in one relational db table

- For # of workflows triggered, we'll need to use DB sharding to accomadate the storage requirement. We can use a consistent sharding mechanism based where our API servers will use the hash of the workflow id to calculate the DBs to pull from. It could be a set of DBs, which will make the user's GET /workflows logic slightly more complex. It will have to combine query results from the set of DBs. The horizontally scaled API pods and use of messaging should allaviate the backpressure issue though.

- For addressing job start latency p50 of < 10s at 200k jobs / minute. let's assume the average job takes 1 minute. We have 200k jobs per minute.

10s / start a job
we have 200k jobs / minute
we want to calculate how many workers we'll need.
let's say a job takes 30 seconds on average, consumes 1 worker => 1 worker does 2 jobs / minute => We need 100k workers to meet our demand such that all workers are available in the next minute.
10 seconds is 1/3 of the average runtime. If we can afford that as a buffer, we can operate with 2/3 of the compute capacity at 67k workers.

- For addressing the artifact storage, one S3 instance is plenty for 5PB

- for addressing the log ingestion, we'll need to use sharding for access to the S3 instance from the API servers. That scale grows too quickly. We should use a consistent sharding based on the job id. This should be a similar flow to the workflows DB.

- for secrets access latency, this design does not fetch the secrets values until runtime. And, at runtime, it does an access check. This will very likely be above 50ms. Thus, this is not satisfied comfortably.

- for addressing cache hit rate, this design doesn't cover the caching mechanism very granularly. It's difficult to say the hit rate expectation. This is not satisfied comfortably.

- For addressing availability, we could use a multi-AZ deployment strategy and DB replication on top of the horizontal scaling pattern that exists for the api servers.

Time elapsed: 125 mins.

---

## Review

**Score: 4/10**

| Criterion | Score | Notes |
|-----------|-------|-------|
| Problem Understanding | 5/10 | Core domain grasped, but missed explicit constraints (runner types, self-hosted, isolation requirements) |
| Architecture Quality | 5/10 | Has structure, but critical flows (DAG resolution, job lifecycle) are hand-waved |
| Tradeoff Reasoning | 3/10 | Almost no alternatives considered; decisions stated without justification |
| Failure Handling | 2/10 | Essentially skipped; no retry logic, idempotency, or graceful degradation |
| Operational Concerns | 3/10 | Multi-AZ mentioned, but no monitoring, metrics, debugging, or rollout strategy |
| Communication | 6/10 | Reasonably organized, honest self-assessment, but inconsistencies and missing diagrams |

### Feedback

The design shows basic domain understanding but misses the architectural core of a CI/CD orchestrator: **workflow state management and DAG execution**. The problem explicitly states workflows are DAGs with dependencies, yet there's no mechanism to track which jobs have completed, unblock dependent jobs, or manage workflow-level state. The "job queue" concept doesn't account for the fact that most jobs *cannot* be scheduled until their dependencies complete—this isn't a simple FIFO queue problem.

Critical gaps: no job isolation model (security requirement), no runner auto-scaling (cost/latency requirement), no failure/retry semantics (explicit constraint), no cancellation support (explicit requirement). The 125-minute time investment didn't translate into depth on core problems.

---

## Detailed Critique

### On Structure and Clarity of Thought

**What worked:**
- The sub-system decomposition approach was the right instinct—breaking into numbered components made the design scannable
- Separating the "high-level" from "break-down" sections showed intent to go deeper
- The self-assessment at the end acknowledging unmet requirements was honest and useful

**What didn't work:**
- The decomposition was *horizontal* (by technical layer) rather than *vertical* (by data lifecycle). You listed "Event ingestion → Job generator → Queue → Runners" but this obscures the **workflow state machine** that's the actual heart of the system
- Sub-systems 1-4 are described but never connected—there's no explanation of how a job completion in the runner triggers the next job in the DAG
- The "Addressing non-functional requirements" section reads as an afterthought rather than a design driver. Requirements like "p50 < 10s start latency" should shape the architecture, not be addressed post-hoc
- Dense prose without diagrams made it hard to verify data flow correctness

**How structure affected reasoning:**
By decomposing into independent "services" first, you missed that this is fundamentally a **state machine problem**. A workflow has states (pending → running → success/failed), jobs within it have states, and transitions between states drive the entire system. Without this framing, the design became a collection of components that don't compose into a working system.

### Specific Points Examined

---

**Point 1: "A job is triggered by an event... scheduling a workflow means scheduling the jobs which the workflow consists of in-order"**

This is incorrect. Jobs in a CI/CD workflow are not scheduled "in-order"—they're scheduled **when their dependencies are satisfied**. A workflow like:

```
build → [test-unit, test-integration] → deploy
```

Has parallelism: `test-unit` and `test-integration` can run concurrently after `build`. The design never addresses how the system knows when to unblock `deploy` (answer: when *both* test jobs complete). This requires a workflow state tracker that the design doesn't have.

---

**Point 2: "Job generator... parse the workload spec into a set of jobs and their dependent jobs"**

The parsing is mentioned, but what happens to this dependency information? It's sent to "the Job Requests queue" and then... disappears. There's no component that:
- Tracks which jobs in a workflow have completed
- Evaluates dependency conditions when a job finishes
- Schedules newly-unblocked jobs

The design assumes jobs can be queued independently. They cannot—a job with unmet dependencies must wait in a "pending" state, not in the scheduling queue.

---

**Point 3: "Queueing system... supports 1 endpoint: get next job"**

This conflates two different queuing needs:

1. **Workflow event queue**: High-throughput ingestion of triggers (Kafka makes sense here)
2. **Job scheduling queue**: Must respect dependencies, quotas, and runner affinity

The "priority score" formula (age, recent usage, current running) is reasonable for fairness, but it's the wrong abstraction. The queue should only contain **ready-to-run jobs**—jobs whose dependencies are satisfied. Mixing pending jobs with ready jobs in one queue means every "get next job" call must filter out blocked jobs, which is O(n) in queue depth.

---

**Point 4: "Job runners... containers that are provided code to run"**

Missing the security model entirely. The constraint states: "Runners execute untrusted user code and must be fully isolated." Containers alone don't provide this—a malicious workflow can:
- Attempt container escape
- Access the host network
- Read secrets from other tenants via timing attacks
- Mine cryptocurrency using your compute

Industry solutions include:
- **Firecracker microVMs** (what GitHub Actions uses for hosted runners)
- **gVisor** sandboxed containers
- **Dedicated VMs** with no shared kernel

None of these are mentioned. This is a critical security gap.

---

**Point 5: "Grab the secret from the secrets store /ORG_ID/REPO_ID/SECRET_NAME"**

This path structure implies secrets are only at the repo level. GitHub Actions supports:
- **Organization secrets** (shared across repos)
- **Repository secrets**
- **Environment secrets** (e.g., prod vs. staging)

The design doesn't model this hierarchy. Also missing:
- How secrets are encrypted (at rest? in transit? in memory?)
- How the runner authenticates to the secrets store
- How secrets are scoped (a PR from a fork shouldn't access repo secrets)

---

**Point 6: "To protect the secrets value, run a middleware process catching all logs... match the log lines to our secrets and redact"**

This is the right idea but incomplete. Secret redaction via string matching can be bypassed:
- Base64-encode the secret
- Print one character per line
- Reverse the string
- Use timing side-channels

Better approaches:
- Network egress filtering (prevent arbitrary outbound connections)
- Secrets not in environment variables but fetched via API with audit logging
- Short-lived tokens instead of static secrets

---

**Point 7: "The job manager... maintains some constant amount of workers at all times"**

This is backwards. CI/CD workloads are extremely bursty—most orgs push code during business hours in their timezone. A constant worker pool means:
- Paying for idle compute at 3 AM
- Not having enough capacity at 10 AM

The industry approach:
- **Warm pool**: Small number of pre-provisioned runners for instant start
- **Auto-scaling**: Spin up additional runners based on queue depth
- **Multiple runner types**: Can't maintain "constant" pools for Linux, Windows, macOS, GPU, ARM simultaneously

The design doesn't address runner types at all—a constraint that affects architecture significantly.

---

**Point 8: "Checks the current amount of workers running (query DB for jobs with STATUS=READY)"**

This is confused. If checking workers, you'd query for `STATUS=RUNNING`. `READY` would indicate jobs ready to be picked up. The job manager polling a DB in a loop is also problematic:
- Polling interval creates latency (p50 < 10s requirement)
- DB becomes a bottleneck at 200k concurrent jobs
- No distributed coordination—multiple job managers might grab the same job

Better: Event-driven scheduling where job completion events trigger dependency evaluation, which pushes newly-ready jobs to the scheduler.

---

**Point 9: "For addressing artifact storage, one S3 instance is plenty for 5PB"**

S3 isn't measured in "instances"—it's a managed service with effectively unlimited capacity. This phrasing suggests a mental model of provisioned storage. The actual concerns for 5 PB are:
- **Cost**: S3 Standard at 5 PB ≈ $115k/month; lifecycle policies to move to cheaper tiers matter
- **Deduplication**: Many workflows produce identical artifacts (same library version); content-addressed storage reduces cost
- **Access patterns**: Artifacts are write-once, read-occasionally; presigned URLs avoid routing through your services
- **Cross-job artifact passing**: Needs low latency for jobs that produce outputs consumed by dependent jobs

---

**Point 10: "For addressing the log ingestion, we'll need to use sharding for access to the S3 instance from the API servers"**

This misidentifies the problem. S3 can handle 100 GB/min easily (it's designed for this). The challenges are:
- **Real-time streaming**: Users want to see logs as jobs run, not after. The WebSocket API says "listens for content changes for log files"—polling S3 for changes is inefficient and high-latency
- **Log structure**: Logs need to be indexed for search ("show me all failures containing 'OOM'")
- **Log retention**: 100 GB/min × 60 × 24 × 90 days = 13 PB for retention period

Better: Stream logs through Kafka → Flink for real-time delivery → S3 for archival, with Elasticsearch for search.

---

**Point 11: "This design does not fetch the secrets values until runtime... This will very likely be above 50ms"**

Good self-awareness. The solution is caching:
- Secrets are fetched once at job start, decrypted, and held in memory
- Secrets store should be replicated in each region
- Pre-warming: When a job is scheduled, start fetching secrets before the runner is ready

---

**Point 12: Numbers calculation**

> "let's say a job takes 30 seconds on average"

This contradicts the problem statement: "Jobs can run from 10 seconds to 6 hours." A 30-second average is arbitrary. CI jobs typically have bimodal distribution: quick lint/test jobs (~1-5 min) and long build/deploy jobs (~15-60 min).

> "200k jobs / minute"

The problem says 200k **concurrent** jobs, not per minute. These are different:
- 200k concurrent with 30-min average = 400k completions/hour = 6.7k/min
- 200k concurrent with 1-min average = 200k completions/min

The calculation conflates these, leading to incorrect capacity planning.

---

**Point 13: Failure handling (missing)**

The constraint explicitly states: "Must gracefully handle runner failures mid-job (retry vs. fail)."

Not addressed:
- **Retry policy**: How many retries? Exponential backoff?
- **Idempotency**: If a job ran partially before crash, can it be re-run safely?
- **Failure classification**: OOM vs. timeout vs. user error vs. infra failure—different responses needed
- **Workflow-level failure**: If one job fails, does the whole workflow fail? What about `continue-on-error`?
- **Orchestrator failure**: What if the scheduler crashes mid-workflow? How does state recover?

---

**Point 14: Cancellation (missing)**

The requirements include "support cancellation." Not mentioned in design.

Cancellation is hard:
- User cancels workflow → must terminate all running jobs
- Running job may have spawned child processes
- Partial artifacts should be cleaned up
- Dependent jobs should be marked as "cancelled", not "pending"
- Billing: Do you charge for partially-run jobs?

---

**Point 15: Self-hosted runners (missing)**

Constraint: "Organizations can bring their own self-hosted runners."

This fundamentally changes the architecture:
- Self-hosted runners pull jobs (they're behind firewalls, can't receive pushes)
- Runner registration and health checking needed
- Jobs must be routable to specific runner pools (org-scoped)
- Security: Self-hosted runners can access internal networks—different trust model

---

## Ideal System Architecture

```
════════════════════════════════════════════════════════════════════════════════
                              MAIN DATA FLOW
════════════════════════════════════════════════════════════════════════════════

                         ┌────────────────────┐
     GitHub/SCM Events   │   Event Gateway    │  • Webhook receiver
     ─────────────────▶  │   (Stateless)      │  • Event validation
                         └─────────┬──────────┘  • Deduplication
                                   │
                                   │ writes workflow trigger
                                   ▼
                         ┌────────────────────┐
                         │   Workflow Queue   │  Kafka topic: workflow.triggers
                         │   (Kafka)          │  Partitioned by org_id
                         └─────────┬──────────┘
                                   │
                                   │ consumes
                                   ▼
                         ┌────────────────────┐
                         │ Workflow Planner   │  • Parse workflow YAML
                         │ (Stateless)        │  • Build job DAG
                         │                    │  • Create workflow record
                         └─────────┬──────────┘  • Mark root jobs as READY
                                   │
                                   │ writes workflow + jobs
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        WORKFLOW STATE STORE                                   │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐          │
│  │   Workflows     │    │      Jobs       │    │  Dependencies   │          │
│  │  workflow_id    │    │    job_id       │    │   job_id        │          │
│  │  org_id         │    │  workflow_id    │    │   depends_on    │          │
│  │  status         │    │  status         │    │   (join table)  │          │
│  │  created_at     │    │  runner_type    │    └─────────────────┘          │
│  └─────────────────┘    │  attempts       │                                  │
│                         └─────────────────┘    Sharded by org_id             │
└──────────────────────────────────────────────────────────────────────────────┘
                                   │
           ┌───────────────────────┴───────────────────────┐
           │ Job status = READY                            │
           ▼                                               │
┌────────────────────┐                                     │
│  Job Ready Queue   │  Kafka topic: jobs.ready            │
│  (Kafka)           │  Partitioned by runner_type         │
└─────────┬──────────┘                                     │
          │                                                │
          │ ┌──────────────────────────────────────────────┤
          │ │                                              │
          ▼ ▼                                              │
┌────────────────────┐    ┌────────────────────┐          │
│     Scheduler      │    │   Quota Service    │          │
│   (Per Region)     │◀───│                    │          │
│                    │    │ • Concurrency cap  │          │
│ • Pull ready jobs  │    │ • Fair share calc  │          │
│ • Check quotas     │    │ • Burst allowance  │          │
│ • Match to runners │    └────────────────────┘          │
└─────────┬──────────┘                                     │
          │                                                │
          │ assigns job to runner                          │
          ▼                                                │
┌────────────────────────────────────────────────────────┐ │
│                   RUNNER FLEET                         │ │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │ │
│  │  Warm Pool   │  │ Auto-scaled  │  │ Self-hosted  │ │ │
│  │  (Firecracker│  │   (Spot/     │  │   (Pulls     │ │ │
│  │   microVMs)  │  │  On-demand)  │  │    jobs)     │ │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │ │
│         │                  │                │         │ │
│         └──────────────────┴────────────────┘         │ │
│                            │                          │ │
└────────────────────────────┼──────────────────────────┘ │
                             │                            │
          ┌──────────────────┴──────────────────┐         │
          │                                     │         │
          ▼                                     ▼         │
   ┌─────────────┐                      ┌─────────────┐   │
   │   SUCCESS   │                      │   FAILURE   │   │
   └──────┬──────┘                      └──────┬──────┘   │
          │                                    │          │
          │              ┌─────────────────────┤          │
          │              │                     │          │
          │              ▼                     ▼          │
          │     ┌────────────────┐    ┌────────────────┐  │
          │     │  Retry?        │    │  Exhausted     │  │
          │     │  attempts < 3  │    │  Mark FAILED   │  │
          │     └───────┬────────┘    └───────┬────────┘  │
          │             │                     │           │
          │             │ re-queue            │           │
          │             └─────────────────────┼───────────┤
          │                                   │           │
          ▼                                   ▼           │
┌────────────────────┐              ┌────────────────────┐│
│ Dependency Eval    │              │ Workflow Failed    ││
│                    │              │ (unless continue-  ││
│ • Mark job DONE    │              │  on-error)         ││
│ • Find dependents  │              └────────────────────┘│
│ • Check all deps   │                                    │
│ • If satisfied →   │────────────────────────────────────┘
│   mark READY       │     enqueue newly ready jobs
└────────────────────┘

════════════════════════════════════════════════════════════════════════════════
                            SUPPORTING SERVICES
════════════════════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────────────────┐
│                           SECRETS SERVICE                                    │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐         │
│  │  Vault Backend  │    │  Secret Cache   │    │  Access Policy  │         │
│  │  (Encrypted)    │    │  (Per-region)   │    │  Engine         │         │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘         │
│                                                                             │
│  Queried by: Runner (at job start, with job token)                         │
│  Scopes: Organization → Repository → Environment                            │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                           ARTIFACT SERVICE                                   │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐         │
│  │  S3 (Archive)   │    │  Fast Tier      │    │  Metadata DB    │         │
│  │  Lifecycle:90d  │    │  (Cross-job)    │    │  (Dedup index)  │         │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘         │
│                                                                             │
│  Queried by: Runner (upload/download), API (user download)                  │
│  Uses presigned URLs; content-addressed for dedup                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                            CACHE SERVICE                                     │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐         │
│  │  Cache Index    │    │  Blob Storage   │    │  LRU Eviction   │         │
│  │  (key→blob)     │    │  (Regional)     │    │  (Per-repo cap) │         │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘         │
│                                                                             │
│  Queried by: Runner (restore/save cache steps)                              │
│  Key: hash(lock file contents, OS, arch)                                    │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                             LOG SERVICE                                      │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐         │
│  │  Log Ingestion  │    │  Live Stream    │    │  Archive        │         │
│  │  (Kafka)        │───▶│  (Redis/WS)     │    │  (S3 + ES)      │         │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘         │
│                                                                             │
│  Written by: Runner (streams stdout/stderr)                                 │
│  Queried by: WebSocket API (live), REST API (historical)                    │
│  Redaction: Applied at ingestion before any storage                         │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                           USER API LAYER                                     │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐         │
│  │  REST API       │    │  WebSocket API  │    │  GraphQL API    │         │
│  │  (CRUD)         │    │  (Live logs,    │    │  (Flexible      │         │
│  │                 │    │   status)       │    │   queries)      │         │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘         │
│                                                                             │
│  Reads from: Workflow State Store, Log Service, Artifact Service            │
│  Auth: OAuth + org membership check                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key design decisions in ideal architecture:**

1. **Workflow State Store is the source of truth** — Not Kafka. Kafka is for decoupling and buffering, but job state must be durable and queryable.

2. **Dependency Evaluator is a separate component** — When a job completes, it evaluates which dependent jobs are now unblocked and marks them READY. This is the core state machine.

3. **Two-tier queueing** — `workflow.triggers` for high-throughput ingestion; `jobs.ready` for schedulable work. Jobs with unmet dependencies never enter the ready queue.

4. **Runner fleet is heterogeneous** — Warm pool for instant start, auto-scaled for capacity, self-hosted for enterprise. Different provisioning strategies, same job protocol.

5. **Firecracker microVMs for isolation** — Not just containers. Untrusted code needs kernel-level isolation.

6. **Retry is explicit** — Failure goes through retry evaluation, not automatic re-queue. Tracks attempt count, applies backoff.

7. **Supporting services are separated** — Secrets, artifacts, cache, logs are independent services with their own scaling characteristics.

---

## Key Differences

| Aspect | Their Design | Ideal Design | Why It Matters |
|--------|--------------|--------------|----------------|
| **Core abstraction** | Collection of services processing jobs | Workflow state machine with dependency resolution | Without the state machine, there's no way to know when dependent jobs should run. The system can't actually execute multi-job workflows correctly. |
| **Job queue model** | Single queue with priority scoring | Two queues: trigger ingestion + ready-to-run jobs | Mixing pending and ready jobs in one queue means O(n) filtering. Ready queue contains only immediately schedulable work. |
| **Runner provisioning** | "Constant amount of workers" | Warm pool + auto-scaling + self-hosted | Constant pools waste money during off-hours and can't handle bursts. CI workloads are highly variable. |
| **Job isolation** | "Containers" | Firecracker microVMs with network isolation | Containers share a kernel with the host. Untrusted code can attempt escape. MicroVMs provide VM-level isolation with container-like density. |
| **Failure handling** | Job emits FAILED event, manager updates DB | Explicit retry policy with attempt tracking, idempotency requirements, failure classification | Without retry semantics, transient failures become permanent. Without idempotency, retries can corrupt state. |
| **Secrets architecture** | Flat `/org/repo/secret` namespace | Hierarchical scopes (org → repo → environment) with access policies | Real CI/CD needs different secrets for prod vs. staging. Flat namespace can't model environment-specific secrets. |
| **Log streaming** | "WebSocket listens for content changes for log files" | Kafka ingestion → Redis pub/sub → WebSocket broadcast | Polling S3 for changes is high-latency and expensive. Real-time needs push-based streaming. |
| **Cache key strategy** | Not specified | Hash of lockfile + OS + arch | Without deterministic cache keys, hit rate will be low. Key must capture all inputs that affect cache validity. |

---

## Core Mental Model

**How they conceptualized the problem:**
A pipeline of services that process events into jobs, queue jobs, and run them on workers. Each service is independent; data flows through Kafka from one stage to the next.

**How they should have conceptualized it:**
A **distributed state machine** managing workflow lifecycles. The central question isn't "how do I queue jobs?" but "how do I track which jobs can run and transition them through states as dependencies are satisfied?"

**Why this reframe matters:**
The pipeline model treats each job as independent. But CI/CD workflows have *structure*—a DAG of dependencies. You can't just "get next job" from a queue because most jobs are blocked waiting for others. The state machine model forces you to design:

1. Where workflow/job state lives (the state store)
2. How state transitions happen (the dependency evaluator)
3. What triggers transitions (job completion events)
4. How to recover state after crashes (event sourcing or checkpointing)

With the pipeline model, you end up with components that don't connect—which is exactly what happened in this design. The job generator creates jobs with dependencies, but nothing consumes that dependency information to actually enforce execution order.

The "aha" moment: **The scheduler doesn't schedule workflows. It schedules individual jobs that are ready. Something else must determine readiness.**