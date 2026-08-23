---
plan: recipe_app_foundation
status: draft
approvals:
  reviewer: 2026-08-23   # date (YYYY-MM-DD) on reviewer APPROVE (round 3 confirm, OFF design)
  human: pending      # date (YYYY-MM-DD) on human approval
---

# Plan — mise-en-place recipe app (foundation)

## Goal
Build a **single-user, self-hosted** web app to store, browse, and add recipes.
Recipes are added via a **structured form** or by **pasting free text** (TikTok /
Instagram caption, website text), which is normalised into one **standardised
recipe format**. Recipes render with optional image(s), an optional source link,
and **macros shown both per portion and per 100 g**. Macros are **auto-computed
from ingredients** via a nutrition database, with a **button to LLM-estimate**
instead. Paste-parsing and macro-estimation each offer a **user-selectable engine:
rule-based or a local LLM** (HTTP API, e.g. Ollama).

This plan covers the **product foundation** (all core features end-to-end) across
six milestones. It is the first workflow phase; later phases (advanced search,
tagging, meal planning, authentication) are out of scope here — but the schema is
built **auth-ready** so auth is a later addition, not a migration.

## Locked design decisions
> These are the decisions agreed with the human at GATE 0. A **C++ backend was
> explicitly chosen to learn C++** — this is a deliberate trade of extra plumbing
> for that goal, and the plan is sequenced to introduce the C++/Drogon stack
> milestone by milestone.

