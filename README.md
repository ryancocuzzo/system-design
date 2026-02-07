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

Same problem complexity, but with structured phases and a 45-minute time box. Focused on building problem framing skill and systematic decomposition.

```
micro/
└── 001/
    ├── problem.md   # constraints, requirements, and framing scaffolding
    ├── context.md   # domain knowledge questions, separate from design
    └── design.md    # phased architecture with review notes
```

## Evaluation

- `rubric.md` — criteria for full exercises
- `micro-rubric.md` — criteria for micro-designs (weighted toward problem framing)

## Commands

- `/new` — generate a full exercise
- `/micro` — generate a micro-design
- `/grade` — evaluate a design against the appropriate rubric
- `/review` — progress report across both tracks
