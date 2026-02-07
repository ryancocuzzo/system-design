# Micro-Design Evaluation Rubric

Micro-designs are time-boxed (45 min), scaffolded exercises focused on problem framing and architectural thinking. Grading reflects these priorities.

## Scoring Scale

| Score | Meaning |
|-------|---------|
| 9–10 | Strong framing, clear structure, right sub-systems identified and explored. |
| 7–8 | Correct framing with 1-2 structural gaps. |
| 5–6 | Partial framing. Some right instincts, but core challenge not fully identified. |
| 3–4 | Wrong framing or major structural problems. |
| 0–2 | Did not complete or fundamentally off-target. |

## Criteria

| Criterion | Weight | Indicators |
|-----------|--------|------------|
| Problem Framing | 25% | Core challenge correctly identified, right mental model chosen, most important decision named |
| Architecture Quality | 20% | Sub-systems justified, responsibilities clear, core sub-systems explored with appropriate depth |
| Tradeoff Reasoning | 20% | Alternatives considered for key decisions, costs acknowledged, reasoning explicit |
| Failure Handling | 15% | Major failure modes identified, mitigations concrete even if brief |
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
| 9-10 | Sub-systems well-chosen, core 2-3 explored with real depth, clear separation of concerns |
| 7-8 | Good sub-system choices, reasonable depth on core components |
| 5-6 | Sub-systems present but responsibilities unclear, or depth spent on wrong components |
| 3-4 | Monolithic or incoherent decomposition |
| 0-2 | No meaningful architecture |

### Tradeoff Reasoning (20%)
| Score | Indicators |
|-------|------------|
| 9-10 | Key decisions have alternatives rejected with reasoning, costs explicit |
| 7-8 | Major tradeoffs discussed for core sub-systems |
| 5-6 | Limited alternatives, shallow reasoning |
| 3-4 | Decisions stated without justification |
| 0-2 | No tradeoff reasoning |

### Failure Handling (15%)
| Score | Indicators |
|-------|------------|
| 9-10 | Major failure modes identified with concrete mitigations |
| 7-8 | Key failures named with reasonable mitigations |
| 5-6 | Basic awareness, generic mitigations |
| 3-4 | Minimal consideration |
| 0-2 | No failure handling |

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
