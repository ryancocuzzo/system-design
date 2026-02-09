# Generate Micro-Design

1. Find next number in `micro/`
2. Create `micro/NNN/problem.md` using the format below
3. Create `micro/NNN/context.md` with the context template (including 2 guided research prompts)
4. Create `micro/NNN/design.md` with the phased design template (including 2-3 decision scaffolds)
5. Display problem inline

## Problem Format

```markdown
# [Problem Title]

## Context
[2-3 sentences. Same complexity as full exercises.]

## Functional Requirements
[3-4 requirements. Focus on the core challenge. Cut anything that adds scope without adding architectural insight.]

## Non-Functional Requirements
[3-4 quantified requirements. Only the ones that actually constrain the architecture.]

## Constraints
[2-3 constraints. Only the ones that force tradeoffs.]

## Concepts to Explore

These areas are relevant to this problem. Research unfamiliar ones in `context.md` before starting your design.

- [Concept area 1 — one-line description of why it's relevant]
- [Concept area 2 — one-line description of why it's relevant]
```

## Context Template

```markdown
# Context — [Problem Title]

## Guided Research (5 min)

Research these before starting your design. For each, write 1-2 sentences on what it is and when you'd use it. If time runs out, move on.

1. **[Concept]** — [Research prompt phrased as a question]

2. **[Concept]** — [Research prompt phrased as a question]

## During Design
[Questions that arise while designing. Note which phase triggered them.]
```

## Design Template

```markdown
# Design: [Problem Title]

Total time budget: 40 minutes (research + design + buffer).
If stuck on any phase beyond its time box, write what you know, mark it incomplete, and move on.

## Phase 1: Framing (5 min)

**One-line summary:**

**Core technical challenge:** [Not the product goal — what makes this hard at scale?]

**Framing:** Which of these best fits this problem?
- [ ] A storage and retrieval problem
- [ ] A streaming/pipeline problem
- [ ] A coordination/consensus problem
- [ ] A scheduling/state machine problem
- [ ] A distributed counting/aggregation problem

**The decision that matters most:**

## Phase 2: Sub-systems (5 min)

Name 3-5 sub-systems. One sentence each. Do NOT describe how they work — that's Phase 3.
If you're writing more than 5 lines, you're over-designing this phase.

## Phase 3: Key Decisions (15 min)

### Decision 1: [Architectural question]

- **A.** [option with one-line description]
- **B.** [option with one-line description]
- **C.** [option with one-line description]

**Your choice:**
**Why:** [1-2 sentences]
**How it works:** [describe the mechanics]

### Decision 2: [Architectural question]

- **A.** [option with one-line description]
- **B.** [option with one-line description]

**Your choice:**
**Why:** [1-2 sentences]
**How it works:** [describe the mechanics]

### Decision 3: [Architectural question]

- **A.** [option with one-line description]
- **B.** [option with one-line description]

**Your choice:**
**Why:** [1-2 sentences]
**How it works:** [describe the mechanics]

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
- Designable in 40 minutes with the phased structure (including 5 min research)
- Should test problem framing — the core challenge should not be the obvious surface-level reading
- Scope is focused: 3-4 functional requirements, 3-4 NFRs, 2-3 constraints

## Decision Scaffold Requirements

- Include 2-3 architectural decisions in the design template's Phase 3
- Each decision must be a genuine choice — all options are real approaches used in production systems
- Options should be concrete (name the pattern/technology) with a one-line description
- The correct option for this problem must not be obvious without understanding the constraints and NFRs
- Decisions should build on each other: the choice in Decision 1 should affect what makes sense in Decision 2
- Do NOT include the rationale for any option — the user must reason about fit

## Concept Hint Requirements

- Include exactly 2 concept areas in "Concepts to Explore" drawn from `concepts.md`
- Pick the 2 most critical to the decisions in Phase 3
- Name the concept category and its relevance, but do not say which specific tool to use or how
- Generate 2 matching research prompts in `context.md`, each phrased as a question

## Categories (rotate)

- Data systems (storage, caching, search)
- Real-time systems (messaging, notifications)
- Transactional systems (payments, inventory)
- Platform services (rate limiting, auth)
- Content systems (file storage, feeds)

Do not include solutions, architecture hints, or tradeoffs in the problem. The framing options in the design template should include the correct framing but also plausible wrong framings.
