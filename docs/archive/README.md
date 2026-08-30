# docs/archive

Resting place for **superseded** planning artifacts, so `docs/plans/` and `docs/reviews/`
show only what is currently live.

## What goes here
- **Plans** (`PLAN_*.md`) whose work is **fully implemented and shipped**, or that were
  **replaced** by a newer plan. Move a plan here once its milestones are done (or it is
  abandoned), not while it is still being built against.
- **Reviews** (`PLAN_REVIEW_*.md`) belonging to a plan that has been archived. Keep a
  review next to its plan while that plan is active — the reviews document how the live
  plan was hardened, so they stay in `docs/reviews/` until the plan itself moves.

## What does NOT go here
- The **active** plan and its reviews (an approved-but-not-yet-implemented plan is still
  active — its reviews are current provenance, not history).
- Handoffs (`docs/handoff/`) and references (`docs/reference/`) — those track ongoing
  state, not a finished plan.

## How to archive (suggested)
When a plan is done or replaced, move it and its reviews together and leave a one-line
pointer here so the trail is findable:

```
git mv docs/plans/PLAN_<slug>.md docs/archive/
git mv docs/reviews/PLAN_REVIEW_*<slug-or-date>*.md docs/archive/
```

Then add a line to the index below.

## Archived index
_(newest first — empty until the first plan is implemented/superseded)_

- _nothing archived yet_
