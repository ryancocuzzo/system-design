# Grade Design

1. Detect whether the current exercise is in `exercises/` or `micro/`
2. Read `problem.md` and `design.md` in the current exercise folder
3. If in `micro/`, also read `context.md`
4. Evaluate against `rubric.md` (for exercises) or `micro-rubric.md` (for micro-designs)
5. Append to `design.md` using the appropriate format below
6. **If grading a micro-design:** After appending the review, read all `micro/*/design.md` files, extract the score from each `## Review` section (pattern: `**Score: X/10**`), and update the `## Progress` section in `README.md` with a Mermaid chart. Replace the x-axis array with micro numbers that have been graded (e.g. `[001, 002, 003]`), and the line array with their scores in the same order. Use `xychart-beta` with `title "Micro Scores"`, `x-axis`, `y-axis "Score" 0 --> 10`, and `line [scores]`.

Key question: "What feedback would this get in a senior staff design review?"

---

## Output Format for Full Exercises (`exercises/`)

```markdown
---

## Review

**Score: X/10**

| Criterion | Score | Notes |
|-----------|-------|-------|
| Problem Understanding | /10 | |
| Architecture Quality | /10 | |
| Tradeoff Reasoning | /10 | |
| Failure Handling | /10 | |
| Operational Concerns | /10 | |
| Communication | /10 | |

### Feedback
[Brief summary of critical issues]

---

## Detailed Critique

### On Structure and Clarity of Thought
- What worked in how they organized and communicated their thinking
- What didn't work (missing sub-system decomposition, dense sections, hand-waving, etc.)
- How the structure choice affected their ability to reason about the problem

### Specific Points Examined
Go through each substantive claim or decision in the design. For each:
- Quote or reference what they said
- Explain why it's correct, incomplete, or wrong
- Show the implications they didn't consider
- Suggest the better approach

Do NOT gloss over points. If they mentioned something, examine it.

---

## Ideal System Architecture

Provide an ASCII diagram showing what a strong solution looks like. Follow these diagramming principles:

### Diagram Structure
1. **Separate data flow from supporting services**
   - Main pipeline (data flow) at the top
   - Supporting/reference services in a separate section below
   - Use section headers like `═══ MAIN DATA FLOW ═══` and `═══ SUPPORTING SERVICES ═══`

2. **Distinguish relationship types**
   - Solid arrows (`───▶`) for data flow (events moving through the system)
   - Label arrows when the action isn't obvious (e.g., "writes delivery tasks", "when due")
   - For query/lookup relationships, don't use arrows—instead, list "Queried by:" or "Reads from:" inside the box

3. **Show branching explicitly**
   - Success/failure/retry paths should branch visually, not be described in prose
   - Use labels like `SUCCESS`, `FAILURE`, `EXHAUSTED` at branch points

4. **Avoid misleading vertical stacking**
   - Don't stack boxes vertically with arrows if they don't represent sequential data flow
   - Components that query each other are not pipeline stages

### Content Requirements
- Name each sub-system and its responsibilities
- Show the main data flow from ingestion to completion
- Show failure/retry paths as branches
- List supporting services separately with what they're queried by
- Include a "detail" section for complex component behavior if needed

---

## Key Differences

| Aspect | Their Design | Ideal Design | Why It Matters |
|--------|--------------|--------------|----------------|
| [aspect] | [what they did] | [what should be done] | [consequence] |
...

Cover at least 6-8 key architectural differences.

---

## Core Mental Model

Identify the fundamental framing shift that would have changed their approach.
- How did they conceptualize the problem?
- How should they have conceptualized it?
- Why does this reframe lead to better architecture?
```

---

## Output Format for Micro-Designs (`micro/`)

```markdown
---

## Review

**Score: X/10**

| Criterion | Score | Notes |
|-----------|-------|-------|
| Problem Framing | /10 | |
| Key Decisions | /10 | |
| Tradeoff Reasoning | /10 | |
| Stress Testing | /10 | |
| Communication | /10 | |

### Framing Assessment

**Their framing:** [what they identified as the core challenge]
**Correct framing:** [what the core challenge actually is]
**Match:** [correct / partially correct / wrong]

If wrong or partially correct, explain:
- What the correct framing is and why
- How the wrong framing affected their decisions
- The specific moment where the wrong framing led them astray

### Phase-by-Phase Feedback

**Phase 1 (Framing):** Did they identify the right core challenge? Did their "decision that matters most" actually matter most?

**Phase 2 (Sub-systems):** Were the sub-systems well-chosen? Did they miss a critical one? Did they include unnecessary ones? Did they stay disciplined (5 lines or fewer)?

**Phase 3 (Key Decisions):** For each decision:
- Was the chosen option correct given the constraints and NFRs?
- Does the justification reference specific requirements, or is it generic ("simplest", "safest")?
- Are the mechanics described concretely enough to evaluate feasibility?
- Did the framing from Phase 1 help or hurt the decision?

**Phase 4 (Stress Test):** Did they triage well — picking the most critical NFRs and failure modes rather than the obvious ones? Were mitigations concrete and specific to this system?

### Research Assessment

Review the `context.md`:
- Were the guided research prompts answered with genuine understanding (not just surface definitions)?
- Did the research translate into decisions in Phase 3?
- Was research done but then ignored when making decisions? (Specific anti-pattern to call out.)

---

## Ideal System Architecture

Provide an ASCII diagram showing what a strong solution looks like. Follow the same diagramming principles as full exercises.

---

## Decision Analysis

For each decision scaffold in Phase 3, explain:
- Which option was correct for this problem and why
- What constraints or NFRs make the other options wrong or worse
- What the mechanics of the correct approach look like in practice
- How this decision connects to the other decisions (cascading effects)

---

## Key Differences

| Aspect | Their Design | Ideal Design | Why It Matters |
|--------|--------------|--------------|----------------|
| [aspect] | [what they did] | [what should be done] | [consequence] |

---

## Core Mental Model

Identify the fundamental framing shift that would have changed their approach.
- How did they conceptualize the problem?
- How should they have conceptualized it?
- Why does this reframe lead to better decisions?
```

---

## Grading Principles

- Be specific: reference their actual words and decisions
- Be thorough: every decision deserves examination
- Be constructive: show what good looks like, not just what's wrong
- Be honest: don't soften feedback to be nice
- For micro-designs, the Decision Analysis section is the most important part of the review — it's where the user learns what the right answer was and why
- For micro-designs, explicitly assess whether guided research translated into Phase 3 decisions
