# Progress Review

1. Scan all `design.md` files in `exercises/` for `## Review` sections
2. Scan all `design.md` files in `micro/` for `## Review` sections
3. Extract scores, patterns, and mental model gaps from both tracks
4. Report using the format below

## Output Format

```markdown
## Progress Report

**Full exercises completed:** N
**Micro-designs completed:** N

---

### Full Exercises

**Average score:** X/10
**Score trend:** [improving / flat / declining]

| Exercise | Problem | Score | Core Mental Model Gap |
|----------|---------|-------|-----------------------|
| 001 | [name] | X/10 | [one-line gap] |
| ... | | | |

#### By Criterion

| Criterion | Avg | Trend | Recurring Gap |
|-----------|-----|-------|---------------|
| Problem Understanding | | | |
| Architecture Quality | | | |
| Tradeoff Reasoning | | | |
| Failure Handling | | | |
| Operational Concerns | | | |
| Communication | | | |

---

### Micro-Designs

**Average score:** X/10
**Score trend:** [improving / flat / declining]

| Micro | Problem | Score | Framing Correct? |
|-------|---------|-------|------------------|
| 001 | [name] | X/10 | [yes / partial / no] |
| ... | | | |

#### By Criterion

| Criterion | Avg | Trend | Recurring Gap |
|-----------|-----|-------|---------------|
| Problem Framing | | | |
| Architecture Quality | | | |
| Tradeoff Reasoning | | | |
| Failure Handling | | | |
| Communication | | | |
| Operational Concerns | | | |

#### Framing Accuracy

Track whether framing is improving across micro-designs:
- Framing accuracy rate: X/N correct
- Most common framing error: [pattern]
- Trend: [improving / flat / declining]

---

### Cross-Track Patterns

Compare performance between full exercises and micro-designs:
- Are micro-design skills translating to full exercises?
- Are the same mental model gaps appearing in both tracks?
- Is time management improving?

### Mental Model Patterns

Identify recurring conceptual gaps across all exercises:
- [Pattern 1: e.g., "Treats distributed systems as database problems"]
- [Pattern 2: e.g., "Jumps to implementation before decomposing sub-systems"]
- [Pattern 3: e.g., "Acknowledges tradeoffs but doesn't propose solutions"]

### Specific Strengths
- [What they consistently do well, with exercise references]

### Specific Weaknesses
- [What they consistently miss, with exercise references]

### Recommended Focus Areas

Prioritized list of skills to develop:
1. [Highest impact improvement area]
2. [Second priority]
3. [Third priority]

For each, suggest:
- A specific technique or framework to practice
- Which past exercise to revisit with this lens
```

## Review Principles

- Reference actual exercise numbers and specific feedback
- Look for patterns, not just averages
- Identify the 1-2 mental model shifts that would have the biggest impact
- Be direct about what's not improving
- Track framing accuracy as a primary metric for micro-designs
- Look for evidence that micro-design practice is (or isn't) improving full exercise performance
