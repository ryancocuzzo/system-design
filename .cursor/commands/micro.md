# Generate Micro-Design

1. Find next number in `micro/`
2. Create `micro/NNN/problem.md` using the format below
3. Create `micro/NNN/context.md` with the context template
4. Create `micro/NNN/design.md` with the phased design template
5. Display problem inline

## Problem Format

```markdown
# [Problem Title]

## Context
[2-3 sentences. Same complexity as full exercises.]

## Before You Design

Answer these before writing any architecture:

1. What is the core technical challenge this system must solve?
   (Not the product goal — what makes this *hard* at the specified scale?)

2. Which of these framings best fits this problem?
   - [ ] A storage and retrieval problem
   - [ ] A streaming/pipeline problem
   - [ ] A coordination/consensus problem
   - [ ] A scheduling/state machine problem
   - [ ] A distributed counting/aggregation problem

3. What is the one design decision that, if wrong, makes everything else irrelevant?

## Sub-system Identification (5 minutes, no details yet)

List 3-6 sub-systems this problem decomposes into.
For each, write ONE sentence about its responsibility.
Do not design any of them yet.

## Functional Requirements
[Same depth as full exercises]

## Non-Functional Requirements
[Quantified: QPS, latency, storage, availability — same as full exercises]

## Constraints
[Same as full exercises]
```

## Context Template

```markdown
# Context Questions — [Problem Title]

Questions asked and answered before or during the design.
Separate from the design document to keep architectural reasoning clean.

## Before Design
[Ask and answer domain knowledge questions here before starting the design.]

## During Design
[Ask and answer questions that arise while designing. Note which phase triggered the question.]
```

## Design Template

```markdown
# Design: [Problem Title]

Target time: 45 minutes total.
If stuck on any phase beyond its time box, write what you know, mark it incomplete, and move on.

## Phase 1: Problem Framing (5 min)

**One-line summary:**

**Core technical challenge:**

**Framing:** [chosen category and why]

**The decision that matters most:**

## Phase 2: Sub-systems (5 min)

| Sub-system | Responsibility | Why it's separate |
|------------|---------------|-------------------|
| | | |

## Phase 3: Core Sub-system Deep Dive (20 min)

Pick the 2-3 sub-systems where the real complexity lives. Design those. Skip the straightforward ones.

### [Hardest sub-system]

### [Second hardest sub-system]

## Phase 4: Integration (5 min)

How do the sub-systems connect?

| From | To | What flows | Sync/Async | Why |
|------|----|-----------|------------|-----|
| | | | | |

## Phase 5: Addressing Non-Functional Requirements (5 min)

| Requirement | How addressed | Confidence |
|-------------|---------------|------------|
| | | high / medium / low |

## Phase 6: Failure Modes (5 min)

| Failure | Impact | Mitigation |
|---------|--------|------------|
| | | |

---

Time elapsed: ___
```

## Problem Requirements

- Genuine tradeoffs (no single obvious answer)
- Scale quantified with numbers
- Designable in 45 minutes with the phased structure
- Should test problem framing — the core challenge should not be the obvious surface-level reading

## Categories (rotate)

- Data systems (storage, caching, search)
- Real-time systems (messaging, notifications)
- Transactional systems (payments, inventory)
- Platform services (rate limiting, auth)
- Content systems (file storage, feeds)

Do not include solutions, architecture hints, or tradeoffs in the problem. The "Before You Design" framing options should include the correct framing but also plausible wrong framings.
