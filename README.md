# System Design Practice

Iterative system design practice with structured feedback loops. Two tracks — full exercises and micro-designs — plus rubrics, concept reference, and AI-assisted grading.

## Why This Exists

To get better at designing systems. Design against constraints, get graded against explicit criteria, revise based on what failed. The [CHANGELOG](CHANGELOG.md) documents the iterations — scope reduction, decision scaffolding, guided research — that emerged from real scores and failure modes.

## Problem Index

| Track | # | Problem |
|-------|---|---------|
| Full | 001 | Stripe Webhook Delivery Infrastructure |
| Full | 002 | CI/CD Orchestration Platform |
| Full | 003 | Global Edge Rate Limiting Service |
| Micro | 001 | Metrics and Logs Ingestion Pipeline |
| Micro | 002 | Shopify Multi-Tenant Storefront Backend |
| Micro | 003 | Multi-Tenant Search-as-a-Service |
| Micro | 004 | Container Image Registry |
| Micro | 005 | Infrastructure Run Orchestration Platform |
| Micro | 006 | URL Shortener Service |
| Micro | 007 | Instagram Photo Feed Service |

## Tracks

**Full exercises** — Open-ended. No scaffolding. Draft a complete architecture, then evaluate and revise.

**Micro-designs** — Scope-reduced. 40-minute time box. Phase 3 provides architectural decisions with named options; you pick, justify, and describe the mechanics.

```
exercises/          micro/
└── 001/            └── 001/
    ├── problem.md      ├── problem.md
    └── design.md       ├── context.md   # guided research
                        └── design.md
```

## Reference

- [concepts.md](concepts.md) — System design concept reference, organized by category
- [rubric.md](rubric.md) — Evaluation criteria for full exercises
- [micro-rubric.md](micro-rubric.md) — Evaluation criteria for micro-designs

## Progress

Micro-design scores (updated when you run `/grade` on a micro):

```mermaid
xychart-beta
    title "Micro Scores"
    x-axis [001, 002, 003, 004, 005, 006, 007]
    y-axis "Score" 0 --> 10
    line [3.5, 4, 4.5, 4, 5, 3.5, 4.5]
```

## Workflow (Cursor Commands)

- `/new` — Generate a full exercise
- `/micro` — Generate a micro-design
- `/grade` — Evaluate a design against the appropriate rubric
- `/review` — Progress report across both tracks
