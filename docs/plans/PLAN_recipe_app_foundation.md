---
plan: recipe_app_foundation
status: draft
approvals:
  reviewer: pending      # REOPENED again 2026-08-24 — human chose to drop vcpkg too, using system packages (apt) for libpq/OpenSSL/Catch2. Build-system change; re-review.
  human: pending      # was 2026-08-24; reset on the from-scratch pivot, awaiting re-signature
---

# Plan — mise-en-place recipe app (foundation)

## Goal
Build a **single-user, self-hosted** web app to store, browse, and add recipes.
Recipes are added via a **structured form** or by **pasting free text** (TikTok /
Instagram caption), which is normalised into one **standardised
recipe format**. (Pasted website text and URL/HTML import are a later phase — see D4.) Recipes render with optional image(s), an optional source link,
and **macros shown both per portion and per 100 g**. Macros are **auto-computed
from ingredients** via a nutrition database, with a **button to LLM-estimate**
instead. Paste-parsing and macro-estimation each offer a **user-selectable engine:
rule-based or a local LLM** (HTTP API, e.g. Ollama).

This plan covers the **product foundation** (all core features end-to-end) across
six milestones. It is the first workflow phase; later phases (advanced search,
tagging, meal planning, authentication, and **website-text / URL import** — see D4)
are out of scope here — but the schema is built **auth-ready** so auth is a later
addition, not a migration.

## Locked design decisions
> These are the decisions agreed with the human at GATE 0. A **C++ backend was
> explicitly chosen to learn C++**, and (revised 2026-08-24) the human chose to build it
> **from scratch — no web framework** — because building the fundamentals (an HTTP server,
> router, JSON, a DB layer over libpq, an HTTP client) is the point. This is a deliberate,
> knowing trade of substantial extra plumbing for that learning; the plan front-loads a
> **core-libraries milestone (M0)** so later milestones stand on our own libs.

### D1 — Architecture: split frontend + **from-scratch** C++ backend (monorepo)
> **Revised 2026-08-24 — the human chose to build the backend "from sockets up" rather than
> use a web framework, because building the fundamentals is the point of the learn-C++ goal.**
> This trades a large amount of up-front plumbing for that learning, and is accepted knowingly.
> Deploy target is Linux behind a VPN (D6), which bounds the security exposure of a hand-rolled
> HTTP server. Dev happens in **WSL2 (Ubuntu)** on the same OS family as deploy.

- **Backend:** **C++17/20, no web framework** — the app builds its own small libraries:
  - **HTTP server** — a TCP listener over OS sockets, an **HTTP/1.1 request parser**
    (request line, headers, body, `Content-Length`/chunked), keep-alive, a **thread pool**
    for concurrency. (**multipart/form-data** parsing is added into `net` when first needed,
    at M5/T5.1 — not part of the M0 server scope.)
  - **Router** — method + path (with `:id` params) → handler; a request/response abstraction.
  - **JSON** — own parser + serializer (replaces jsoncpp) and own **JSON-Schema validation**
    to the extent the `Recipe`/`Macros` schema needs (replaces valijson).
  - **DB layer** — a thin C++ wrapper over **libpq** (connect from a pool, parameterized
    `exec`, map results to structs). libpq speaks the Postgres wire protocol; we do NOT
    reimplement that — using libpq is using Postgres's own client, not a framework.
  - **HTTP client** — own client over sockets + **OpenSSL** for HTTPS (to OFF and the LLM;
    replaces libcurl), behind the `IHttpClient` seam (D3).
  - **Irreducible externals** (via **system packages / `apt`** — no package manager): **libpq**
    (PG protocol), **OpenSSL** (TLS), a **test framework** (Catch2). Everything else is ours.
    **No vcpkg** — the human chose system packages, knowingly trading vcpkg's reproducible,
    checked-in version pinning for a simpler, dependency-manager-free build; reproducibility
    instead rests on **pinned base-image / distro versions** in the Dockerfile (T6.1) and a
    documented `apt install` list (T6.2). CMake finds the libs via `find_package`/`pkg-config`.
- **Frontend:** **Angular SPA** (Angular CLI, `ng build` → static bundle), served as
  static files; it calls the backend over HTTP at `/api/*`. Chosen to learn a robust,
  structured framework; its **Reactive Forms** suit the dynamic ingredient-row form and
  editable paste-preview particularly well. **Pin an exact Angular version** (a specific
  `17.x.y`, not "v17+") in `package.json` up front — a floor like `^17` lets a fresh build
  pull a newer major with different builder/test defaults, the exact drift a pin prevents.
- **Same-origin strategy (no CORS):** in **dev**, `ng serve` (:4200) proxies `/api` →
  the backend via `frontend/proxy.conf.json` (`ng serve --proxy-config`); in **prod**,
  nginx serves the static bundle and **reverse-proxies `/api/*`** to the backend (see T6.1).
  Same-origin, so no browser CORS is needed. If any cross-origin path is later introduced,
  the own HTTP layer adds the CORS response headers.
- **Monorepo layout:**
  ```
  /backend    C++ from-scratch API (CMake; libpq/OpenSSL/Catch2 via system apt packages)
    /lib          our libraries: /net (sockets+http server) /router /json /db /httpclient
    /app          /controllers  /models  /services  /migrations
    /tests
  /frontend   Angular SPA
  /docs
  docker-compose.yml
  ```
