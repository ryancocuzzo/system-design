# Grade Design

1. Read `problem.md` and `design.md` in current exercise folder
2. Evaluate against `rubric.md`
3. Append to `design.md` using the format below

Key question: "What feedback would this get in a senior staff design review?"

## Output Format

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

Provide an ASCII diagram or clear description of what a strong solution looks like:
- Name the sub-systems and their responsibilities
- Show the data flow
- Explain key design decisions

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

## Grading Principles

- Be specific: reference their actual words and decisions
- Be thorough: every substantive point deserves examination
- Be constructive: show what good looks like, not just what's wrong
- Be honest: don't soften feedback to be nice
