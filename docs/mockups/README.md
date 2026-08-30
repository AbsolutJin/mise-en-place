# docs/mockups

UI mockups for the recipe app's screens — designed **before** the Angular build (Milestone
2) so layout and interaction decisions are made on paper, not mid-code. These are visual
references, not the spec; the canonical field list and behaviour live in
`docs/plans/PLAN_recipe_app_foundation.md`.

## What goes here
- One file per screen (any format: `.dc.html` design-canvas, exported PNG/SVG, or a
  wireframe image). Name by screen, e.g. `browse-list.png`, `recipe-detail.png`.
- Keep it lightweight — enough to agree on layout, states, and controls.

## Screens to cover (from the plan)
- [ ] **Browse / list** — recipe cards from the paginated summary (title, image, favorite,
  per-serving macros, tags); the **"estimated"** marker on cards; search/filter/sort bar
  (title search, tag AND-filter, favorites toggle, `minProtein`/`maxCalories`, sort)
- [ ] **Recipe detail** — image(s), source link, ingredients (grouped by section), steps,
  both macro tables (per-serving + per-100 g, with "—" and "estimated" states), the
  **`macroSource` badge** (ingredients / llm / manual), favorite, notes
- [ ] **Add / Edit form** — Reactive-Forms layout: title/description/servings/weight,
  dynamic ingredient rows (name, qty, unit, group, note, per-row food pick), steps, tags,
  images, macro fields; manual macro entry/override; delete-with-confirm
- [ ] **Paste import** — textarea, engine toggle (rule-based / local LLM), parse →
  **editable preview** (reuses the form), warning banner for the LLM fallback state
- [ ] **Ingredient search-and-pick (OFF)** — per-ingredient search box, candidate list
  (prefer complete nutriments / nutrition grade), pick → macros fill
- [ ] **Empty / error / loading states** — empty browse, an error-envelope message, a
  `200`+warning (cached-only / degraded) banner

## States worth showing (easy to forget)
per-100 g `—`, `macrosEstimated` marker, unpicked/flagged ingredient, "to-taste" rows
(null qty/unit), grouped ingredient sections, a down-LLM warning draft.

## Index
_(add a line per mockup as it lands)_

- _nothing yet_
