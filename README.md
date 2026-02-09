# System Design Practice

Structured system design practice across two tracks: full exercises and micro-designs.

## Tracks

### Full Exercises

Open-ended system design. No scaffolding. Draft a complete architecture, then evaluate and revise.

```
exercises/
└── 001/
    ├── problem.md   # constraints and requirements
    └── design.md    # architecture, tradeoffs, review notes
```

### Micro-Designs

Scope-reduced problems with guided decision-making. 40-minute time box. Phase 3 provides architectural decisions with named options — you pick, justify, and describe the mechanics.

```
micro/
└── 001/
    ├── problem.md   # focused requirements, concept hints
    ├── context.md   # guided research prompts and questions during design
    └── design.md    # phased design with decision scaffolds and review notes
```

## Reference

- `concepts.md` — system design concept reference, organized by category
- `rubric.md` — evaluation criteria for full exercises
- `micro-rubric.md` — evaluation criteria for micro-designs

## Commands

- `/new` — generate a full exercise
- `/micro` — generate a micro-design
- `/grade` — evaluate a design against the appropriate rubric
- `/review` — progress report across both tracks
