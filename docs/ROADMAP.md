# mise-en-place — Roadmap

A human-readable, checkable progress tracker for the recipe app. The **spec** lives in
`docs/plans/PLAN_recipe_app_foundation.md` (approved); this file is the **progress view** —
tick a box when that step is done and verified.

**How to use:** `[ ]` = not started · `[~]` = in progress · `[x]` = done & verified.
A task is **done** only when its code is written, its **"Done when"** check passes, and the
change is committed. Milestones close after a `workflow:review` pass at the boundary.

**Status:** ✅ Planning complete (GATE 0 passed 2026-08-24) · ⏳ Implementation not started.

Legend for the pills below: 🟢 done · 🟡 in progress · ⚪ not started.

---

## Phase 0 — Planning ✅ (complete)

- [x] **Understand** — intent + repo context gathered
- [x] **Plan written** — `docs/plans/PLAN_recipe_app_foundation.md`
- [x] **Recipe format + scope locked** — field list, browse/search scope, parser depth, LLM policy, milestone order
- [x] **Reviewer approval** — `approvals.reviewer: 2026-08-24`
- [x] **3 external reviews folded in** — converged 3→1→0 blocking (`docs/reviews/`)
- [x] **Human GATE 0 approval** — `approvals.human: 2026-08-24`, `status: approved`

---

## Milestone 1 — Backend scaffold, schema, DB, browse API ⚪

- [ ] **T1.1 — Toolchain + Drogon skeleton + migration runner**
  - Build: CMake + `vcpkg.json` (`drogon[postgres]`, `valijson`, `libcurl`, `re2`, Catch2)
  - `/health` endpoint; Postgres connection from env; `setThreadNum` sized
  - Migration runner: version-tracking, per-migration transaction, session `pg_advisory_lock` on one dedicated connection
  - Dev `docker-compose` with a Postgres service
  - **Done when:** `cmake --build` succeeds; app boots; `GET /health` → 200; logs show DB connect; re-run does not re-apply migrations; a failing migration leaves `schema_migrations` unchanged
- [ ] **T1.2 — `Recipe` JSON Schema + SQL migrations + model round-trip**
  - Draft-7 schema in `docs/` (single source of truth) + `definitions/Macros` sub-schema
  - Relational tables: `recipes` (+ macro cols, `macros_estimated`), `recipe_ingredients`, `recipe_steps`, `recipe_images`, `recipe_tags` — all child FKs `ON DELETE CASCADE`
  - C++ structs + jsoncpp assemble/emit canonical JSON + `valijson` validation
  - **Done when:** migrations apply on a fresh DB; round-trip test passes for a grouped/to-taste recipe, ordered + empty steps, tags/images order preserved, `macrosEstimated` surviving; valijson rejects invalid & accepts valid
- [ ] **T1.3 — Recipe repository/service (CRUD) + PUT full-replace**
  - CRUD across parent + child tables in a transaction; per-100 g derived
  - PUT = full-replace (client sends complete recipe; `food_id` picks survive; `position` reassigned; recompute `macrosEstimated`); image-file GC ordering (read names → delete → unlink)
  - **Done when:** integration tests (dedicated test Postgres, truncate per test) pass for CRUD, a PUT edit preserving picks + reordering, and the per-100 g "no weight → null" path
- [ ] **T1.4 — Browse controllers + seed**
  - `GET /api/recipes` (paginated summary projection) + `GET /api/recipes/:id` (full); seed 2–3 recipes with `food_id` null
  - **Done when:** integration test — list honors `limit`/`offset`, `/:id` returns schema-valid full `Recipe`
- [ ] **🚦 M1 review** — `workflow:review` at the milestone boundary passes

---

## Milestone 2 — Frontend scaffold, browse/detail, structured form ⚪

- [ ] **T2.1 — Angular scaffold + browse/detail**
  - Exact-pinned Angular; dev `proxy.conf.json`; typed API service coding against the error envelope; browse list (paginated summaries) + detail (macros, `macrosEstimated` marker on list & detail, `macroSource` badge)
  - **Done when:** `ng build` passes; `ng test` green (ChromeHeadlessNoSandbox); list + a detail render a seeded recipe with both macro columns
- [ ] **T2.2 — Add/Edit form + DELETE**
  - Reactive Forms (dynamic ingredient `FormArray`, steps, images, macros) → `POST`/`PUT /api/recipes/:id`; server-side validation (422 + envelope); `DELETE /api/recipes/:id` (→ 204) with UI confirm
  - **Done when:** valid submit persists & shows in browse; schema-invalid POST → 422 envelope; PUT preserves picked `food_id`s; delete removes recipe + child rows
- [ ] **T2.3 — Manual macro entry + override**
  - Wired into the form (no nutrition DB yet)
  - **Done when:** manually entered macros persist & render both columns; per-100 g shows "—" with no weight
- [ ] **🚦 M2 review** — boundary review passes _(M1+M2 = first end-to-end vertical slice)_

---

## Milestone 3 — Paste import with selectable engine ⚪