- **API conventions (2nd-review Major 4 + 5 — pinned once, used everywhere):**
  - **Error envelope:** every non-2xx response is one JSON shape —
    `{ "error": { "code": "<machine_slug>", "message": "<human text>", "details": <any?> } }`
    — so the typed frontend service (T2.1) has one thing to code against.
  - **Warning envelope (success path — 3rd-review minor #1):** a `200` that carries a soft
    warning uses **one shape** — `{ "data": <payload>, "warning": { "code": "<slug>",
    "message": "<text>" } }` — for all three degradation flows (`/api/foods/search`
    cached-only; the parse LLM-invalid draft, T3.2; the LLM-macro fallback, T4.3). Absent
    `warning` ⇒ a clean result.
  - **Status codes:** `400` malformed request; `422` schema-invalid `Recipe`/macro body
    (our own validator, M0 `jsonschema`, failure — with failing paths in `details`); `404` unknown `:id`; `504` upstream
    timeout. **OFF-throttling behaviour for `/api/foods/search` (3rd-review minor #2):** if
    any **local candidate** exists → `200` + cached results + `warning`; otherwise propagate
    the upstream condition — OFF returned `429` → **`429`**; OFF unreachable → **`502`**;
    OFF timeout → **`504`**. **LLM unreachable/timeout** on `/api/parse` and
    `/api/macros/estimate` is **not** a 5xx: it degrades to the **`200` + warning-draft**
    path (the best-effort/empty editable draft of T3.2/T4.3), so a down LLM never blocks
    entry (consistent with D6) — the T3.2/T4.3 "invalid output" fallback and transport
    failure share this path. Mutations use the error envelope: **`PUT`/`DELETE` target
    `/api/recipes/:id`** (so the `404`-on-`:id` rule applies), and a successful **`DELETE`
    returns `204`**.
  - **List vs detail + pagination:** `GET /api/recipes` returns a **lightweight summary
    projection** (id, title, first image, `favorite`, per-serving macros + **`macrosEstimated`
    so the list can flag estimated macros too**, tags) — **not** full child-assembled
    objects — and is **paginated** via `?limit=&offset=`; **`limit` defaults to 50 and is
    clamped to a hard server-side maximum (e.g. 100)** so a large client `limit` cannot
    defeat the bound (3rd-review minor #5). Full canonical `Recipe` (all child tables
    assembled) is only `GET /api/recipes/:id`. This bounds the browse query as the store
    grows.
- **Trade-off accepted:** two build systems and two deploys; the `Recipe` type is
  **not auto-shared** across C++/TS — three representations (JSON Schema, C++ struct, TS
  type) are kept in sync **by hand** this phase, a known drift risk (Q9). Mitigated by D2
  (a single JSON-Schema source of truth in `docs/`, validated on both sides) and the T1.2
  round-trip test; a codegen step (e.g. `json-schema-to-typescript` for the TS type) is a
  clean later addition if drift bites.

### D2 — Storage: PostgreSQL (own libpq wrapper), with JSON as the interchange format
- **PostgreSQL** via our **own DB layer over libpq** (D1). Chosen over SQLite because the
  human intends to add **auth / multi-user later**, where Postgres is the sturdier base; the
  extra container is cheap under Docker Compose. The DB layer exposes a small **synchronous**
  API — connect (from our own connection pool), **parameterized `exec`** (`$1,$2…` binds,
  never string-concatenated SQL → no injection), and result→struct mapping — kept synchronous
  to stay approachable while learning.
- **Threading (own thread pool):** the HTTP server (D1) dispatches each request to a
  **worker thread** from our pool; a worker owns a libpq connection for the request and the
  blocking `exec`/HTTPS calls happen on that worker, never on the accept loop. For a
  **single user** a small pool (e.g. 4–8 workers) is ample; we do **not** claim non-blocking
  I/O — blocking a worker is fine at this scale. (This is where building it ourselves teaches
  the concurrency model a framework would have hidden.)
- **Dependencies via system packages (no vcpkg):** the only externals are **libpq**,
  **OpenSSL**, and **Catch2**, installed with **`apt`** (`libpq-dev`, `libssl-dev`,
  `catch2`/`libcatch2-dev`) in dev (WSL) and in the Docker build stage. CMake locates them
  with `find_package`/`pkg-config`. No framework, no dependency manager. (Reproducibility
  rests on pinned distro/base-image versions per D1; the build compiles **none** of these
  from source, so it is fast and light — Q8.)
- **JSON + schema validation are OURS (D1):** own parser/serializer, and own validation of
  the `Recipe`/`Macros` schema. The schema in `docs/` stays the single source of truth, but
  since we validate it ourselves we are **not bound to a library's supported draft** — we
  author it to a clear, self-consistent subset (object/array/string/number/enum/required/
  nullable + `definitions/Macros` via `$ref`) and our validator implements exactly that
  subset. This backs every "schema-validated" step (T2.x, T3.2, T4.3). (Supersedes the old
  valijson/Draft-7 constraint.)
- **Migrations:** plain **SQL files** in `/backend/migrations`, applied by a small
  **version-tracking runner** on startup that records applied versions in a
  `schema_migrations` table. Each migration runs **inside its own transaction** and its
  version is recorded **only on success** (a failure rolls back just that migration, not the
  ones already committed). **Concurrent-boot safety (Q7 + 2nd-review Major 3):** the runner
  must hold its lock and do all its work on **one dedicated libpq connection** (not one drawn
  from our request pool) — a `pg_advisory_lock` taken from a *pooled* connection could land
  the lock, the migrations, and the unlock on **different** connections and so fail to
  serialize. Concretely: acquire a **session-level `pg_advisory_lock` on that one dedicated
  connection, held across all the per-migration transactions**, then release it at the end.
  (A single wrapping transaction is *not* used — that would give all-or-nothing rollback
  across every migration, a different granularity; and a per-migration `pg_advisory_xact_lock`
  is avoided because it releases at each migration's commit, reopening the race between
  migrations.) The **"already applied?" check reads `schema_migrations` while the lock is
  held**, so two booting instances serialize. ("Idempotent" = safe to re-run the runner via
  version-tracking; the DDL itself is not required to be idempotent.) No ORM code-gen magic.
- **Relational storage shape (B1 — decided, not JSONB blobs):**
  - `recipes` — one row per recipe: scalar fields (`title`, `description`, `source_url`,
    `servings`, `prep_time_min`, `cook_time_min`, `total_weight_g`, `favorite`, `notes`,
    `schema_version`, `owner_id`, `created_at`, `updated_at`) + the **per-serving macro
    columns** (`cal`, `protein`, `carbs`, `fat`) + `macro_source` + **`macros_estimated`
    boolean** (persists the `macrosEstimated` flag so the "estimated" marker survives a
    save→reload — it is a **stored** field, not derived like per-100 g).
  - `recipe_ingredients` — **child table**, FK `recipe_id`, an explicit **`position`**
    integer for ordering, and columns `group_label` (nullable section label — the column is
    named `group_label`, **not** `group`, which is a Postgres reserved word; the JSON field
    stays `group`), `name`, `quantity` (nullable), `unit` (nullable), `food_id` (nullable
    UUID; **the FK → `foods.id` is added later in T4.1** when the `foods` table exists — the
    column is created here in M1 **without** the constraint), `note` (nullable). This is
    where the nullable-quantity/to-taste rows and grouping live.
  - `recipe_steps` — **child table**, FK `recipe_id`, `position`, `text`. (Ordered; may
    be empty for video-only captions.)
  - `recipe_images` — **child table**, FK `recipe_id`, `position`, `url`.
  - `recipe_tags` — **join table**, FK `recipe_id`, `tag`, plus a **`position`** so the
    `tags[]` array order round-trips (unique on `(recipe_id, tag)`). The T5.2 **tag
    AND-filter** is `... WHERE tag = ANY($tags) GROUP BY recipe_id HAVING count(*) = $n`.
  - **All child/join tables** carry `FK recipe_id … ON DELETE CASCADE`, so
    `DELETE /api/recipes/:id` (T2.2) removes the whole aggregate cleanly.
  - Rationale: ordered/queried collections are real rows (clean ordering, the tag-filter
    and macro-range SQL in T5.2, and future FTS all work), not opaque JSONB. The canonical
    `Recipe` **JSON** is assembled from these tables at the API boundary (D2's interchange
    layer); JSON is the wire format, these tables are the persistence.
- **Auth-ready schema (no auth implemented this phase):** a `users` table stub and a
  **nullable `owner_id`** FK on `recipes`. Nothing enforces it yet; it exists so auth
  is a later addition, not a schema migration of live data.
- A **`foods` table** (created in T4.1, M4) caches Open Food Facts entries the user has
  picked. It has a **surrogate UUID `id` primary key** (this is what `foodId: uuid?` in the
  `Recipe` format refers to) with the OFF **barcode `code`** as a **`UNIQUE` column** (not
  the PK), storing name, per-100 g macros (kcal/protein/carbs/fat), `lang`, and
  `fetched_at`, so repeat lookups need no network (D5). `recipe_ingredients.food_id`
  references **`foods.id`**; cache-hit resolution can look up by either `id` or the unique
  `code`.
- **Standardised recipe format (`Recipe`)** is defined once as a **JSON Schema in
  `docs/`** (the single source of truth), mirrored by a C++ struct (our own JSON
  (de)serialization) and a TS type. It is validated (by our own validator) at every API
  boundary, on paste-parser output, on LLM output, and used for export. So "everything becomes standardised
  JSON" holds at the API/interchange layer; Postgres is the persistence detail. The format
  carries a **`schemaVersion`** — under relational storage this **earns its keep mainly at
  the export/interchange layer** (a stored-format change is a SQL migration regardless), so
  it is kept but not oversold (Q6). (A YAML import/export convenience could be layered on
  later; JSON stays canonical.)

### D3 — Local-LLM integration (pluggable, behind an HTTP-client seam)
- **HTTP-client seam (B2 — testability):** outbound HTTP goes through a tiny
  **`IHttpClient` interface** (`get`/`post` → status + body), with **our own client
  implementation** (sockets + **OpenSSL** for HTTPS — D1) for production and a
  **fake/stub implementation** for tests. This is the seam the mocked-HTTP verifications in
  T3.2, T4.1, and T4.3 depend on — the real client is never hit in tests. Blocking behaviour
  runs on a worker thread (D2).
- `LlmClient` (built on the `IHttpClient` seam) calls a local LLM. Config via env:
  `LLM_BASE_URL` (server root, default `http://localhost:11434`), `LLM_MODEL` (**no baked
  default — documentation-only; if unset, the LLM engine is unavailable and the toggle is
  disabled, never a silent guess**), optional `LLM_API_KEY`.
- **Endpoint + structured output (M5):** default to **Ollama's native `/api/chat` with the
  `format` parameter set to the `Recipe`/macro JSON Schema** — reports indicate the native
  `format` (schema-enforced) is **more reliable** than the OpenAI-compatible
  `/v1/chat/completions` + `response_format: json_schema` path, which several models
  **ignore**. The OpenAI-compatible surface is kept as a **configurable fallback** for
  non-Ollama servers. Either way, **every response is validated with our own validator (M0
  `jsonschema`)** against the schema, with the documented graceful fallback (an empty/partial
  draft into the editable preview + a warning — see D4/T3.2) on invalid output. Not
  prompt-only coercion.
- LLM is **off unless the engine toggle selects it**, so the app is fully usable with
  no LLM running.
- **No translation:** the LLM only parses captions into schema JSON, **preserving the
  original language** (German stays German). OFF search already handles German terms, so
  German↔English translation is explicitly out of scope this phase.
- **Model is not hard-coded** — chosen at runtime by `LLM_MODEL`; switching models is an
  env change + restart (the model must be pulled in Ollama), no code change or rebuild.
  Documented **reference default: `qwen3.5-9b`** (fast; best-in-family German, and the
  fallback engine so speed matters). Higher-quality option: **`qwen3.8-27b`** for messy
  captions. **These tags are illustrative** (they mirror the user's local launcher labels,
  not confirmed Ollama registry ids) — **T6.2 must verify the exact `ollama pull` tag** so
  the `.env.example` default actually resolves. If strict JSON adherence ever becomes the bottleneck despite structured
  output, **Gemma 4 27B (Q4_K_M)** is a noted alternative. (Rationale: 2026 benchmarks
  put Qwen3 as the leader for non-English/German, while JSON reliability here comes from
  the server's structured-output mode + our own schema validation + fallback, not model
  obedience.)

### D4 — Paste parsing: two engines behind one interface
- **Scope this phase: social captions only** — TikTok / Instagram free-text captions
  (the three fixtures). **Out of scope:** pasted recipe-website text and URL/HTML
  fetching (schema.org/`Recipe` JSON-LD). Those are a clean later addition behind the
  same `RecipeParser` interface and are **not** built now.
- `RecipeParser` interface with two C++ implementations:
  - `RuleBasedParser` — **the best-effort primary engine**: the everyday workhorse that
    fully handles the pinned caption patterns (sections→`group`, quantity/unit regex,
    macro block, to-taste rows, parenthetical→note, hashtag/emoji stripping). Offline,
    free, deterministic; the app is fully usable with no LLM running.
    - **UTF-8 is a first-class concern (B3), handled by our own scanning — no regex engine.**
      Captions are full of multi-byte content — emoji (🍗💪🛒), umlauts/ß (`Eiweiß`,
      `Hähnchen`, `Kohlenhydrate`), `%`/`€`. Consistent with the from-scratch decision (D1)
      and the minimal externals (no `re2`), the parser does its **own UTF-8-aware scanning**:
      decode to Unicode codepoints, **strip emoji/symbols by codepoint ranges** (not byte
      hacks), tokenize on whitespace/punctuation, and match units/labels/quantities as tokens
      — **anchoring quantity matches at line/token start** so `140ml Kochsahne 7%` does not
      read the `7` as a quantity. German unit words (`TL`/`EL`/`Stück`/`Prise`)
      and macro labels (`Eiweiß`/`Kohlenhydrate`/`Fett`) are matched as whole tokens.
  - `LlmParser` — **the fallback** for messy captions the rules miss: sends pasted text
    + the `Recipe` JSON Schema to the local LLM, asks for schema-valid JSON, validates it.
    "Fallback" here means **user-selected** (the human toggles to the LLM engine when the
    rules do poorly) — there is **no automatic rule→LLM handoff**. When the LLM returns
    invalid/unparseable JSON, the documented behaviour is to surface a **best-effort or
    empty draft into the editable preview with a warning** (never a silent save, never an
    auto-retry with the other engine); the human then corrects it in the preview.
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
- **Known data-quality risk (M4 — relocated, not eliminated):** OFF is a **branded-product**
  database with sparse, uneven, user-contributed per-100 g data for **generic whole foods**
  (chicken breast, rump steak, rice, onion) — exactly what these fixtures are made of.
  Search-and-pick removes the *auto-match* error, but the user still picks among branded
  entries for a whole food, and quality varies. Mitigations (all already in the flow):
  the pick UI **prefers products with complete nutriments / a nutrition grade** (T4.2b),
  **manual override is always available** (a generic whole-food value the user trusts),
  and the **USDA seam** (clean generic values) is the intended future secondary source for
  exactly this gap. Documented as a real limitation, not a solved problem.
- **Search-and-pick per ingredient (human in the loop):** rather than auto-guessing a
  match (the previous plan's biggest accuracy risk), the user **searches OFF for each
  ingredient and picks the right food**; `quantity × per-100 g` → macros. This removes
  the risky fuzzy auto-match.
- **OFF API contract (M1 + 2nd-review Major 2 — endpoint/service pinned correctly):** these
  are **distinct** and must not be conflated. **Primary text search = the classic OFF API v2
  search, `GET https://world.openfoodfacts.org/api/v2/search`** (documented, stable).
  **Search-a-licious** is the *newer, separate* search service on its **own host**
  (`https://search.openfoodfacts.org`, its own `/search` endpoint) meant to eventually
  replace v2 — a valid future swap behind the `NutritionSource` interface, **not** the same
  thing as `/api/v2/search`. The legacy `/cgi/search.pl` is deprecated ("not recommended for
  new integrations") — last-resort fallback only. Product reads by barcode.
  OFF **mandates a descriptive `User-Agent`** (e.g. `mise-en-place/0.1 (contact)`) — a
  generic/empty UA is throttled/blocked, so our HTTP client always sets it — and OFF enforces
  **rate limits** returning 429.
  **Do not hard-code the old "~100/min product" figure — it is wrong/too high** (current
  reported product limit is materially lower, ~15/min; search ~10/min). **T4.1 must confirm
  the exact current limits against the live OFF docs** and size the backoff conservatively
  (a too-generous budget invites an IP ban). The client sets the UA and handles 429 with
  exponential backoff.
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
- **Piece/spoon resolution (M2 — so per-100 g isn't dark for real recipes):** the fixtures
  are dominated by piece and spoon units (`2 Knoblauchzehen`, `1 rote Zwiebel`, `1 TL`,
  `2 tbsp`) that the strict rule below would leave unresolved — making per-100 g "—" for
  virtually every real recipe. To fix that, the converter also carries a **small
  piece-weight table** (e.g. 1 clove ≈ 5 g, 1 onion ≈ 150 g, 1 egg ≈ 60 g, 1 bell pepper
  ≈ 150 g) and **fixed spoon volumes** (TL/tsp ≈ 5 ml, EL/tbsp ≈ 15 ml → grams via the
  density table). It may use OFF **`serving_size`** *only* when it clearly parses as a
  single-piece weight; it must **not** use OFF **`product_quantity`** — that is the
  **package** quantity (e.g. 500 g for a bag of rice), **not** a per-piece weight (2nd-review
  Major 2). These are **approximate defaults, clearly overridable**. Units with no table
  entry and no density remain **flagged** (not silently zeroed).
- **Honesty of approximate macros (2nd-review Major 1 — introduced by the M2 fix):** the
  piece/spoon table feeds **both** the weight denominator **and** each ingredient's grams, so
  when it contributes, the shown per-serving macros *and* per-100 g are **approximate**. To
  avoid presenting guessed numbers as exact, the recipe carries **`macrosEstimated: true`**
  whenever any piece/spoon-table (or serving_size-derived) weight fed the math, and the **UI
  marks those macros "estimated"**. This restores the honesty the strict "—" used to give,
  without going dark for real recipes. **The same flag is set on the LLM-estimate path (T4.3
  + 3rd-review Major):** LLM-estimated macros and total weight are approximate by nature, so
  `macrosEstimated: true` and the UI marks them estimated after save/reload, not only during
  entry — plus a `macroSource` badge (ingredients/llm/manual) so provenance survives reload
  regardless.
- **LLM-estimate button:** asks the local LLM for per-portion macros (and an estimated
  total weight) when the DB lookup is incomplete or the user prefers it.
- **Manual:** the user can always type/override macros.
- The app stores **per-portion macros + portion count + total recipe weight (g)**.
  **Per-100 g is a derived value** — `perServing × servings ÷ totalWeightG × 100`, not
  independently persisted — locked in T1.2 so it can't drift. `totalWeightG` is
  auto-populated **only when *every* ingredient is picked *and* every unit resolves to
  grams**; if any ingredient is unpicked or has an unresolvable (flagged) unit, total
  weight is partial, so per-100 g shows **"—"** (same path as manual/LLM entries with no
  weight) rather than a silently wrong value. With the M2 piece/spoon resolution above,
  common recipes now *do* reach a full weight; the "—" is the honest last-resort, not the
  normal case. Macro fields: calories, protein, carbs, fat (extensible).
- **Raw vs cooked weight (Q10 — documented caveat):** `totalWeightG` is the **sum of raw
  ingredient weights**, not the finished-dish weight (water evaporates in cooking), so the
  derived per-100 g is *per 100 g of raw input* and slightly understates the cooked dish.
  Acceptable for this phase; surfaced in the UI/docs so the number isn't mistaken for
  cooked-weight nutrition. The user can override `totalWeightG` with a measured cooked
  weight if they want cooked-basis per-100 g.

### D6 — Media, source link, deployment
- **Images (M6 — concrete limits, not just "validated"):** optional upload(s) via our **own
  multipart/form-data parser (added into `net` at T5.1)** → disk volume, referenced by URL. The backend
  **generates its own filename** (UUID + validated extension) and **never trusts the client
  filename** (no path traversal); it enforces **size ≤ 8 MB** and **content-type ∈ {jpeg,
  png, webp}** verified by magic bytes, not just the header. The uploads directory is
  **served by nginx** in prod (a `location /uploads/` block), not by the app.
- **External image URL (M6b — no SSRF):** the "external image URL" option is **store-only**
  — the URL is saved and rendered by the browser; the **backend does not fetch it**. This
  closes the SSRF hole (a server-side fetch could hit the home LAN / Ollama / Postgres).
- **Source link:** optional URL field, shown as a link on the recipe page (store-only).
- **Deploy:** Docker Compose — **backend** (multi-stage C++ build → slim runtime),
  **frontend** (static build served by nginx), **postgres** (with a volume); Ollama is
  the user's own service referenced by `LLM_BASE_URL`. Runs on a home server / VPS.
  Needs **outbound network** for first-time Open Food Facts lookups (cached thereafter,
  subject to OFF's rate limits); optional Ollama for LLM features. **Graceful degradation:**
  if OFF is unreachable, search returns cached-only results with a clear message, and
  manual + LLM macro entry still work — a network outage never blocks recipe entry.
- **Build cost (Q8 — now minimal):** with `apt` system packages, libpq/OpenSSL/Catch2 install
  as **prebuilt binaries** — they are **not compiled from source** — so the OOM risk that the
  old Drogon/vcpkg-from-source build carried is essentially gone. Only our own code compiles.
  (Base-image apt layers cache naturally; document the apt list + min build RAM in T6.2.)
- **Auth assumption (M6c):** the app is **unauthenticated this phase** (single user).
  Because it is phone-reachable with full write + upload access, it MUST run behind a
  **VPN, or a reverse-proxy with basic-auth *over TLS/HTTPS*** — basic-auth without TLS
  ships credentials in clear over a phone-reachable link, so **HTTPS is part of the
  caveat, not optional** — until in-app auth lands. Schema is auth-ready (D2).

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
                                    // PRIMARY provenance. Precedence: any manual edit to a
                                    //   macro or to totalWeightG flips this to "manual" AND
                                    //   clears macrosEstimated to false (the user asserts the
                                    //   edited values); a later recompute re-derives both.
                                    //   Shown as a badge in the UI (T2.1) so provenance
                                    //   survives reload. Finer per-field provenance is a
                                    //   future schemaVersion bump.
  "macrosEstimated": false,         // TRUE whenever the macros do NOT rest on exact
                                    //   per-100 g × gram-resolved-weight math — i.e. any of:
                                    //   an approximate piece/spoon-table (or serving_size)
                                    //   weight fed the math (M2), OR the macros came from the
                                    //   LLM estimator (T4.3). FALSE only for all-exact-grams
                                    //   ingredient computation and for manual entry the user
                                    //   asserts as exact. The UI marks estimated macros
                                    //   accordingly rather than showing them as exact.
  "createdAt": "iso", "updatedAt": "iso"
}
```
**Dropped / folded** (revisit later via `schemaVersion` if missed): `cuisine` and
`category` → fold into `tags`; `difficulty` → skipped (subjective, low payoff);
`yield` → `servings` (numeric) is authoritative for the macro math; a numeric
`rating` → replaced by the boolean `favorite`.

---

## Milestone 0 — Core backend libraries (from scratch)
> New milestone from the 2026-08-24 from-scratch pivot (D1). Builds the plumbing a framework
> would have given us, so later milestones have libraries to stand on. Everything here is
> unit-tested in isolation; no recipe logic yet.
- **T0.1** Toolchain + project skeleton: **CMake** finding the **system (apt) packages**
  `libpq-dev`, `libssl-dev`, `catch2`/`libcatch2-dev` via `find_package`/`pkg-config` (no
  vcpkg, no framework) + the `/backend/lib` + `/app` + `/tests` layout (D1) + a Catch2 test
  target. Dev happens in **WSL2 (Ubuntu)**. *Verify:*
  `cmake --build` succeeds; a trivial Catch2 test runs green.
- **T0.2** `net` — TCP socket listener + **HTTP/1.1 request parser** (request line, headers,
  body via `Content-Length` **and** chunked) + response writer + keep-alive + a **thread
  pool** (D2). Hardening: cap header size / count **and total body size (enforced *during*
  chunked decode too, where there is no upfront `Content-Length`)**; reject malformed with
  `400`; a **socket read/idle timeout** so a slow/stalled client (slowloris-style) cannot pin
  a pool worker indefinitely. *Verify:* unit tests parse well-formed and malformed requests
  (partial, oversized headers, bad `Content-Length`); an integration test makes a real
  localhost request and gets the expected response; oversized body/headers (incl. a chunked
  body exceeding the cap) are rejected, not OOM'd; a stalled connection is closed on timeout,
  freeing its worker.
- **T0.3** `router` + request/response abstraction: `method + path` (with `:id` params) →
  handler; unknown path → `404`, wrong method → `405`; the **error envelope + warning
  envelope** helpers (D1). *Verify:* routing tests incl. `:id` extraction, `404`/`405`, and
  an envelope-shaped error body.
- **T0.4** `json` — own parser + serializer (objects, arrays, strings **with unicode
  escapes**, numbers, `true`/`false`/`null`), round-tripping to a small DOM/value type.
  *Verify:* round-trip tests incl. nested structures, unicode escapes, big/edge numbers, and
  malformed input → a clear parse error (never a crash).
- **T0.5** `jsonschema` — own validator for the **authored subset** the app uses (types,
  `required`, `enum`, nullable, arrays, and `$ref` to `#/definitions/Macros`), reporting the
  failing path. *Verify:* accepts a valid `Recipe` and a valid `Macros` body; rejects each
  violation class (wrong type, missing required, bad enum, bad `$ref` target) with the path.
- **T0.6** `db` — libpq wrapper: a **connection pool**, **parameterized `exec`** (`$1,$2…`
  binds — SQL is never string-concatenated), result→row mapping, and a transaction helper;
  plus a **standalone (non-pooled) `connect()`** the migration runner (T1.1) uses to hold its
  advisory lock on one dedicated connection (D2). *Verify:* against a dev Postgres, a
  parameterized round-trip returns rows; a value containing SQL metacharacters passed as a
  **bind** is stored/returned literally (injection inert); the pool hands out and returns
  connections under concurrent use; a standalone connection can be opened outside the pool.
- **T0.7** App wiring: `main()` starts the `net` server on a port, mounts the `router`, builds
  the `db` pool from env, adds structured logging + config; `/health` endpoint. *Verify:*
  the app boots; **`GET /health` → 200 through our own server**; logs show a successful DB
  connection.
- **T0.8** `httpclient` — the **own HTTP/1.1 client** that fulfills the `IHttpClient` seam
  (D3), used by the LLM + OFF clients (T3.2/T4.1/T4.3). Scope: connect over sockets; write an
  HTTP/1.1 request; read the response incl. **chunked-transfer decode**; response body-size
  cap; **HTTPS via OpenSSL with certificate verification ON + SNI** (OFF is public-internet),
  and **plain HTTP** for the local Ollama path; a read/connect **timeout**. *Verify (real
  network, kept out of the default unit suite):* an HTTPS `GET` to a known good host succeeds
  and its cert is verified; a host with an **invalid/mismatched cert is rejected**, not
  silently accepted; a **plain-HTTP** GET to a local test server works; a chunked response
  decodes correctly; the fake `IHttpClient` remains what the T3.2/T4.1/T4.3 unit tests use.
- **🚦 M0 review** — `workflow:review` at the boundary passes _(the libraries are the
  foundation everything else stands on — worth a careful read)._

## Milestone 1 — Schema, DB, browse API (on the M0 libs)
- **T1.1** **Migration runner** (on the M0 `db` lib) with a `schema_migrations` table: plain
  SQL files, each migration in its **own transaction** (version recorded only on success),
  the runner holding a **session-level `pg_advisory_lock` on one dedicated libpq connection
  across all the per-migration transactions** (releasing at the end) so concurrent boots
  serialize (Q7 + 2nd-review Major 3, per D2); `docker-compose` with a postgres service for
  dev. *Verify:* migrations apply on boot; re-running does not re-apply; a deliberately
  failing migration leaves `schema_migrations` unchanged (that migration's transaction rolls
  back).
- **T1.2** `Recipe` JSON Schema in `docs/` (source of truth — the **locked field list**
  above), authored to the **self-consistent subset our own validator implements** (D2 — not
  bound to a library's draft). **Also author the macro body as a named sub-schema
  `definitions/Macros`** (the `{calories,protein,carbs,fat}` shape), referenced by
  `macrosPerServing` via `"$ref": "#/definitions/Macros"`; `POST /api/macros/compute` and
  `POST /api/macros/estimate` validate their request/response macro bodies against it, and the
  Ollama `format` for the macro estimator (D3) uses the same sub-schema — so the D1 `422` path
  and every "schema-validated **macro** body" step has one authored schema, not a second
  source of truth (3rd-review minor #3). SQL migrations for the **relational shape decided in D2/B1**:
  `users` stub; `recipes` (nullable `owner_id`, scalar fields, **per-serving macro columns
  cal/protein/carbs/fat, `total_weight_g`, `macro_source`, `macros_estimated`**); **child tables
  `recipe_ingredients`** (FK, `position`, nullable
  `quantity`/`unit`/`group_label`/`food_id`/`note` — **`food_id` is a plain nullable UUID
  column here, no FK yet** (the `foods` table lands in T4.1), and the section column is
  `group_label` not the reserved word `group`; language-neutral unit vocabulary),
  **`recipe_steps`** (FK, `position`, `text`),
  **`recipe_images`** (FK, `position`, `url`); **join table `recipe_tags`** (FK, `tag`,
  `position`) — all child/join FKs **`ON DELETE CASCADE`**; C++ model structs + **our own JSON
  (de)serialization (M0 `json`)** that **assemble/emit the canonical `Recipe` JSON from these
  tables** (note the JSON↔column name maps, e.g. `macrosPerServing.calories` ↔ column `cal`) +
  a validation function using the **M0 `jsonschema` validator**. *Verify:* migrations apply on
  a fresh DB; unit test round-trips a `Recipe` struct↔JSON (via the child tables) — including a
  **grouped, to-taste-ingredient recipe** (null quantity/unit), ordered `steps`, an
  **empty-`steps`** recipe, **`tags[]` + `images[]` with order preserved**, **and
  `macrosEstimated: true` surviving the round-trip** — and the validator **rejects a
  schema-invalid document** and accepts a valid one.
- **T1.3** Recipe repository/service (create, read, list, update, delete) via the **M0 `db`
  lib** (parameterized `exec`) — writing/reading across the parent + child tables in a
  transaction; per-100 g derived computation. **Update (PUT) strategy (2nd-review blocker):
  a full-replace within one transaction** — the client PUTs the *complete* recipe (the M2
  form already holds every ingredient's `foodId`, so it round-trips them), and the service
  replaces the child rows from that payload, reassigning `position` from array order; **a
  picked `food_id` survives an edit** because the payload carries it (a bare
  delete-and-reinsert that dropped `food_id` is explicitly rejected). **On PUT the backend
  recomputes `macrosEstimated` from the payload's ingredient/weight resolution** rather than
  trusting the client-sent flag (3rd-review minor #7). **Image-file GC (3rd-review minor #6):**
  because `ON DELETE CASCADE` removes `recipe_images` rows, the service **reads the owned
  image filenames first**, then deletes the recipe, then unlinks the backend-owned files
  (best-effort, logged on failure — an orphaned file is a warning, never a failed request);
  on PUT it diffs old vs. new image URLs and unlinks the dropped **owned** files after commit
  (external-URL images are store-only — never touched). *Verify:* integration tests run
  against a **dedicated test Postgres** (compose service; migrations applied before the
  suite; **each test truncates the recipe tables** for determinism — Q1) for CRUD incl. a
  multi-group/multi-step recipe, **a PUT edit that preserves `food_id` picks and reorders
  ingredients**, plus a unit test for per-100 g (incl. the "no weight" → null path).
- **T1.4** REST controllers `GET /api/recipes` (**paginated summary list** — `?limit=&offset=`,
  summary projection per D1, not full child-assembled objects) and `GET /api/recipes/:id`
  (full canonical `Recipe`); seed 2–3 example recipes (**with `food_id` left null** so the
  T4.1 FK-add finds no orphans). *Verify:* integration test hits both endpoints — the list
  returns summaries honoring `limit`/`offset`, `/:id` returns a schema-valid full `Recipe`.

## Milestone 2 — Frontend scaffold, browse/detail, structured form
- **T2.1** Angular CLI scaffold (**exact pinned version `17.x.y`**, not `^17` — Q3) + **dev `proxy.conf.json`**
  (`/api` → backend) + typed API service (`HttpClient`) **coding against the D1 error
  envelope** (one `{error:{code,message,details}}` shape) + browse list page (**consuming the
  paginated summary list**) + detail page rendering title, image, source link, ingredients,
  steps, and both macro tables (**showing the "estimated" marker when `macrosEstimated` — on
  both the detail page and the browse list, which the summary projection already carries the
  flag for — plus a small `macroSource` badge (ingredients/llm/manual) on the detail page so
  provenance survives reload**).
  *Verify:* `ng build` passes and `ng test` runs green using **ChromeHeadlessNoSandbox**
  (Chromium installed in the test env); against the running API (via the dev proxy) the
  list + a detail page render a seeded recipe with both macro columns (component test
  where practical).
- **T2.2** Add/Edit form built with Angular **Reactive Forms** (title, description,
  servings, weight, dynamic ingredient-row `FormArray`, steps, source URL, images, macro
  fields) → `POST`/`PUT /api/recipes` with **server-side validation** in the backend
  (**schema-invalid → `422` with failing paths in the error envelope; unknown id → `404`**,
  per D1); **`PUT` sends the complete recipe** so `food_id` picks survive the full-replace
  (T1.3). Plus a **`DELETE /api/recipes/:id`** controller (exposing the T1.3 repository
  `delete`; child rows cascade) wired to a delete action in the UI with a confirm. *Verify:*
  a valid submission persists and appears in browse; **a schema-invalid POST returns `422`
  with the error envelope**; **a PUT edit preserves picked `food_id`s**; **deleting a recipe
  removes it (and its child rows) from `GET /api/recipes`** (backend validation + delete API
  test + a form-validation component test).
- **T2.3** Wire **manual macro entry + override** into the form (self-contained; no
  nutrition DB yet). *Verify:* a recipe saved with manually entered macros persists and
  renders both macro columns; per-100 g shows "—" when no weight is given. _(Auto
  "compute from ingredients" is deferred to M4 — see T4.2b — because the compute path
  does not exist until the nutrition DB lands.)_

## Milestone 3 — Paste import with selectable engine
- **T3.1** `RecipeParser` interface + `RuleBasedParser` in C++ + `POST /api/parse`.
  Scope is **social captions only** (TikTok/Instagram); the rule-based engine is the
  **best-effort primary** parser (LLM is the fallback in T3.2).
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
  and any **storage/reheating block → `notes`**. Steps may be absent (video-only).
  **UTF-8 handling per D4/B3: NFC-normalize the input first** (pasted captions may arrive
  NFD-decomposed, e.g. `ä` = `a`+U+0308, which would break whole-token matches for
  `Eiweiß`/`Hähnchen` and codepoint-range emoji stripping — 2nd-review minor), then **our own
  UTF-8 scanning (no `std::regex`, no regex engine), codepoint-range emoji/symbol stripping,
  and line-start-anchored quantities** so `140ml … 7%` doesn't misread `7`. *Verify:* unit tests parse the three
  representative captions into the expected structured fields, including grouping,
  null-quantity rows, the extracted macro block, empty steps, **correct emoji/umlaut
  handling (`Eiweiß`, `Hähnchen`), an NFD-decomposed input variant, and the
  `7%`-not-a-quantity case**.
- **T3.2** `LlmClient` (on the **`IHttpClient` seam** from D3/B2 — real impl is the T0.8
  `httpclient`, plain-HTTP for local Ollama) + `LlmParser` (Ollama
  native `/api/chat` `format`=schema by default, OpenAI-compatible fallback; schema
  validation; fallback on invalid). Documented fallback = on invalid/unparseable LLM JSON,
  return a **best-effort or empty draft into the editable preview with a warning** (no
  auto-retry, no silent save). *Verify:* unit test with the **fake `IHttpClient`** (no live
  LLM, no real network) asserts valid JSON is accepted and malformed JSON triggers that
  documented fallback — an empty/partial draft plus a warning flag, not an exception or a
  saved record.
- **T3.3** Paste screen: textarea, engine toggle (rule-based / local LLM), parse →
  **editable preview form** (reuses M2 form) → save. *Verify:* pasting a sample with the
  rule-based engine produces a pre-filled, editable form that saves correctly.

## Milestone 4 — Macros from Open Food Facts (search & pick) + LLM estimate
- **T4.1** `NutritionSource` interface + **OFF client** (on the **`IHttpClient` seam**
  from D3/B2 — real impl is the T0.8 `httpclient`, **HTTPS with cert verification** to the
  public OFF host → **OFF API v2 search `/api/v2/search`** primary (**not** Search-a-licious,
  which is a separate service/host — see D5 Major 2), legacy `/cgi/search.pl` last-resort
  fallback; **descriptive `User-Agent`**; 429/backoff) + migration creating the **`foods`
  cache table (surrogate UUID `id` PK, `code` UNIQUE)** and **adding the FK constraint
  `recipe_ingredients.food_id → foods.id`** onto the column that already exists from T1.2
  (no new column) + `GET /api/foods/search?q=` (**local `foods` first, live OFF only when
  insufficient**, then
  persist picked results) + the per-100 g nutrient mapping (`energy-kcal_100g`, else
  `energy_100g ÷ 4.184`; proteins/carbohydrates/fat `_100g`) + a **unit→gram converter with
  the density table AND the M2 piece-weight / spoon-volume table**. **First confirm the
  current OFF rate limits against the live docs (M1)** — do not hard-code the old
  "~100/min" figure — and size backoff conservatively. *Verify:* unit test with the **fake
  `IHttpClient`** returns candidates; **resolving an already-picked food (by
  `food_id`/barcode) makes no API call** (cache hit); a **kJ-only mock** is converted (or
  rejected), never summed raw; unit-conversion tests (g/kg/ml/l + **piece units like
  `Knoblauchzehen`/`Zwiebel` and spoons `TL`/`EL`**) incl. the "no table entry, no density
  → flagged" path.
- **T4.2** Macro engine: for ingredients with a picked `foodId`, sum `quantity × per-100 g`
  → totals → per-serving + per-100 g. Ingredients with **no pick**, a product **missing
  `*_100g` nutriments**, or an **unresolvable unit** are surfaced to the UI (never zeroed),
  and any of them makes `totalWeightG` partial → per-100 g renders **"—"**. **Sets
  `macrosEstimated: true` whenever a piece/spoon-table (or serving_size) weight fed the math**
  (2nd-review Major 1), so the UI can mark those macros estimated rather than exact. *Verify:*
  (a) an **all-exact-grams** recipe computes per-serving + per-100 g against **hand-computed
  expected values** (not the converter's own constants — avoids the tautology the earlier
  ±5% test had) with `macrosEstimated:false`; (b) a **piece/spoon** recipe computes a full
  `totalWeightG` **and sets `macrosEstimated:true`**; (c) a recipe with an unpicked/flagged
  ingredient reports it and shows per-100 g as "—".
- **T4.2b** `POST /api/macros/compute` + the frontend **search-and-pick UI** (per
  ingredient: search box → candidate list — **preferring products with complete nutriments
  / a nutrition grade** — → pick → macros fill; wired into the M2 form). *Verify:* in the
  form, searching an ingredient (mocked/live OFF) lists candidates, picking one fills its
  macros, and per-serving + per-100 g compute correctly when all ingredients are picked
  and gram-resolved (backend compute unit test + a form component test for the pick flow).
- **T4.3** `POST /api/macros/estimate` + "Estimate with local LLM" button →
  `LlmMacroEstimator` (reuses `LlmClient` on the **`IHttpClient` seam**; schema-validated;
  also returns estimated total weight), fills macro fields for user review **and sets
  `macrosEstimated: true`** (LLM output is an estimate — 3rd-review Major). *Verify:* unit
  test with the **fake `IHttpClient`** (no live LLM) fills macro fields **and asserts the
  filled recipe carries `macrosEstimated: true`**; the user can still override before save.

## Milestone 5 — Media, polish, search
- **T5.1** Image upload endpoint — **this task adds a `multipart/form-data` parser into the
  `net` lib** (deferred from M0, first needed here) → disk volume + external-URL option
  (**store-only, no server-side fetch — M6b/SSRF**). Concrete validation per D6/M6:
  **server-generated filename** (UUID + extension, client filename ignored — no path
  traversal), **size ≤ 8 MB**, **content-type ∈ {jpeg,png,webp} verified by magic bytes**.
  **Files are lifecycle-managed (3rd-review minor #6):** the uploads service **owns** the
  UUID-named files it wrote and unlinks any owned file no longer referenced by a
  `recipe_images` row — on recipe delete and on the PUT image diff (per the T1.3 ordering);
  external-URL images own no file. *Verify:* uploading a valid image attaches it and renders;
  **oversized, wrong-type, and a crafted path-traversal filename are all rejected**; an
  external image URL is stored and **never fetched by the backend** (API test asserts no
  outbound request); **deleting a recipe (and replacing an image via PUT) removes the
  corresponding owned file from the uploads volume, and an external-URL image is never
  touched**.
- **T5.2** Browse **search/filter/sort** via `GET /api/recipes` query params + SQL
  (locked at GATE 0), layered onto the **paginated summary list** (`limit`/`offset`, D1):
  **title text** search (case-insensitive substring on `title`, optionally `description` —
  using a **German-aware case-insensitive match**, e.g. `ILIKE` under a
  case/umlaut-appropriate collation or `citext`, so `ß`/umlaut folding behaves; 2nd-review
  minor); **tag filter** (multi-select, **AND** semantics); a **favorites-only** toggle
  (`favorite = true`); **macro filters** `minProtein` + `maxCalories` on the per-serving
  macro columns; and **sort** by newest (`createdAt` desc, default), title A–Z, or highest
  protein. _(Full-text search over ingredients/steps is deferred to a later phase — needs
  Postgres FTS.)_ *Verify:* search tests return the expected subset from seeded data for a
  title query (**including a German umlaut/ß case-insensitivity case**), a tag AND-filter,
  the favorites toggle, and a `minProtein`/`maxCalories` range; confirm each sort order **and
  that `limit`/`offset` paginate — and that a `limit` above the max is clamped, not honored
  verbatim** (3rd-review minor #5).

## Milestone 6 — Deployment & docs
- **T6.1** Multi-stage **Dockerfile** for the C++ backend (build → slim runtime), on a
  **pinned base-image tag** (reproducibility now rests on this, not a manifest — D1); the
  build stage does **`apt install` of `libpq-dev`/`libssl-dev`/`catch2`** (prebuilt, fast).
  The **slim runtime image MUST include the runtime libs (`libpq5`, `libssl`) and
  `ca-certificates`** — our own OpenSSL HTTPS client (T0.8) verifies the OFF cert against the
  system trust store, so without CA certs (or the runtime libs) the OFF path fails at deploy
  even though it passed in dev. + Angular
  `ng build` static bundle served by nginx with a **`location /api/ { proxy_pass → backend }`**
  block **and a `location /uploads/`** block (same-origin in prod; nginx serves uploads — M6)
  + `docker-compose.yml` (backend, frontend, postgres volume; `LLM_BASE_URL` → external
  Ollama) + `.env.example`. *Verify:* `docker compose up` builds and serves the app; browse
  works against a persisted Postgres volume; `/api/*` and `/uploads/*` are reachable through
  nginx; **an OFF search from inside the running backend container succeeds (CA trust works)**.
- **T6.2** `docs/` usage + config (env vars, pointing at Ollama, Postgres backup, C++
  build notes **incl. the exact `apt install` package list** and the pinned base-image tag,
  Angular build notes, **and the minimum build RAM** — Q8). **Verify the exact Ollama pull tag** for the documented `LLM_MODEL`
  (`ollama list`) and put a resolvable tag in `.env.example`. *Verify:* a **concrete
  copy-pasteable sequence** from the docs succeeds: `docker compose up` → `curl /health`
  returns 200 → `POST` a sample recipe → it appears in `GET /api/recipes`.

---

## Open questions for the human (GATE 0)
_Resolved this session (all locked): `Recipe` field list; browse/search scope (T5.2);
paste-parser depth + source scope (D4 — social captions only); LLM model policy +
no-translation (D3); milestone order (deploy stays M6); **per-100 g gap (M2) → add a
piece-weight/spoon-volume table so it resolves for real recipes**; **scope (Q2) → keep all
six milestones**. Remaining sign-off questions:_
1. **Auth seam default:** OK to include the auth-ready schema (users stub + nullable
   `owner_id`) now with **no auth implemented**, per D2/D6?
2. **Scope:** all six milestones this phase, as laid out?

_Stack: Angular SPA + **from-scratch C++ backend (no framework — own HTTP server/router/JSON/
schema/DB-over-libpq/HTTP-client; externals = libpq, OpenSSL, Catch2)** + PostgreSQL._

## Reviewer notes
_(newest round first)_

**Drop vcpkg (2026-08-24)** — after the from-scratch pivot was reviewer-approved, the human
chose to **drop vcpkg too and use system packages (`apt`)** for the three externals
(libpq/OpenSSL/Catch2). A build-system change (not architecture): reviewer stamp reset again.
Edits: D1 (externals via apt, no package manager; reproducibility now rests on pinned
base-image/distro versions + a documented apt list), D2 (dependencies-via-apt note, CMake
`find_package`/`pkg-config`), Q8 (build cost now minimal — prebuilt binaries, nothing compiled
from source), T0.1 (CMake finds apt packages), T6.1 (pinned base image + `apt install` in the
build stage + runtime libs `libpq5`/`libssl` and `ca-certificates`), T6.2 (document the apt
list + base tag). **Knowing trade recorded:** loses vcpkg's checked-in version pinning; the
human accepted this. Awaiting a reviewer confirm pass, then the human signature.

**From-scratch pivot (2026-08-24)** — after GATE 0 was passed, the human chose to **drop the
Drogon framework and build the backend from sockets up** (own HTTP server/router/JSON/schema/
DB-over-libpq/HTTP-client; externals shrink to libpq + OpenSSL + Catch2). This is a material
change to the approved D1–D3 + milestones, so `approvals` were **reset to pending** and the
plan re-opened. Revisions: D1 rewritten (own libraries, new `/lib` layout); D2 (own DB layer,
own thread pool, own JSON + validator, vcpkg trimmed); D3 (own HTTP client on the `IHttpClient`
seam, own validator); parser uses **own UTF-8 scanning, no RE2**; multipart is our own (M0
`net`); build-cost note shrinks. **New Milestone 0 — Core backend libraries** (T0.1–T0.7:
toolchain, `net`/HTTP server, `router`, `json`, `jsonschema`, `db`, `/health` wiring) inserted
before M1; **M1 becomes "Schema, DB, browse API" on the M0 libs** (T1.1 = migration runner,
T1.2–T1.4 unchanged in number). M2–M6 (Angular, paste, OFF, media, deploy) keep their numbers
and all task references. **T0.8 `httpclient` added** after the pivot review (below).

**Pivot review R1** (workflow reviewer over the from-scratch revision, aimed at
implementability + hand-rolling security): **CHANGES REQUIRED** — 1 blocking + 4 non-blocking,
all addressed:
- _BLOCKING: the own HTTP **client** (HTTPS/OpenSSL) was declared in D1/D3 but built by no
  task_ → *Fixed*: added **T0.8 `httpclient`** (OpenSSL TLS with cert verification + SNI,
  chunked decode, timeouts, body cap) with a real-network verify; referenced from T3.2/T4.1;
  pinned **`ca-certificates` in the slim runtime image** (T6.1).
- _NB multipart built by no task_ → *Fixed*: D1/D6/T5.1 state it's added into `net` at **T5.1**.
- _NB stale `valijson` in the D1 `422` line_ → *Fixed*: "our own validator (M0 `jsonschema`)".
- _NB T0.6 lacked the non-pooled connection T1.1's migration lock needs_ → *Fixed*: standalone
  `connect()` added to T0.6.
- _NB no server idle timeout (slowloris); body cap not enforced on chunked decode_ → *Fixed*:
  T0.2 adds a read/idle timeout + chunked-body cap.
Reviewer confirmed the other from-scratch claims sound and that previously-approved content
survived intact.

**Pivot review R2** (diff-only confirm on the R1 fixes): **APPROVE** — T0.8 `httpclient` is a
real M0 task referenced consistently (T3.2/T4.1/T6.1 `ca-certificates`); multipart→T5.1; own
validator in the `422` line; T0.6 standalone connect; T0.2 timeout + chunked cap. M0→M1 chain
sound, no new inconsistency. Reviewer signature re-stamped `2026-08-24`; **human GATE 0
signature is the only remaining step.**

**Round 12** (diff-only confirm on the 3rd-review fold-in): **APPROVE** — `macrosEstimated`
verified consistent across format/D5/T4.2/T4.3/T2.1/D1; `definitions/Macros`, file-GC
ordering, warning envelope, clamp, and recompute rule all sound. Two non-blocking notes
(LLM-unreachable status; `$defs`→`definitions` Draft-7 idiom) **also folded in**. Reviewer
signature stamped `2026-08-24`. **Human GATE 0 signature is the only remaining step** — the
3rd external review named this the exit point.

**3rd external review** (`docs/reviews/PLAN_REVIEW_2026-08-24_external-3.md`, two passes,
over the Round-10 plan): **APPROVE with findings — 0 blocking** (first time in the chain), 1
major, 8 minor. It confirms the loop has converged (R1 3 blocking+6 major → R2 1 blocking+5
major → R3 0 blocking+1 major) and names this the exit point. Reviewer stamp rolled back to
fold in; all findings addressed:
- _Major: `macrosEstimated` didn't cover the LLM-estimate path (T4.3) — LLM macros reload as
  `false`, indistinguishable from exact_ → *Fixed* (recommended "do both", **pending human
  override at sign-off**): broadened `macrosEstimated` to **any non-exact path incl. LLM**
  (format + D5 + T4.3 sets it + verify), and added a **`macroSource` badge** on the detail
  page + the estimated marker on the **list** too (T2.1).
- _Minor #1 warning shape unpinned_ → D1 adds a **success-path warning envelope**
  `{data,warning{code,message}}` for all three degradation flows.
- _Minor #2 429-vs-502 ambiguous_ → D1 disambiguates (cached→200+warning; else 429/502/504
  by upstream condition).
- _Minor #3 macro-body schema never authored_ → T1.2 authors **`definitions/Macros`** in the same
  Draft-7 doc; compute/estimate + Ollama `format` validate against it.
- _Minor #4 list didn't render the flag its projection carries_ → T2.1 renders the estimated
  marker on the list too.
- _Minor #5 pagination default not a max_ → D1 **hard-clamps `limit`** (default 50, max e.g.
  100) + T5.2 verify.
- _Minor #6 uploaded files never GC'd_ → T5.1/T1.3 **lifecycle-manage owned files** (unlink
  on delete + PUT image-diff, read filenames before cascade; external-URL untouched) + verify.
- _Minor #7 `macrosEstimated`×override undefined_ → format: manual edit **clears** the flag;
  PUT **recomputes** it from the payload (T1.3).
- _Minor #8 stale "Fixed" note claimed `pg_advisory_xact_lock`_ → corrected the 2nd-review
  Major-3 note to the actual session-lock mechanism.
Awaiting a confirm pass, then the human signature (the exit the review recommends).

**Round 10** (diff-only confirm on the Round 9 fixes): **APPROVE** — `macros_estimated`
now consistent across format → storage → compute → detail + list display; the migration
lock pins one sound mechanism; `429`-vs-cache precedence and `PUT`/`DELETE` `:id`/`204`
pinned. No blocking, no non-blocking, no new inconsistency. Reviewer signature stamped
`2026-08-24`. **Human GATE 0 signature still pending** (a possible 3rd external review is
the user's call).

**Round 9** (workflow reviewer over the 2nd-external-review fold-in, aimed at
implementability depth + fact-checking): confirmed all the folded fixes present and
consistent, but found **1 blocking + 4 non-blocking**. All addressed:
- _BLOCKING: `macrosEstimated` had no storage column — the Major-1 honesty flag couldn't
  survive save→reload (present in the format, T4.2 compute, and T2.1 display, but not in the
  `recipes` columns)_ → *Fixed*: added a stored **`macros_estimated` boolean** column
  (D2/B1 + T1.2), included it in the assembly + the T1.2 round-trip test, and added
  `macrosEstimated` to the D1 summary projection so the browse list flags it too.
- _NB migration-lock phrasing described two non-equivalent mechanisms_ → *Fixed*: D2 + T1.1
  now pin one — a **session `pg_advisory_lock` on a dedicated connection held across the
  per-migration transactions**, with the applied-version check read under the lock (dropped
  the "wrapping transaction / `pg_advisory_xact_lock`" conflation).
- _NB `429` vs partial-cache precedence unpinned_ → *Fixed*: D1 states cached `200`+warning
  takes precedence over surfacing `429` when local candidates exist.
- _NB REST surface details_ → *Fixed*: D1 pins `PUT`/`DELETE` target `/api/recipes/:id` and
  `DELETE` → `204`.
- _NB `macroSource` single enum lossy for mixed provenance_ → accepted as a documented trade
  (precedence rule + future `schemaVersion`); no change.
Awaiting a confirm pass.

**2nd external review** (`docs/reviews/PLAN_REVIEW_2026-08-24_external-2.md`, two independent
passes, over the Round-8 plan): **CHANGES REQUIRED** — it confirmed the Round 7 fixes are
genuinely in the body, then found **1 blocking + 5 major (two introduced by the M2 fix) +
6 minor**. Reviewer stamp **rolled back again**. All addressed:
- _BLOCKING: PUT child-table update semantics undefined (delete-and-reinsert would drop
  `food_id` picks)_ → *Fixed*: T1.3 pins a **full-replace in one transaction** where the
  client PUTs the complete recipe (form holds every `foodId`), so **picks survive** and
  `position` is reassigned from array order; T2.2 verify checks it.
- _Major 1: piece/spoon weights make macros approximate but shown as exact (introduced by
  M2)_ → *Fixed*: added **`macrosEstimated`** to the `Recipe` format; D5/T4.2 set it whenever
  a piece/spoon/serving_size weight fed the math; UI marks those macros "estimated".
- _Major 2: OFF endpoint/service conflated ("Search-a-licious `/api/v2/search`") + misuse of
  `product_quantity` as per-piece weight_ → *Fixed*: D5/T4.1 pin **OFF API v2 `/api/v2/search`**
  as primary (Search-a-licious is a separate service/host, a future swap), and **forbid
  `product_quantity`** (package qty, not per-piece); only cautious `serving_size`.
- _Major 3: `pg_advisory_lock` unsound over Drogon's pooled connections_ → *Fixed*: D2/T1.1
  use a **session-level `pg_advisory_lock` on a dedicated connection held across the
  per-migration transactions** (no wrapping transaction; the `pg_advisory_xact_lock` phrasing
  was superseded by Round 9 — see there).
- _Major 4: API error contract (envelope + status codes) unspecified_ → *Fixed*: D1 pins one
  `{error:{code,message,details}}` envelope + a status-code table (400/422/404/502/504/429);
  T2.1/T2.2 code against it.
- _Major 5: `GET /api/recipes` had no summary projection / pagination_ → *Fixed*: D1 + T1.4 +
  T5.2 — list returns a **paginated summary projection** (`limit`/`offset`); full object only
  on `/:id`.
- _Minors_ → all fixed: `IHttpClient` name corrected in T3.2 verify; **NFC-normalize** parser
  input (T3.1) + German-aware case-insensitive title search (T5.2); `macroSource` precedence
  rule (any manual edit → "manual"); T1.2 round-trip now covers `tags`/`images` with order
  (added `position` to `recipe_tags`); seed recipes leave `food_id` null + **`ON DELETE
  CASCADE`** on child tables (T1.2/T1.4); `macrosPerServing.calories`↔`cal` mapping noted.
Awaiting a fresh workflow-reviewer pass on these fixes.

**Round 8** (diff-only confirm on the Round 7 fixes): **APPROVE** — all six changes
verified in place and consistent (food_id/foods sequencing, foodId/UUID, group_label,
IHttpClient rename, T4.3 fake seam, DELETE endpoint), no new inconsistency. Reviewer
signature stamped `2026-08-24`. **Human GATE 0 signature still pending.**

**Round 7** (fresh full re-read after the Round 6 fold-in): reviewer **confirmed all B1–B3
and M1–M6 / Q1–Q10 fixes present and sound**, but found **1 blocking + 5 non-blocking** —
mostly exposed by the new relational schema. All addressed:
- _BLOCKING: `food_id` column vs. `foods` FK sequencing (T1.2 creates it, T4.1 also
  "adds" it; FK target absent in M1)_ → *Fixed*: T1.2 creates `food_id` as a plain nullable
  UUID column (no FK); T4.1 creates `foods` (surrogate UUID `id` PK, `code` UNIQUE) and
  **adds the FK constraint** onto the existing column (no duplicate column). D2 updated.
- _NB `foods` key vs. `foodId: uuid?` mismatch_ → *Fixed*: `foods.id` is a surrogate UUID
  PK (= `foodId`), `code` is a UNIQUE column, `recipe_ingredients.food_id → foods.id`.
- _NB `group` is a Postgres reserved word_ → *Fixed*: column renamed `group_label`; JSON
  field stays `group`.
- _NB seam name `HttpClient` collides with Drogon's class_ → *Fixed*: renamed the seam
  **`IHttpClient`** everywhere (D3, T3.2, T4.1, T4.3).
- _NB T4.3 verify didn't name the fake seam_ → *Fixed*: T4.3 reuses `LlmClient` on the
  `IHttpClient` seam and verifies with the fake.
- _NB no `DELETE` endpoint despite a repository `delete`_ → *Fixed*: added
  `DELETE /api/recipes/:id` (+ UI confirm + test) to T2.2.
Awaiting a confirm pass, then the human signature.

**Round 6** (deep external review — two independent passes, one blind cross-check;
`PLAN_REVIEW_recipe_app_foundation.md`): **CHANGES REQUIRED** — the workflow reviewer
(Rounds 1–5) checked structure/consistency well but missed implementability depth and
fact-checking. `approvals.reviewer` was **rolled back to pending**. Findings + resolutions:
- _B1 storage shape never decided (JSONB vs relational)_ → *Fixed*: D2 pins a **relational
  shape** — `recipes` + child tables `recipe_ingredients`/`recipe_steps`/`recipe_images`
  (with `position`) + join table `recipe_tags`; T1.2/T1.3 build to it; T5.2 tag-AND SQL
  uses `recipe_tags`.
- _B2 no HTTP-client seam but tests mock HTTP_ → *Fixed*: D3 adds an **`HttpClient`
  interface** (libcurl impl + fake) that `LlmClient` and the OFF client use; T3.2/T4.1/T4.3
  verify via the fake.
- _B3 UTF-8/emoji/umlaut unaddressed (`std::regex` not UTF-8-aware)_ → *Fixed*: D4/T3.1
  pin **RE2** (added to T1.1 manifest) + codepoint-range emoji stripping + line-start-
  anchored quantities (the `7%` trap); tests assert `Eiweiß`/`Hähnchen`/`7%`.
- _M1 OFF product limit wrong (~100/min) + legacy endpoint pinned_ → *Fixed*: D5/T4.1 make
  **Search-a-licious** primary, legacy `/cgi/search.pl` a fallback, and require confirming
  **current** limits against live docs before sizing backoff (don't hard-code 100 or 15).
- _M2 per-100 g renders "—" for essentially all real recipes_ → **Decided (human): add a
  piece-weight/spoon-volume table** (D5) + use OFF `serving_size` so common piece/spoon
  units resolve; T4.2 verify uses a representative mixed-unit recipe, not a synthetic
  fully-gram one.
- _M3 `execSqlSync` blocks the event loop; "off the event loop" had no mechanism_ →
  *Fixed*: D2 drops the false claim, states blocking is accepted for single-user, and sizes
  the pool via `setThreadNum`.
- _M4 OFF whole-food data quality is the real risk_ → *Fixed*: D5 documents it as a real
  (relocated, not solved) limitation; mitigations = prefer complete-nutriment products,
  manual override, future USDA seam.
- _M5 Ollama OpenAI `response_format` is the less-reliable path_ → *Fixed*: D3 defaults to
  the **native `/api/chat` `format`**=schema, OpenAI surface as fallback.
- _M6 security (upload/SSRF/TLS)_ → *Fixed*: D6/T5.1 — server-generated filenames, ≤8 MB +
  magic-byte type check, **external URL store-only (no fetch)**, nginx serves uploads,
  **basic-auth over TLS** required.
- _Q1 test framework unnamed_ → **Catch2** (T1.1) + truncate-between-tests (T1.3).
  _Q2 scope_ → **Decided (human): keep all six milestones.**
  _Q3 Angular "v17+" not a pin_ → exact `17.x.y` (D1/T2.1). _Q4 valijson draft_ → **Draft
  7** + `$schema` (D2/T1.2). _Q5 model tags_ → already illustrative; T6.2 verifies.
  _Q6 schemaVersion oversold_ → softened (D2). _Q7 migration lock_ → advisory lock +
  per-migration transaction (D2/T1.1). _Q8 Docker build cost_ → vcpkg binary cache + build
  RAM note (D6/T6.1/T6.2). _Q9 3 representations drift_ → noted + codegen as future (D1).
  _Q10 raw vs cooked weight_ → documented caveat (D5)._
Awaiting a fresh reviewer pass on these fixes, then the human signature.

**Round 5** (diff-only confirm on Round 4 fixes): returned APPROVE and was briefly stamped
`2026-08-24` — **superseded/rolled back** after the Round 6 external review found the B1–B3
blocking gaps above. Kept for history.

**Round 4** (re-confirm on this session's deltas — D4 parser scope, D3 LLM policy,
milestone order): **CHANGES REQUIRED** — 1 blocking, 4 non-blocking. All addressed:
- _B1 Goal still listed "website text" as in-scope, contradicting the D4 scope lock_ →
  *Fixed*: struck from the Goal + added website/URL import to the out-of-scope list.
- _NB1 out-of-scope list omitted website/URL parsing_ → *Fixed* (same edit).
- _NB2 "documented fallback" undefined; primary/fallback vs. user-toggle ambiguity_ →
  *Fixed*: D4 states "fallback" = user-selected (no auto rule→LLM handoff), and invalid
  LLM JSON returns a best-effort/empty draft into the editable preview with a warning
  (no auto-retry, no silent save); T3.2 verify updated to assert exactly that.
- _NB3 model tags may not map to real Ollama registry ids_ → *Fixed*: D3 labels the tags
  illustrative; T6.2 must verify the exact `ollama pull` tag for `.env.example`.
- _NB4 `LLM_MODEL` had no defined unset behaviour_ → *Fixed*: D3 states no baked default
  (documentation-only); if unset, the LLM engine is unavailable and the toggle disabled.

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
