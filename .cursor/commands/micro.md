# Generate Micro-Design

1. Find next number in `micro/`
2. Create `micro/NNN/problem.md` using the format below
3. Create `micro/NNN/context.md` with the context template (including guided research prompts)
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

## Functional Requirements
[Same depth as full exercises]

## Non-Functional Requirements
[Quantified: QPS, latency, storage, availability — same as full exercises]

## Constraints
[Same as full exercises]

## Concepts to Explore

These areas are relevant to this problem. Research unfamiliar ones in `context.md` before starting your design.

- [Concept area 1 — one-line description of why it's relevant]
- [Concept area 2 — one-line description of why it's relevant]
- [Concept area 3 — one-line description of why it's relevant]
```

## Context Template

```markdown
# Context — [Problem Title]

## Guided Research (10 min)

Research these before starting your design. For each, write 1-2 sentences on what it is and when you'd use it. If time runs out, move on — don't let research eat your design time.

1. **[Concept]** — [Research prompt phrased as a question]

2. **[Concept]** — [Research prompt phrased as a question]

3. **[Concept]** — [Research prompt phrased as a question]

## During Design
[Questions that arise while designing. Note which phase triggered them.]
```

## Design Template

```markdown
# Design: [Problem Title]

Total time budget: 50 minutes (research + design + buffer).
If stuck on any phase beyond its time box, write what you know, mark it incomplete, and move on.

## Phase 1: Framing (5 min)

**One-line summary:**

**Core technical challenge:**

**Framing:** [chosen category and why]

**The decision that matters most:**

## Phase 2: Sub-systems and Connections (5 min)

List sub-systems, their responsibilities, and how they connect. Bulleted list, not a table.

## Phase 3: Deep Dive (20 min)

Pick the 2 sub-systems where the real complexity lives. Design those. Skip the straightforward ones.

### [Sub-system name]

**Approaches:** [A] vs [B]
**Choice:** [X] — [one sentence why]

**Design:**

### [Sub-system name]

**Approaches:** [A] vs [B]
**Choice:** [X] — [one sentence why]

**Design:**

## Phase 4: Stress Test (5 min)

Pick the 3-5 most critical NFRs and failure modes. You cannot cover all of them — prioritize.

| Concern | Type | How addressed | Confidence |
|---------|------|---------------|------------|
| | NFR / Failure | | high / medium / low |

---

Time elapsed: ___
```

## Problem Requirements

- Genuine tradeoffs (no single obvious answer)
- Scale quantified with numbers
- Designable in 50 minutes with the phased structure (including 10 min research)
- Should test problem framing — the core challenge should not be the obvious surface-level reading

## Concept Hint Requirements

- Include exactly 3 concept areas in "Concepts to Explore" drawn from `concepts.md`
- Pick the 3 most design-critical areas — the ones that, if unknown, would most damage the design
- Name the concept category and its relevance, but do not say which specific tool to use or how
- Generate 3 matching research prompts in `context.md`, each phrased as a question

## Categories (rotate)

- Data systems (storage, caching, search)
- Real-time systems (messaging, notifications)
- Transactional systems (payments, inventory)
- Platform services (rate limiting, auth)
- Content systems (file storage, feeds)

Do not include solutions, architecture hints, or tradeoffs in the problem. The "Before You Design" framing options should include the correct framing but also plausible wrong framings.
