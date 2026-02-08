# Micro-Design Evaluation Rubric

Micro-designs are time-boxed (50 min total including research), scaffolded exercises focused on problem framing, concept application, and architectural thinking. Grading reflects these priorities.

## Scoring Scale

| Score | Meaning |
|-------|---------|
| 9–10 | Strong framing, right concepts applied, clear structure, right sub-systems identified and explored. |
| 7–8 | Correct framing with 1-2 structural gaps. Research mostly translated to design. |
| 5–6 | Partial framing. Some right instincts, but core challenge not fully identified or research not applied. |
| 3–4 | Wrong framing or major structural problems. Research disconnected from design. |
| 0–2 | Did not complete or fundamentally off-target. |

## Criteria

| Criterion | Weight | Indicators |
|-----------|--------|------------|
| Problem Framing | 25% | Core challenge correctly identified, right mental model chosen, most important decision named |
| Architecture Quality | 20% | Sub-systems justified, responsibilities clear, core sub-systems explored with appropriate depth, guided research informed technology and pattern choices |
| Tradeoff Reasoning | 20% | Alternatives named and compared for key decisions, chosen approach justified, costs acknowledged |
| Stress Testing | 15% | Critical NFRs and failure modes identified and triaged, mitigations concrete even if brief |
| Communication | 15% | Phases structured clearly, reasoning traceable, incomplete sections explicitly marked |
| Operational Concerns | 5% | Basic awareness of observability, deployment, debugging |

## Per-Criterion Scoring

### Problem Framing (25%)
| Score | Indicators |
|-------|------------|
| 9-10 | Core challenge named precisely, framing category correct with explanation, identifies the decision that makes or breaks the design |
| 7-8 | Correct framing, most critical decisions identified |
| 5-6 | Partially correct framing, some confusion about what makes the problem hard |
| 3-4 | Wrong framing that leads design in wrong direction |
| 0-2 | No framing attempt or completely wrong problem |

### Architecture Quality (20%)
| Score | Indicators |
|-------|------------|
| 9-10 | Sub-systems well-chosen, core 2 explored with real depth, guided research clearly informed technology choices (e.g., chose a purpose-built store over a general-purpose DB because research revealed why) |
| 7-8 | Good sub-system choices, reasonable depth, research partially applied |
| 5-6 | Sub-systems present but responsibilities unclear, or research done but not connected to design decisions |
| 3-4 | Monolithic or incoherent decomposition, research ignored |
| 0-2 | No meaningful architecture |

### Tradeoff Reasoning (20%)
| Score | Indicators |
|-------|------------|
| 9-10 | Each core sub-system has 2+ approaches named, chosen approach justified with one clear sentence, costs acknowledged |
| 7-8 | Alternatives named for most decisions, reasoning present |
| 5-6 | Limited alternatives, shallow reasoning, or alternatives listed but no clear rationale for choice |
| 3-4 | Decisions stated without justification, no alternatives considered |
| 0-2 | No tradeoff reasoning |

### Stress Testing (15%)
| Score | Indicators |
|-------|------------|
| 9-10 | 3-5 most critical NFRs and failure modes identified and triaged, mitigations concrete |
| 7-8 | Key concerns named with reasonable mitigations, good prioritization |
| 5-6 | Basic awareness, generic mitigations, or poor prioritization (addressed easy concerns, skipped hard ones) |
| 3-4 | Minimal consideration or only generic failures (e.g., "DB goes down") |
| 0-2 | No stress testing |

### Communication (15%)
| Score | Indicators |
|-------|------------|
| 9-10 | Phases clearly structured, reasoning explicit, incomplete areas honestly marked with confidence levels |
| 7-8 | Clear overall, minor gaps in reasoning trail |
| 5-6 | Understandable but structure breaks down in places |
| 3-4 | Difficult to follow or phases ignored |
| 0-2 | Cannot follow the reasoning |

### Operational Concerns (5%)
| Score | Indicators |
|-------|------------|
| 9-10 | Key metrics named, basic deployment awareness |
| 7-8 | Some operational thinking present |
| 5-6 | Mentioned but not developed |
| 3-4 | Would be difficult to run |
| 0-2 | No operational consideration |
