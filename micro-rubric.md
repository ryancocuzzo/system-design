# Micro-Design Evaluation Rubric

Micro-designs are time-boxed (40 min total including research), guided exercises focused on problem framing, decision-making between concrete alternatives, and architectural reasoning. Problems are scope-reduced (3-4 FRs, 3-4 NFRs, 2-3 constraints) and Phase 3 provides named decision scaffolds with options.

## Scoring Scale

| Score | Meaning |
|-------|---------|
| 9–10 | Correct framing, right decisions with clear justification, mechanics well-described. |
| 7–8 | Correct framing and decisions with 1-2 gaps in justification or mechanics. |
| 5–6 | Partially correct. Some right choices, but reasoning weak or mechanics hand-waved. |
| 3–4 | Wrong framing or wrong decisions that conflict with stated constraints. |
| 0–2 | Did not complete or fundamentally off-target. |

## Criteria

| Criterion | Weight | Indicators |
|-----------|--------|------------|
| Problem Framing | 20% | Core challenge correctly identified, right mental model chosen, most important decision named |
| Key Decisions | 30% | Correct options chosen given constraints, justified with reasoning, mechanics described concretely |
| Tradeoff Reasoning | 20% | Justifications reference specific constraints/NFRs, rejected options explained, costs acknowledged |
| Stress Testing | 15% | Critical NFRs and failure modes triaged well, mitigations concrete |
| Communication | 15% | Phases structured clearly, reasoning traceable, incomplete sections explicitly marked |

## Per-Criterion Scoring

### Problem Framing (20%)
| Score | Indicators |
|-------|------------|
| 9-10 | Core challenge named precisely, framing category correct with explanation, identifies the decision that makes or breaks the design |
| 7-8 | Correct framing, most critical decisions identified |
| 5-6 | Partially correct framing, some confusion about what makes the problem hard |
| 3-4 | Wrong framing that leads decisions in wrong direction |
| 0-2 | No framing attempt or completely wrong problem |

### Key Decisions (30%)
| Score | Indicators |
|-------|------------|
| 9-10 | All decisions correct for the given constraints, justified by referencing specific requirements, mechanics described with enough detail to evaluate feasibility |
| 7-8 | Most decisions correct, justification present but could be sharper, mechanics mostly clear |
| 5-6 | Some decisions correct but at least one wrong or unjustified, mechanics vague |
| 3-4 | Decisions conflict with stated constraints, or chosen without reasoning |
| 0-2 | Decisions not made or fundamentally wrong |

### Tradeoff Reasoning (20%)
| Score | Indicators |
|-------|------------|
| 9-10 | Each decision references why rejected options don't fit, costs of chosen approach acknowledged |
| 7-8 | Reasoning present for most decisions, some costs acknowledged |
| 5-6 | Reasoning shallow or generic ("simplest" without explaining why simple matters here) |
| 3-4 | Decisions stated without justification |
| 0-2 | No tradeoff reasoning |

### Stress Testing (15%)
| Score | Indicators |
|-------|------------|
| 9-10 | 3-5 most critical concerns identified and triaged, mitigations concrete and specific to this system |
| 7-8 | Key concerns named with reasonable mitigations, good prioritization |
| 5-6 | Basic awareness, generic mitigations, or poor triage (addressed easy concerns, skipped hard ones) |
| 3-4 | Minimal consideration or only generic failures |
| 0-2 | No stress testing |

### Communication (15%)
| Score | Indicators |
|-------|------------|
| 9-10 | Phases clearly structured, reasoning explicit, incomplete areas honestly marked |
| 7-8 | Clear overall, minor gaps in reasoning trail |
| 5-6 | Understandable but structure breaks down in places |
| 3-4 | Difficult to follow or phases ignored |
| 0-2 | Cannot follow the reasoning |