- [ ] **T3.1 — `RecipeParser` + `RuleBasedParser` + `POST /api/parse`**
  - Social captions only; primary offline engine; NFC-normalize → RE2, codepoint emoji stripping, line-anchored quantities; sections→group, qty/unit, macro block, to-taste rows, parenthetical→note, storage→notes
  - **Done when:** unit tests parse the 3 fixtures (grouping, null-qty rows, macro block, empty steps, `Eiweiß`/`Hähnchen`, an NFD variant, the `7%`-not-a-quantity case)
- [ ] **T3.2 — `LlmClient` + `LlmParser` (fallback engine)**
  - On the `IHttpClient` seam; Ollama native `/api/chat` `format`=schema (OpenAI fallback); validate + graceful fallback (empty/partial draft + warning)
  - **Done when:** unit test with fake `IHttpClient` — valid JSON accepted; malformed → documented fallback (draft + warning, no exception/save)
- [ ] **T3.3 — Paste screen**
  - Textarea + engine toggle → parse → editable preview (reuses M2 form) → save
  - **Done when:** pasting a sample (rule-based) yields a pre-filled, editable, saveable form
- [ ] **🚦 M3 review** — boundary review passes

---

## Milestone 4 — Macros from Open Food Facts (search & pick) + LLM estimate ⚪

- [ ] **T4.1 — `NutritionSource` + OFF client + `foods` cache + converter**
  - OFF API v2 `/api/v2/search` (verify live rate limits first), UA, 429/backoff, on `IHttpClient` seam; `foods` table (UUID `id` PK, `code` UNIQUE) + FK add onto `food_id`; local-first search; per-100 g mapping (kcal, else ÷4.184); unit→gram (density + piece/spoon tables)
  - **Done when:** fake-`IHttpClient` test returns candidates; already-picked food resolves with no API call; kJ-only mock converted/rejected; unit-conversion tests incl. piece/spoon + "no table entry → flagged"
- [ ] **T4.2 — Macro engine**
  - Sum `quantity × per-100 g` → per-serving + per-100 g; unpicked/missing/unresolvable surfaced (never zeroed) → per-100 g "—"; sets `macrosEstimated` when piece/spoon fed the math
  - **Done when:** exact-grams recipe matches hand-computed values (`estimated:false`); piece/spoon recipe computes full weight (`estimated:true`); flagged-ingredient recipe → "—"
- [ ] **T4.2b — `POST /api/macros/compute` + search-and-pick UI**
  - Per ingredient: search → candidates (prefer complete nutriments) → pick → macros fill; wired into the M2 form
  - **Done when:** in-form search lists candidates, picking fills macros, per-serving + per-100 g compute when all picked/resolved
- [ ] **T4.3 — `POST /api/macros/estimate` + LLM-estimate button**
  - `LlmMacroEstimator` (on the seam; validates against `definitions/Macros`); sets `macrosEstimated: true`
  - **Done when:** fake-LLM test fills macro fields & asserts `macrosEstimated: true`; user can override before save
- [ ] **🚦 M4 review** — boundary review passes

---

## Milestone 5 — Media, polish, search ⚪

- [ ] **T5.1 — Image upload + lifecycle**
  - Multipart → disk; server-generated filename, ≤8 MB, magic-byte type check; external URL store-only (no fetch/SSRF); owned files GC'd on delete + PUT diff
  - **Done when:** valid upload attaches/renders; oversized/wrong-type/path-traversal rejected; external URL never fetched; delete/PUT removes owned files, external untouched
- [ ] **T5.2 — Browse search / filter / sort**
  - Title search (German-aware case-insensitive); tag AND-filter; favorites toggle; `minProtein`+`maxCalories`; sort newest/title/protein; on the paginated list (hard-clamped `limit`)
  - **Done when:** search tests return expected subsets (incl. umlaut/ß case); each sort order confirmed; `limit`/`offset` paginate; over-max `limit` clamped
- [ ] **🚦 M5 review** — boundary review passes

---

## Milestone 6 — Deployment & docs ⚪

- [ ] **T6.1 — Docker + nginx + compose**
  - Multi-stage backend Dockerfile (vcpkg binary cache); Angular static bundle via nginx (`/api` proxy + `/uploads`); `docker-compose.yml` (backend, frontend, postgres volume; `LLM_BASE_URL`); `.env.example`
  - **Done when:** `docker compose up` builds & serves; browse works against a persisted volume; `/api/*` and `/uploads/*` reachable through nginx
- [ ] **T6.2 — Docs**
  - Env vars, Ollama pointer (verify exact pull tag), Postgres backup, C++/vcpkg + Angular build notes, min build RAM
  - **Done when:** the copy-pasteable sequence works: `docker compose up` → `curl /health` 200 → POST a recipe → it appears in `GET /api/recipes`
- [ ] **🚦 M6 review** — boundary review passes

---

## Definition of done (whole phase)

- [ ] All six milestones complete, each with its boundary review passed
- [ ] `docker compose up` runs the full app on a home server / VPS behind VPN or basic-auth **over TLS**
- [ ] Docs let a fresh setup reach a working app from scratch
- [ ] Plan + reviews archived to `docs/archive/` (per `docs/archive/README.md`)
