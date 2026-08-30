# Plan Review (second external) — `PLAN_recipe_app_foundation.md`

**Date:** 2026-08-24
**Scope:** Round-8 revision of the plan, after the first external review (`PLAN_REVIEW_2026-08-24_external.md`) was folded in and the workflow reviewer re-ran (Rounds 6–8).
**Method:** Two independent passes (one primary, one blind cross-check that received only the current plan + fixtures, not the primary's reasoning). The "→ Fixed" claims in the plan's Reviewer notes were verified against the plan **body**, not taken at face value. Key facts (pg advisory-lock semantics, OFF Search-a-licious vs. API v2, RE2 Unicode behaviour) were checked before reporting.

---

## Verdict: CHANGES REQUIRED — 1 blocking + 5 major, but the substance is sound

The Round 6/7 fold-in is **real and mostly solid** — both passes independently confirmed the items marked "Fixed" (food_id/foods FK sequencing, `foods.id` UUID PK, `group_label`, the `DELETE` endpoint, the T4.3 fake-seam verify) are genuinely present in the body, not merely asserted in the notes. But this review finds one blocker and five major issues — and **two were introduced by the Round 6 fix itself**. Round 8's "APPROVE, no new inconsistency" was again over-confident (same pattern as the first external review).

---

## Confirmed genuinely fixed in the body (credit where due)

- `food_id` sequencing: T1.2 creates a plain nullable UUID column (no FK) → T4.1 `ADD CONSTRAINT` onto the existing column, no duplicate. ✓
- `foods.id` surrogate UUID PK = `foodId`, `code` as a `UNIQUE` column. ✓
- `group_label` rename (JSON field stays `group`). ✓
- `DELETE /api/recipes/:id` added to T2.2 (with UI confirm + test). ✓
- T4.3 verifies via the fake seam. ✓

---

## Blocking

### PUT child-table update semantics are undefined
The relational split (`recipes` + `recipe_ingredients` / `recipe_steps` / `recipe_images` + `recipe_tags`) makes UPDATE non-trivial, but no task specifies how. T2.2 exposes `PUT /api/recipes`; T1.3 only says "update … across parent + child in a transaction." Delete-and-reinsert vs. diff is left open — and it is not safely guessable: delete-and-reinsert of `recipe_ingredients` **destroys the `food_id` picks** unless the client round-trips every `food_id` back, and it churns `position`. Because it changes both behaviour and persisted data integrity, it violates the self-contained requirement. Pin the update strategy and state explicitly whether `food_id` links survive an edit.

---

## Major

### 1. Piece/spoon approximations produce precise-looking macros with no confidence signal *(introduced by the Round 6 fix)*
The M2 piece-weight / spoon-volume table (clove ≈ 5 g, onion ≈ 150 g, egg ≈ 60 g, TL ≈ 5 ml …) feeds **both** the denominator (`totalWeightG`) and the numerator (each ingredient's grams → `quantity × per-100 g`). For the fixtures — dominated by piece/spoon units — the displayed per-serving macros *and* per-100 g are now approximate, yet rendered as exact numbers. The revision deliberately removed the honest "—" (D5: "the '—' is the honest last-resort, not the normal case") **without** introducing an "approximate/estimated" flag to replace it. The old design was honest-by-omission; the new one is confidently wrong. T4.2's "±5 % tolerance" test is tautological — it compares against the same table constants — so it does not catch this. Add a per-recipe "macros include approximate piece/spoon weights" indicator (and mark the derived per-100 g as estimated) instead of presenting guessed weights as exact.

### 2. OFF endpoint / service pairing is factually wrong
D5 and T4.1 pin "Search-a-licious (`/api/v2/search`)" as the primary path. These are two different things: **Search-a-licious** is the newer OFF search service (its own host `search.openfoodfacts.org`, its own `/search`-style endpoint), whereas **`/api/v2/search`** is the classic OFF API v2 search path — the kind of endpoint Search-a-licious is meant to replace. A dev would build the "primary" client against a path that doesn't belong to the named service. T4.1's verify only directs confirming **rate limits** against live docs, not endpoint/service correctness, so the mis-pin passes no gate.
- **Related (per-piece weight):** D5 also says the converter uses "OFF `serving_size` / `product_quantity` … as a per-piece weight." `product_quantity` is the **package** quantity (e.g. 500 g for a bag of rice), **not** a per-piece weight — reading it as per-piece is badly wrong. Another correctness slip introduced with the M2 fix.

### 3. `pg_advisory_lock` is unsound with Drogon's connection pool *(the Q7 "fix" doesn't hold)*
D2/T1.1 present the advisory lock "around the run" as the fix for concurrent-boot races. Session-level `pg_advisory_lock` is bound to **one physical connection**, but `DbClient` / `execSqlSync` draws from a pool and the lock is held "around the run" spanning multiple per-migration transactions. Lock, migrations, and unlock can land on different pooled connections → the lock either never serializes or leaks. Use `pg_advisory_xact_lock` inside a single wrapping transaction, or a dedicated single connection for the whole runner. As written, the mechanism it claims to have solved does not actually work.

### 4. API error contract (response shape + HTTP status codes) is completely unspecified
No task defines the error-response format or status codes for any endpoint. T2.2 says only "invalid input is rejected with a message." Undefined: 404 for `GET /api/recipes/:id` on a missing id; 400/422 for schema-invalid POST/PUT; the behaviour of `/api/parse`, `/api/macros/estimate`, `/api/foods/search` when the LLM/OFF is unreachable (D6 promises "graceful degradation" and "a clear message" but never defines the JSON shape or status); 429 pass-through from OFF. A fresh dev must invent the whole error surface, and the typed frontend service in T2.1 has nothing to code against. Define one consistent error envelope + a status-code table.

### 5. `GET /api/recipes` has no list-vs-detail distinction and no pagination
T1.4's verify expects the list endpoint to return full canonical `Recipe` objects, each assembled from all child tables. There is no summary projection, no `limit`/`offset`, no pagination anywhere (T5.2 adds filter/sort but still no paging). For a growing personal store this is an assemble-all-children scan on every browse. State whether list returns summaries or full objects and how it bounds result size.

---

## Minor

- **`IHttpClient` rename missed in T3.2's verify** *(half-applied fix).* Rounds 7/8 assert the seam was renamed `IHttpClient` "everywhere (D3, T3.2, T4.1, T4.3)." Body check: D3 ✓, T4.1 ✓, T4.3 ✓ — but T3.2's verify line still reads "fake `HttpClient`" (the un-prefixed name that collides with Drogon's class — the exact thing the rename existed to fix).
- **Unicode normalization (NFC/NFD) unaddressed** *(genuine blind spot, no round raised it).* B3 correctly moves to RE2 for UTF-8 but assumes a single byte form. Pasted captions may arrive NFD-decomposed (`ä` = `a` + U+0308), so whole-token matches for `Eiweiß`/`Hähnchen`/`Kohlenhydrate` and the codepoint-range emoji stripping can silently miss. NFC-normalize input first. The same gap undermines T5.2's "case-insensitive" German title search (ß/umlaut folding depends on collation).
- **`macroSource` cannot represent mixed provenance.** One enum per recipe (`ingredients | llm | manual`), but the search-and-pick flow naturally yields recipes where some ingredients are computed and one macro is hand-overridden (D5: "always overridable"). The single-value field can't express that, so stored provenance is misleading for the common edited case.
- **T1.2 round-trip test omits `tags` and `images`; tag order not preserved.** The verify covers grouped/to-taste ingredients and ordered/empty steps but never `tags`/`images`. `recipe_tags` has no `position`, so `tags[]` order is not preserved across a round-trip, and nothing tests it.
- **FK-add in T4.1 assumes no orphan `food_id`.** The `ALTER … ADD CONSTRAINT` succeeds only if no `recipe_ingredients` row already holds a `food_id` absent from the freshly-created (empty) `foods` table, so the T1.4 seed recipes must leave `food_id` null — note it explicitly. Related: `ON DELETE CASCADE` for the child tables is not specified in the T1.2 DDL.
- **Nit:** JSON field `macrosPerServing.calories` ↔ DB column `cal` — harmless, but the assembly code must map the name; not stated.

---

## Bottom line

The plan is clearly stronger than the Round 5 state, and the Round 7 fixes are real. But it is not approval-ready: one blocker (PUT semantics) and five major issues, **two of them introduced by the last fix** (piece-weight honesty; the advisory lock). This is exactly why every substantive change needs a fresh independent pass — a fix can give birth to a new problem.

Meta-point, now with two data points: **twice in a row** an external pass found blockers *after* the workflow `plan-reviewer` stamped APPROVE. That is no longer a coincidence — the workflow reviewer checks structure/consistency well but is weak on implementability depth and fact-checking. If the `claude-tools` workflow continues in use, extend its reviewer prompt on both axes: fact-verification against live docs, and a "could a fresh dev build this from the plan alone?" depth pass.
