# Changelog

## 2026-02-08 — Add guided research and tighten micro-design time budget

After the first micro-design (micro/001, score: 3.5/10), the phased structure worked — the process was followed — but the design still missed because critical building blocks weren't in vocabulary. You can't pick a time-series database over Postgres if you've never worked closely with one. The current scaffolding addressed the process gap but not the knowledge gap.

**What changed:**

- Added `concepts.md` — a system design concept reference organized by category (storage, caching, messaging, reliability, etc.) for use during any exercise.
- Micro-design problems now include a **"Concepts to Explore"** section: 3 concept areas relevant to the problem, drawn from `concepts.md`. These name the category to research without giving away which tool to use or how.
- `context.md` now starts with **3 guided research prompts** matching the concept hints. Each is a question to research before designing.
- The design template now requires **naming 2+ approaches** before choosing one in the deep dive, preventing default to the first familiar tool.
- **6 design phases compressed to 4** by merging sub-systems + integration and merging NFRs + failure modes into a triaged "stress test."
- **Total time budget is 50 minutes** including research. No separate clock — everything fits in one box.
- Grading now explicitly assesses whether guided research translated into design decisions.

**Why:**

The goal is to close the loop between "concepts I've heard of" and "concepts I can apply during a design." The research phase builds vocabulary; the alternatives prompt in the deep dive forces that vocabulary into decisions. The tighter time budget and fewer phases reduce overhead and force triage over exhaustiveness.

---

## 2026-02-07 — Introduce micro-design track

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