### D1 — Architecture: split frontend + C++ backend (monorepo)
- **Backend:** **C++17/20 with the [Drogon](https://github.com/drogonframework/drogon)
  web framework** — HTTP routing (controllers), an async **PostgreSQL** driver, JSON
  (jsoncpp), and multipart file uploads, all built in. Chosen so the learning effort
  goes into recipe logic, not raw sockets/DB plumbing.
- **Frontend:** **Angular SPA** (Angular CLI, `ng build` → static bundle), served as
  static files; it calls the backend over HTTP at `/api/*`. Chosen to learn a robust,
  structured framework; its **Reactive Forms** suit the dynamic ingredient-row form and
  editable paste-preview particularly well. **Pin the Angular major version** (v17+
  application builder) up front — it affects builder/test defaults and the proxy config.
- **Same-origin strategy (no CORS):** in **dev**, `ng serve` (:4200) proxies `/api` →
  the Drogon backend via `frontend/proxy.conf.json` (`ng serve --proxy-config`); in
  **prod**, nginx serves the static bundle and **reverse-proxies `/api/*`** to the
  backend (see T6.1). This keeps calls same-origin, so no browser CORS is needed. If any
  cross-origin path is later introduced, add a Drogon CORS filter/advice.
- **Monorepo layout:**
  ```
  /backend    C++ Drogon API (CMake, vcpkg)
    /controllers  /models  /services  /data  /migrations  /tests
  /frontend   Angular SPA
  /docs
  docker-compose.yml
  ```
- **Trade-off accepted:** two build systems and two deploys; the `Recipe` type is
  **not auto-shared** across C++/TS — mitigated by D2 (a single JSON-Schema source of
  truth in `docs/`, validated on both sides).

### D2 — Storage: PostgreSQL, with JSON as the interchange format
- **PostgreSQL** via Drogon's `DbClient`. Chosen over SQLite because the human intends
  to add **auth / multi-user later**, where Postgres is the sturdier base; the extra
  container is cheap under Docker Compose. **DB access uses the synchronous
  `execSqlSync` style** (run off the event loop) — chosen over coroutines/callbacks to
  keep it approachable for a C++ beginner; standardized in docs.
- **vcpkg note:** Drogon must be pulled with the **`postgres` feature** enabled
  (`"drogon": { "features": ["postgres"] }` in `vcpkg.json`) — the default port has no
  libpq backend and `DbClient` for Postgres won't link without it. First build compiles
  Drogon's dependency tree and is slow.
- **JSON Schema validation:** jsoncpp (bundled with Drogon) only parses/serializes, so a
  dedicated validator is required — **`valijson`** (header-only, in vcpkg, has a jsoncpp
  adapter). This backs every "schema-validated" step (T1.2, T3.2, T4.3).
- **Migrations:** plain **SQL files** in `/backend/migrations`, applied by a small
  **idempotent runner** on startup that records applied versions in a `schema_migrations`
  table (so re-running on boot is safe) — no ORM code-gen magic, easy to read while
  learning.
- **Auth-ready schema (no auth implemented this phase):** a `users` table stub and a
  **nullable `owner_id`** FK on `recipes`. Nothing enforces it yet; it exists so auth
  is a later addition, not a schema migration of live data.
- A **`foods` table** caches Open Food Facts entries the user has picked, keyed by the OFF
  **barcode `code`** (unique), storing name, per-100 g macros (kcal/protein/carbs/fat),
  `lang`, and `fetched_at`, so repeat lookups need no network (D5). Recipe ingredients
  reference it via a nullable `food_id`.
- **Standardised recipe format (`Recipe`)** is defined once as a **JSON Schema in
  `docs/`** (the single source of truth), mirrored by a C++ struct (jsoncpp
  serialization) and a TS type. It is validated at every API boundary, on paste-parser
  output, on LLM output, and used for export. So "everything becomes standardised
  JSON" holds at the API/interchange layer; Postgres is the persistence detail. The format
  carries a **`schemaVersion`** so future format changes are migratable. (A YAML
  import/export convenience could be layered on later; JSON stays canonical.)

### D3 — Local-LLM integration (pluggable, OpenAI-compatible)
- A small C++ `LlmClient` using **libcurl (synchronous)** — chosen over Drogon's
  event-loop-bound `HttpClient` because a plain blocking POST is simpler to call from a
  request handler and easier for a beginner (note: HTTPS to a remote LLM pulls in
  OpenSSL). It calls an **OpenAI-compatible chat endpoint**. Config via env:
  `LLM_BASE_URL` (server root, default `http://localhost:11434`), `LLM_MODEL`, optional
  `LLM_API_KEY`. The client appends **`/v1/chat/completions`** (Ollama's OpenAI-compatible
  surface — most portable; not the native `/api/chat`). JSON reliability uses the
  server's structured-output / `response_format` support, not prompt-only coercion, and
  every response is **validated with `valijson` against the `Recipe`/macro schema** with
  graceful fallback.
- LLM is **off unless the engine toggle selects it**, so the app is fully usable with
  no LLM running.

### D4 — Paste parsing: two engines behind one interface
- `RecipeParser` interface with two C++ implementations:
  - `RuleBasedParser`: heuristics (line splitting, quantity/unit regex,
    "Ingredients"/"Instructions" section detection, hashtag/emoji stripping). Offline,
    free, deterministic.
  - `LlmParser`: sends pasted text + the `Recipe` JSON Schema to the local LLM, asks
    for schema-valid JSON, validates it.
- **UI:** on the paste screen the user picks the engine, sees the parsed result in an
  **editable preview form** (reuses the M2 form), corrects anything, then saves.
  Parsing never saves directly — the human always confirms.

### D5 — Macros: from-ingredients (search & pick), LLM-estimate, or manual (always overridable)
- **Nutrition source: Open Food Facts (OFF)** behind a `NutritionSource` interface.
  Chosen over USDA because the user cooks from **German** recipes: OFF is multilingual
  with strong German-language coverage and a text-search API, whereas USDA is
  English-only/US-centric (its *numbers* are universal but its *names* won't match German
  ingredients, and German-specific foods like Quark/Schmand are absent). The interface
  leaves a seam to add USDA (clean generic whole-food values) as a secondary source later.
  OFF is **ODbL**-licensed — fine for personal self-hosted use; a small "Data from Open
  Food Facts (ODbL)" attribution line is shown in the UI.
- **Search-and-pick per ingredient (human in the loop):** rather than auto-guessing a
  match (the previous plan's biggest accuracy risk), the user **searches OFF for each
  ingredient and picks the right food**; `quantity × per-100 g` → macros. This removes
  the risky fuzzy auto-match.
- **OFF API contract (pinned):** text search via `/cgi/search.pl?search_terms=…&json=1`
  (or Search-a-licious); product reads by barcode. OFF **mandates a descriptive
  `User-Agent`** (e.g. `mise-en-place/0.1 (contact)`) — the default libcurl UA is
  throttled/blocked — and enforces **rate limits** (~10 req/min search, ~100 req/min
  product) returning 429. The client sets the UA and handles 429 with backoff.
- **Per-100 g nutrient mapping (pinned — avoids the kJ/kcal trap):** read
  `energy-kcal_100g` for calories (fall back to `energy_100g ÷ 4.184`, since `energy_100g`
  is kJ), and `proteins_100g` / `carbohydrates_100g` / `fat_100g`. **Products missing the
  `*_100g` nutriments are not pickable** and are flagged — never zero-filled. (~half of
  OFF products lack complete nutrition data.)
- **Cache: search local first, OFF only when insufficient.** Search matches the local
  **`foods` table** first and calls OFF only when local candidates are thin; every
  **picked** food is persisted (keyed by OFF **barcode `code`**, unique, with `lang` +
  `fetched_at`) so **resolving an already-picked food needs no network**. Over time this
  builds the user's own vetted local subset — fast, offline-reusable, absorbing OFF's
  uneven quality (each entry vetted once on pick).
- **Unit→gram conversion:** OFF gives per-100 g, so quantities must resolve to grams.
  Mass units (g/kg) are exact. **Volume units (ml/l/cup/tbsp/tsp) are density-dependent**
  — OFF's `serving_size` is free text and rarely yields a usable density, so the converter
  uses a **small built-in density table for common liquids** (water, milk, oil…), falls
  back to water-equivalent (1 g/ml) only as a last resort, and **flags** volume ingredients
  with no density (flagged, not silently zeroed). Macros stay always-overridable.
- **LLM-estimate button:** asks the local LLM for per-portion macros (and an estimated
  total weight) when the DB lookup is incomplete or the user prefers it.
- **Manual:** the user can always type/override macros.
- The app stores **per-portion macros + portion count + total recipe weight (g)**.
  **Per-100 g is a derived value** — `perServing × servings ÷ totalWeightG × 100`, not
  independently persisted — locked in T1.2 so it can't drift. `totalWeightG` is
  auto-populated **only when *every* ingredient is picked *and* every unit resolves to
  grams**; if any ingredient is unpicked or has an unresolvable (flagged) unit, total
  weight is partial, so per-100 g shows **"—"** (same path as manual/LLM entries with no
  weight) rather than a silently wrong value. Macro fields: calories, protein, carbs, fat
  (extensible).

### D6 — Media, source link, deployment
- **Images:** optional upload(s) via Drogon multipart → disk volume, referenced by URL;
  also accept an external image URL. Validated type/size.
- **Source link:** optional URL field, shown as a link on the recipe page.
- **Deploy:** Docker Compose — **backend** (multi-stage C++ build → slim runtime),
  **frontend** (static build served by nginx), **postgres** (with a volume); Ollama is
  the user's own service referenced by `LLM_BASE_URL`. Runs on a home server / VPS.
  Needs **outbound network** for first-time Open Food Facts lookups (cached thereafter,
  subject to OFF's rate limits); optional Ollama for LLM features. **Graceful degradation:**
  if OFF is unreachable, search returns cached-only results with a clear message, and
  manual + LLM macro entry still work — a network outage never blocks recipe entry.
- **Auth assumption:** the app is **unauthenticated this phase** (single user). Because
  it is phone-reachable with full write + upload access, it MUST run behind a
  VPN / reverse-proxy basic-auth until in-app auth lands. Schema is auth-ready (D2).

## Standardised recipe format (`Recipe`) — locked field list
> **Locked at GATE 0** and validated against real German + English TikTok captions
> (three worked examples: a video-only caption with pre-computed macros, a grouped
> multi-section recipe, and an English one with steps + a storage note). The JSON Schema
> in `docs/` is the single source of truth, formalised at **T1.2**; the C++ struct and TS
> type mirror it. Because the format carries **`schemaVersion`**, any refinement found
> while building the paste parser (T3.x) is a **versioned migration, not a redesign** —
> the field list can evolve deliberately without breaking stored recipes.

```jsonc
{
  "schemaVersion": 1,               // format version, for future migrations
  "id": "uuid",
  "ownerId": "uuid?",               // auth-ready; unused this phase
  "title": "string",
  "description": "string?",
  "sourceUrl": "string?",           // optional source link
  "images": ["string"],             // optional, 0..n URLs
  "tags": ["string"],               // 0..n free-form tags; absorbs category/cuisine
                                    //   (e.g. "Meal Prep", "vegetarisch"). Not scraped
                                    //   from hashtag walls — user-curated.
  "servings": 4,                    // portion count (parsed from "4 Portionen"/"Serves 4")
  "prepTimeMin": 15,                // optional; minutes
  "cookTimeMin": 25,                // optional; minutes (total time is derived, not stored)
  "totalWeightG": 1200,             // optional; enables accurate per-100g
  "ingredients": [
    // quantity & unit are BOTH nullable → "to taste" / garnish (e.g. "Salz + Pfeffer",
    //   "Petersilie zum garnieren"); such rows never contribute to totalWeightG.
    // group: optional section label, kept flat on each row (e.g. "Für die Sauce",
    //   "Crispy Beef Strips"). null = ungrouped. Display groups by it; macro summation
    //   and the Reactive Form stay flat.
    // unit: one language-neutral vocabulary — g, kg, ml, l, Stück(piece), TL(tsp≈5ml),
    //   EL(tbsp≈15ml), cup, Prise… ; count/spoon/volume units resolve to grams only when
    //   a density/piece-weight is known, else they are flagged (per D5).
    // foodId links to a cached OFF food (from search-and-pick); null = manual/unmatched.
    // note: free text — captures parentheticals like "(diced)", "(uncooked weight)".
    {
      "group": "Für die Sauce?", "name": "chicken breast",
      "quantity": 300, "unit": "g", "foodId": "uuid?", "note": "?"
    }
  ],
  "steps": ["string"],              // flat, ordered; may be empty (video-only captions)
  "notes": "string?",              // free-form: storage/reheating tips, "next time…"
  "favorite": false,                // single-user star (chosen over a 1–5 rating)
  "macrosPerServing": { "calories": 0, "protein": 0, "carbs": 0, "fat": 0 },
  // macrosPer100g is DERIVED: perServing × servings ÷ totalWeightG × 100 (null if no weight)
  "macroSource": "ingredients | llm | manual",
                                    // caption-provided macros fold into "manual"
                                    //   (the user vets them in the editable preview)
  "createdAt": "iso", "updatedAt": "iso"
}
```
**Dropped / folded** (revisit later via `schemaVersion` if missed): `cuisine` and
`category` → fold into `tags`; `difficulty` → skipped (subjective, low payoff);
`yield` → `servings` (numeric) is authoritative for the macro math; a numeric
`rating` → replaced by the boolean `favorite`.

---

## Milestone 1 — Backend scaffold, schema, DB, browse API
- **T1.1** C++ toolchain + CMake + **`vcpkg.json` manifest** (`drogon[postgres]`,
  `valijson`, `libcurl`, a test framework; jsoncpp comes vendored with Drogon) + Drogon
  skeleton; `/health`
  endpoint; Postgres connection from env; idempotent **migration runner** with a
  `schema_migrations` table; `docker-compose` with a postgres service for dev. *Verify:*
  `cmake --build` succeeds (first build pulls Drogon's deps, slow); app boots; `GET
  /health` returns 200; logs show a successful Postgres connection; re-running the app
  does not re-apply migrations.
- **T1.2** `Recipe` JSON Schema in `docs/` (source of truth — the **locked field list**
  above); SQL migrations (`users` stub, `recipes` with nullable `owner_id`, **the recipe
  metadata columns — `tags`, `prep_time_min`, `cook_time_min`, `notes`, `favorite`** —
  **and macro columns — per-serving cal/protein/carbs/fat, `total_weight_g`,
  `macro_source`**, `ingredients` **with nullable `quantity`/`unit`/`group`/`food_id`/`note`
  and a language-neutral unit vocabulary**, `images`); C++ model structs + jsoncpp
  (de)serialization + a **`valijson` validation function**. *Verify:* migrations apply on
  a fresh DB; unit test round-trips a `Recipe` struct↔JSON — including a **grouped,
  to-taste-ingredient recipe** (null quantity/unit) and an **empty-`steps`** recipe — and
  `valijson` **rejects a schema-invalid document** and accepts a valid one.
- **T1.3** Recipe repository/service (create, read, list, update, delete) via Drogon
  `DbClient` (`execSqlSync`); per-100 g derived computation. *Verify:* integration tests
  run against a **dedicated test Postgres** (compose service; migrations applied before
  the suite) for CRUD, plus a unit test for per-100 g (incl. the "no weight" → null path).
- **T1.4** REST controllers `GET /api/recipes` (list) and `GET /api/recipes/:id`; seed
  2–3 example recipes. *Verify:* integration test hits both endpoints and gets the seeded
  recipes as schema-valid `Recipe` JSON.

## Milestone 2 — Frontend scaffold, browse/detail, structured form
- **T2.1** Angular CLI scaffold (pinned major version) + **dev `proxy.conf.json`**
  (`/api` → backend) + typed API service (`HttpClient`) + browse list page + detail page
  rendering title, image, source link, ingredients, steps, and both macro tables.
  *Verify:* `ng build` passes and `ng test` runs green using **ChromeHeadlessNoSandbox**
  (Chromium installed in the test env); against the running API (via the dev proxy) the
  list + a detail page render a seeded recipe with both macro columns (component test
  where practical).
- **T2.2** Add/Edit form built with Angular **Reactive Forms** (title, description,
  servings, weight, dynamic ingredient-row `FormArray`, steps, source URL, images, macro
  fields) → `POST`/`PUT /api/recipes` with **server-side validation** in the backend.
  *Verify:* a valid submission persists and appears in browse; invalid input is rejected
  with a message (backend validation unit test + a form-validation component test).
- **T2.3** Wire **manual macro entry + override** into the form (self-contained; no
  nutrition DB yet). *Verify:* a recipe saved with manually entered macros persists and
  renders both macro columns; per-100 g shows "—" when no weight is given. _(Auto
  "compute from ingredients" is deferred to M4 — see T4.2b — because the compute path
  does not exist until the nutrition DB lands.)_

## Milestone 3 — Paste import with selectable engine
- **T3.1** `RecipeParser` interface + `RuleBasedParser` in C++ + `POST /api/parse`.
  Concrete patterns to handle (from the worked German + English caption examples):
  **ingredient sections** (`🍗 Für das Hähnchen:` / `Crispy Beef Strips` → each row's
  `group`); **quantity/unit regex** over the language-neutral vocabulary (`600 g`,
  `140 ml`, `2 Knoblauchzehen`→`Stück`, `1 TL`, `2 tbsp`; a bare `tsp black pepper`
  defaults to quantity 1); **parentheticals → `note`** (`(diced)`, `(uncooked weight)`,
  `(tenderises the beef)`); **servings** from prose (`4 Portionen`, `Serves 4`); a
  **macro block** by label synonyms (`kcal`/`calories`; `Eiweiß`/`Protein`/`P`;
  `Kohlenhydrate`/`Carbs`/`C`; `Fett`/`Fat`/`F`) in any order → `macrosPerServing`,
  `macroSource: "manual"`; **hashtag walls + emoji stripped** (never auto-tagged);
  **to-taste rows** (`Salz + Pfeffer`, `Petersilie zum garnieren`) → null quantity/unit;
  and any **storage/reheating block → `notes`**. Steps may be absent (video-only). *Verify:*
  unit tests parse the three representative captions into the expected structured fields,
  including grouping, null-quantity rows, the extracted macro block, and empty steps.
- **T3.2** `LlmClient` + `LlmParser` (OpenAI-compatible call, schema validation, fallback
  on invalid). *Verify:* unit test with a **mocked** HTTP/LLM response asserts valid JSON
  is accepted and malformed JSON triggers the documented fallback (no live LLM in tests).
- **T3.3** Paste screen: textarea, engine toggle (rule-based / local LLM), parse →
  **editable preview form** (reuses M2 form) → save. *Verify:* pasting a sample with the
  rule-based engine produces a pre-filled, editable form that saves correctly.

## Milestone 4 — Macros from Open Food Facts (search & pick) + LLM estimate
- **T4.1** `NutritionSource` interface + **OFF client** (libcurl → OFF `/cgi/search.pl`,
  **descriptive `User-Agent`**, 429/backoff handling) + migration adding the **`foods`
  cache table (keyed by barcode `code`) + `ingredients.food_id`** + `GET /api/foods/search?q=`
  (**local `foods` first, live OFF only when insufficient**, then persist picked results) +
  the per-100 g nutrient mapping (`energy-kcal_100g`, else `energy_100g ÷ 4.184`;
  proteins/carbohydrates/fat `_100g`) + density-aware **unit→gram converter**. *Verify:*
  unit test with a **mocked** OFF response returns candidates; **resolving an
  already-picked food (by `food_id`/barcode) makes no API call** (cache hit); a **kJ-only
  mock** is converted (or rejected), never summed raw; unit-conversion tests (g/kg/ml/l +
  common units) incl. the "no density → flagged" path.
- **T4.2** Macro engine: for ingredients with a picked `foodId`, sum `quantity × per-100 g`
  → totals → per-serving + per-100 g. Ingredients with **no pick**, a product **missing
  `*_100g` nutriments**, or an **unresolvable unit** are surfaced to the UI (never zeroed),
  and any of them makes `totalWeightG` partial → per-100 g renders **"—"**. *Verify:* unit
  test on a recipe of fully-picked, gram-resolved foods gives expected macros **within a
  defined tolerance** (e.g. ±5%); a recipe with an unpicked/flagged ingredient reports it
  and shows per-100 g as "—".
- **T4.2b** `POST /api/macros/compute` + the frontend **search-and-pick UI** (per
  ingredient: search box → candidate list — **preferring products with complete nutriments
  / a nutrition grade** — → pick → macros fill; wired into the M2 form). *Verify:* in the
  form, searching an ingredient (mocked/live OFF) lists candidates, picking one fills its
  macros, and per-serving + per-100 g compute correctly when all ingredients are picked
  and gram-resolved (backend compute unit test + a form component test for the pick flow).
- **T4.3** `POST /api/macros/estimate` + "Estimate with local LLM" button →
  `LlmMacroEstimator` (schema-validated; also returns estimated total weight), fills macro
  fields for user review. *Verify:* unit test with a mocked LLM fills macro fields; the
  user can still override before save.

## Milestone 5 — Media, polish, search
- **T5.1** Image upload endpoint (Drogon multipart → disk volume) + external-URL option;
  type/size validation. *Verify:* uploading a valid image attaches it to a recipe and
  renders; oversized/wrong-type is rejected (API test).
- **T5.2** Browse **search/filter/sort** via `GET /api/recipes` query params + SQL
  (locked at GATE 0): **title text** search (case-insensitive substring on `title`,
  optionally `description`); **tag filter** (multi-select, **AND** semantics); a
  **favorites-only** toggle (`favorite = true`); **macro filters** `minProtein` +
  `maxCalories` on `macrosPerServing`; and **sort** by newest (`createdAt` desc, default),
  title A–Z, or highest protein. _(Full-text search over ingredients/steps is deferred to
  a later phase — needs Postgres FTS.)_ *Verify:* search tests return the expected subset
  from seeded data for a title query, a tag AND-filter, the favorites toggle, and a
  `minProtein`/`maxCalories` range, and confirm each sort order.

## Milestone 6 — Deployment & docs
- **T6.1** Multi-stage **Dockerfile** for the C++ backend (build → slim runtime) +
  Angular `ng build` static bundle served by nginx with a **`location /api/ { proxy_pass
  → backend }`** block (same-origin in prod) + `docker-compose.yml` (backend, frontend,
  postgres volume; `LLM_BASE_URL` → external Ollama) + `.env.example`. *Verify:* `docker
  compose up` builds and serves the app; browse works against a persisted Postgres
  volume; `/api/*` is reachable through nginx.
- **T6.2** `docs/` usage + config (env vars, pointing at Ollama, Postgres backup, C++
  build/vcpkg notes, Angular build notes). *Verify:* a **concrete copy-pasteable
  sequence** from the docs succeeds: `docker compose up` → `curl /health` returns 200 →
  `POST` a sample recipe → it appears in `GET /api/recipes`.

---

## Open questions for the human (GATE 0)
1. **Auth seam default:** OK to include the auth-ready schema (users stub + nullable
   `owner_id`) now with **no auth implemented**, per D2/D6?
2. **Scope:** all six milestones this phase, as laid out?

_Stack is settled: Angular SPA + C++/Drogon backend + PostgreSQL._

## Reviewer notes
_(newest round first)_

**Round 1** (TS/SvelteKit stack, superseded): APPROVE with 8 non-blocking; findings
folded into D3/D5 before the stack changed.

**Round 3** (nutrition source changed USDA → **Open Food Facts**): D5, D2, the `Recipe`
ingredient shape, D6, and all of M4 rewritten for OFF search-and-pick + live-API-then-
Postgres-cache. Re-review returned **CHANGES REQUIRED** — 3 blocking, 6 non-blocking, all
addressed:
- _B1 OFF API contract unspecified (UA/rate-limit/endpoint)_ → *Fixed* D5 + T4.1 pin
  endpoint, descriptive User-Agent, 429/backoff; D6 rate-limit note.
- _B2 per-100 g mapping unspecified, kJ/kcal ~4.18× bug_ → *Fixed* D5 + T4.1/T4.2 pin
  `energy-kcal_100g` (else `energy_100g ÷ 4.184`) + proteins/carbs/fat `_100g`; missing
  nutriments → not pickable/flagged; kJ-mock test added.
- _B3 "total weight always known" invariant broken by search-and-pick_ → *Fixed* D5 +
  T4.2/T4.2b: weight known only when every ingredient picked AND every unit resolves to
  grams; else per-100 g = "—".
- _NB fixes:_ prefer complete-nutriment products in pick UI (T4.2b); search local-first,
  cache-hit test resolves by `food_id`/barcode (T4.1); `foods` keyed by barcode `code` +
  `lang`/`fetched_at` (D2); built-in density table for liquids (D5); ODbL UI attribution
  (D5); OFF-unreachable graceful degradation (D6).
- _Also (human request):_ added `schemaVersion` to the `Recipe` format + a note that YAML
  export could layer on later (JSON stays canonical).
Reviewer signature reset to pending for a confirmation pass.

**Round 2** (Angular + C++/Drogon + Postgres, full re-review): **CHANGES REQUIRED** —
4 blocking, 10 non-blocking. All addressed:

_Blocking (all fixed):_
1. **T2.3 verified a macro compute path that doesn't exist until M4** — *Fixed*: T2.3
   scoped to manual entry/override; compute-from-ingredients moved to new **T4.2b** with
   an explicit `POST /api/macros/compute`.
2. **No JSON-Schema validator (jsoncpp can't validate)** — *Fixed*: added **valijson**
   (jsoncpp adapter) to D2, D3, T1.1 manifest, T1.2.
3. **CORS/dev-proxy never addressed** — *Fixed*: D1 adds dev `proxy.conf.json` + prod
   nginx `/api` reverse-proxy (same-origin, no CORS); T2.1 and T6.1 updated.
4. **vcpkg drogon lacks Postgres by default** — *Fixed*: D2 + T1.1 specify
   `drogon[postgres]` in a `vcpkg.json` manifest.

_Non-blocking (folded in):_
1. Macro columns missing from schema → added to T1.2. 2. per-100 g formula missing `×100`
→ fixed in D5 + schema comment. 3. weight known on ingredients path → D5 auto-populates
`totalWeightG`, "—" only for manual/LLM. 4. `ng test` needs headless Chrome + pin Angular
version → T2.1 + D1. 5. test-DB provisioning → T1.3 dedicated compose Postgres. 6. async
DB style → D2 fixes on `execSqlSync`. 7. migration versioning → D2/T1.1 `schema_migrations`
table. 8. LlmClient transport → D3 defaults to libcurl sync (+ OpenSSL note). 9. T6.2
subjective → made a concrete curl sequence. 10. prod nginx `/api` proxy → T6.1.
