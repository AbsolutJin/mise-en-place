# Plan Review (third external) — `PLAN_recipe_app_foundation.md`

**Date:** 2026-08-24
**Scope:** Round-10 revision of the plan, after the second external review (`PLAN_REVIEW_2026-08-24_external-2.md`) was folded in and the workflow reviewer re-ran (Rounds 9–10, currently APPROVE).
**Method:** Two independent passes (one primary, one blind cross-check that received only the current plan + fixtures, not the primary's reasoning). Each focus area was traced through the plan **body** (D1–D6 + Milestone tasks), not the Reviewer notes; the "→ Fixed" claims were verified against the body. Key mechanisms (session-level advisory-lock semantics, OFF v2 vs. Search-a-licious, the `macrosEstimated` data path) were checked before reporting.

---

## Verdict: APPROVE with findings — no blocking defect

For the first time in this chain, the independent pass also finds **no blocker**. The major recent fixes are genuinely present in the body and technically sound. One **Major** remains that should be decided/fixed before human sign-off, plus a set of minor coherence / implementability gaps. This is a mature plan, not a wobbly one — the review loop has converged (Review 1: 3 blocking + 6 major; Review 2: 1 blocking + 5 major; Review 3: 0 blocking + 1 major).

---

## Confirmed sound in the body (both passes, no finding)

- **Migration lock — now correct.** Session-level `pg_advisory_lock` on **one dedicated** connection, held across all per-migration transactions, released at the end. Correct semantics (session locks survive `COMMIT`s on the same connection). D2 explicitly rejects both the wrapping transaction (wrong all-or-nothing granularity) and `pg_advisory_xact_lock` (releases at each commit, reopening the between-migrations race). No contradiction between "one connection" and "each migration in its own transaction."
- **`macrosEstimated`** is consistent across format → `macros_estimated` column → assembly → T1.2 round-trip test → T4.2 compute → D1 summary projection — **for the ingredients path** (the gap is the LLM path, see Major below).
- **PUT full-replace** is sound for a single-user app: the payload carries every `foodId` so picks survive; `position` from array order. Lost-update/concurrency is a non-issue at this scale.
- **OFF endpoint** (v2 `/api/v2/search` primary, Search-a-licious as a separate future service) is consistent in the body.
- **Pagination projection** is well-defined and consistent across D1/T1.4/T5.2.

---

## Major (decide before sign-off)

### The `macrosEstimated` honesty flag does not cover the LLM-estimate path (T4.3)
The flag exists to stop guessed numbers being shown as exact (2nd-review Major 1). It is set **only** "whenever a piece/spoon-table (or serving_size) weight fed the math" (D5, format, T4.2). **T4.3** (`/api/macros/estimate`, "Estimate with local LLM") never sets it — even though LLM-estimated macros + an LLM-estimated total weight are the *most* approximate values in the system. Compounding it: the detail UI (T2.1) renders the "estimated" marker **only** from `macrosEstimated` and shows **no** `macroSource` badge. So a saved LLM-estimated recipe reloads with `macrosEstimated=false`, no provenance badge, indistinguishable from exact ingredient-computed macros — exactly the dishonesty Major 1 set out to prevent, reintroduced for the LLM path. The field name "macrosEstimated" also implies it should be true here.
**Decision needed:** set the flag on the LLM path too, **or** surface `macroSource` in the UI as a badge (ideally both — trivial). This is a product/honesty call, not a mechanical coder fix.

---

## Minor (polish; none blocking)

1. **Success-path `warning` shape is unpinned.** The error envelope `{error:{code,message,details}}` is pinned, but three degradation flows return `200 + a warning` with no defined warning payload: `/api/foods/search` cached-only (D1), the parse LLM-invalid fallback (T3.2), and the LLM-macro fallback (T4.3). The typed frontend must code against these too, which partly undercuts the "one shape to code against" goal.
2. **429-vs-502 no-cache branch is ambiguously worded.** D1: "surfaces `429` (or `502` when nothing is cached) only when it cannot." Both branches read as the no-cache case, so which status is returned when OFF responds 429 *and* nothing is cached is unclear.
3. **The "macro body" JSON Schema is referenced but never authored.** D1 (`422` on "schema-invalid Recipe/**macro body**"), D3 (Ollama `format` = "Recipe/**macro** schema"), and `/api/macros/estimate` + `/api/macros/compute` all validate a macro body — but T1.2 declares only the **Recipe** schema as source of truth. The separate macro-body schema is never listed as an authored artifact, so the macro-validation and 422 paths have no defined schema to validate against.
4. **The list page doesn't render the flag its projection was widened for.** Round 9 added `macrosEstimated` to the D1 summary projection "so the browse list flags it too," but T2.1 attaches the "estimated" marker to the **detail** page only; the list page is "consuming the paginated summary list" with no marker rendering tasked.
5. **Pagination is a *default* cap, not a *max* clamp.** D1 says "a sane default cap, e.g. 50" and justifies it as bounding the browse query as the store grows — but a default without an enforced maximum means `?limit=100000` defeats the bound. The stated rationale isn't guaranteed by a default-only cap.
6. **Uploaded image files on disk are never garbage-collected** *(blind spot, unraised in any round).* `ON DELETE CASCADE` and PUT full-replace remove `recipe_images` **rows**; nothing deletes the underlying **files** in the uploads volume (T5.1). Deleted recipes and removed/replaced images leak files on disk indefinitely. Low impact for single-user, but open.
7. **`macrosEstimated` × `macroSource`-precedence interaction on manual override is unspecified.** If macros are computed with the piece/spoon table (`macrosEstimated=true`) and the user then hand-edits one macro or `totalWeightG` (flipping `macroSource` to "manual"), whether `macrosEstimated` is cleared is undefined. On PUT the backend stores the client-provided flag with no stated recompute/trust rule, so its value after a mixed compute-then-override is implementation-dependent.
8. **A stale "Fixed" note contradicts (and misleads about) the current body.** The 2nd-review Major-3 resolution line still claims "`pg_advisory_xact_lock` in one wrapping txn." The current body uses a **session-level `pg_advisory_lock`, no wrapping transaction** — and Round 9 explicitly replaced the xact-lock phrasing. The body is the correct version; the stale note is the defect (it actively misleads a reader who trusts the resolution notes).

---

## Recommendation — this is the exit point

The loop has converged: 3 blocking + 6 major (R1) → 1 blocking + 5 major (R2) → 0 blocking + 1 major (R3). The single substantive point is a cheap, well-scoped honesty fix that is a human decision anyway. A fourth review would be diminishing-to-zero returns.

Concretely: **(1)** decide the Major (flag the LLM path *and/or* surface `macroSource` — trivial); **(2)** optionally fold the minors in one pass (especially #3 macro schema and #6 file GC — the two real implementability gaps); **(3)** then go to human GATE 0. The two open sign-off questions (auth-seam default; six-milestone scope) and the VPN/TLS caveat remain in the plan as the final human decisions.

---

## Concrete plan-text changes (paste-ready fold-in)

These are proposed edits to `PLAN_recipe_app_foundation.md` that resolve every finding above. The Major needs a human decision on *which* option; the rest are mechanical.

### Major — `macrosEstimated` must cover every approximate path (human decides scope)

**Recommended:** do both — widen `macrosEstimated` to all non-exact paths *and* surface `macroSource` in the UI. They are cheap and cover each other.

- **`Recipe` format comment** (the `macrosEstimated` field) — broaden the definition:
  > `macrosEstimated`: TRUE whenever the macros do **not** rest on exact per-100 g × gram-resolved-weight math — i.e. any of: an approximate piece/spoon-table (or `serving_size`) weight fed the math (M2), **or** the macros came from the LLM estimator (T4.3). FALSE only for all-exact-grams ingredient computation and for manual entry the user asserts as exact. The UI marks estimated macros accordingly rather than showing them as exact.
- **D5, "Honesty of approximate macros"** — add a sentence:
  > The same flag is set on the **LLM-estimate path** (T4.3): LLM-estimated macros and total weight are approximate by nature, so `macrosEstimated: true` and the UI marks them estimated after save/reload, not only during entry.
- **T4.3** — append to the task: `… fills macro fields for user review **and sets `macrosEstimated: true`** (LLM output is an estimate).` And to its *Verify:* `… asserts the filled recipe carries `macrosEstimated: true`.`
- **T2.1** — extend the detail/list rendering: `… showing the "estimated" marker when `macrosEstimated`, **and a small `macroSource` badge (ingredients / llm / manual) on the detail page** so provenance survives reload.` (Resolves Minor #4 as well — see below.)
- **`macroSource` × `macrosEstimated` on manual override (Minor #7)** — add to the format precedence comment:
  > A manual edit flips `macroSource` → "manual" **and clears `macrosEstimated` to false** (the user is asserting the edited values); a later recompute from ingredients/LLM re-derives both. On PUT the backend **recomputes** `macrosEstimated` from the payload's ingredient/weight resolution rather than trusting the client-sent flag blindly.

### Minor #3 — author the macro-body schema

- **T1.2** — after "author the `Recipe` JSON Schema", add:
  > Author the macro body as a **named sub-schema `$defs/Macros`** inside the same Draft-7 schema document (the `{calories,protein,carbs,fat}` shape), referenced by `macrosPerServing`. `POST /api/macros/compute` and `POST /api/macros/estimate` validate their request/response macro bodies against `$defs/Macros`; the Ollama `format` for the macro estimator (D3) uses the same sub-schema. This gives the D1 `422` path and every "schema-validated" macro step a single authored schema — no second source of truth.

### Minor #6 — garbage-collect uploaded image files

- **T5.1** — add:
  > **Backend-owned files are lifecycle-managed.** The uploads service owns the files it wrote (UUID-named). External-URL images are store-only (no file). On recipe **delete** and on the **PUT full-replace** image diff, the service **unlinks any owned file no longer referenced** by a `recipe_images` row.
- **T1.3 / T2.2** — note the ordering subtlety (so `ON DELETE CASCADE` doesn't hide the file list):
  > Because `ON DELETE CASCADE` removes `recipe_images` rows, the service **reads the owned image filenames first**, then deletes the recipe, then unlinks the files (best-effort, logged on failure — an orphaned file is a warning, never a failed request). On PUT, diff old vs. new image URLs within the replace transaction and unlink the dropped owned files after commit.
- **T5.1 / T2.2 *Verify:*** add: `deleting a recipe (and replacing an image via PUT) removes the corresponding owned file from the uploads volume; an external-URL image is never touched.`

### Minor #1 — pin the success-path `warning` shape

- **D1, API conventions** — add a third bullet next to the error envelope:
  > **Warning envelope (success path):** a `200` that carries a soft warning uses `{ "data": <payload>, "warning": { "code": "<slug>", "message": "<text>" } }` — one shape for all three degradation flows (`/api/foods/search` cached-only; the parse LLM-invalid draft, T3.2; the LLM-macro fallback, T4.3). Absent `warning` ⇒ a clean result.

### Minor #2 — disambiguate 429 vs 502

- **D1** — replace the parenthetical with:
  > `/api/foods/search` under OFF throttling: if any local candidate exists → `200` + cached results + `warning`. Otherwise, propagate the upstream condition: OFF returned `429` → **`429`**; OFF unreachable/timeout → **`502`/`504`**.

### Minor #4 — render the flag the list projection carries

Covered by the T2.1 edit under the Major (add the estimated marker to the **list** page too, not only detail — the projection already carries `macrosEstimated` for this).

### Minor #5 — make pagination a hard clamp

- **D1** — change "a sane default cap, e.g. 50" to:
  > `limit` defaults to 50 **and is clamped to a hard maximum (e.g. 100)** server-side, so a large client `limit` cannot defeat the bound.
- **T5.2 *Verify:*** add `a `limit` above the max is clamped, not honored verbatim.`

### Minor #8 — delete the stale resolution note

- **Reviewer notes, 2nd-external-review Major-3 line** — replace the "`pg_advisory_xact_lock` in one wrapping txn" wording with the actual mechanism: "session-level `pg_advisory_lock` on a dedicated connection held across the per-migration transactions (no wrapping transaction; superseded by Round 9)." The body (D2/T1.1) is already correct; only this historical note misleads.
