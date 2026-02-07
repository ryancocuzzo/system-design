# Changelog

## 2025-02-07 — Introduce micro-design track

After three full exercises (scores: 5, 4, 3), a consistent pattern emerged: time was being spent building out the wrong abstraction because the core problem was misidentified. The full exercises exposed this effectively but with a slow feedback loop — 90-125 minutes invested before learning the framing was wrong.

**What changed:**

- Added a **micro-design track** (`micro/`) — same problem complexity as full exercises, but with a phased 45-minute structure that forces problem framing and sub-system decomposition before any architecture work begins.
- Added a **context questions document** (`context.md`) per micro-design — a separate space for domain knowledge questions so that genuine knowledge gaps don't get conflated with design reasoning errors.
- Added `micro-rubric.md` with evaluation criteria weighted toward problem framing (25%) and communication (15%), reflecting the skills being trained.
- Added `/micro` command for generating micro-designs with scaffolded templates.
- Updated `/grade` to detect exercise type and apply the appropriate rubric.
- Updated `/review` to track both tracks, with framing accuracy as a primary metric for micro-designs.

**Why:**

The goal is to shorten the feedback loop on framing errors, build pattern recognition across more problem types in less time, and develop a repeatable process for approaching unfamiliar systems. Full exercises remain for testing end-to-end design ability without scaffolding.
