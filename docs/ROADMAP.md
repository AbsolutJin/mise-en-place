# mise-en-place — Roadmap

A human-readable, checkable progress tracker. The **spec** is
`docs/plans/PLAN_recipe_app_foundation.md`; this is the **progress view** — tick a box when a
step is done and verified.

**How to use:** `[ ]` = not started · `[~]` = in progress · `[x]` = done & verified.
A task is **done** only when its code is written, its **"Done when"** check passes, and the
change is committed. Milestones close after a `workflow:review` at the boundary.

**Status (2026-08-24):** ✅ **GATE 0 PASSED** — plan **approved** (`approvals.reviewer` +
`approvals.human` both `2026-08-24`, `status: approved`). Human decisions: auth-ready-no-auth;
**seven milestones**; **D1 frozen** (no more stack pivots); TLS stays hand-rolled + the S1
fix. Five external reviews + the internal reviewer folded in. **Ready to start implementation
at Milestone 0 / T0.1.**

> Stack (FROZEN): **from-scratch C++ backend** (own HTTP server/router/JSON/schema/
> DB-over-libpq/HTTP-client), Angular SPA, PostgreSQL; externals via **system apt** (libpq,
> OpenSSL, utf8proc, Catch2 — **no vcpkg**); dev in **WSL2**.

---

## Phase 0 — Planning ✅ (GATE 0 passed 2026-08-24)

- [x] Understand — intent + repo context
- [x] Plan written — `docs/plans/PLAN_recipe_app_foundation.md`
- [x] Recipe format + product scope locked
- [x] **Five external reviews** folded in (`docs/reviews/`) — incl. the from-scratch pivot,
      the vcpkg drop, and the M0-security pass
- [x] Reviewer approval (`approvals.reviewer: 2026-08-24`)
- [x] **Human GATE 0 signature** (`approvals.human: 2026-08-24`) — auth-ready-no-auth · seven
      milestones · D1 frozen · TLS hand-rolled + S1 fix · TLS/basic-auth caveat accepted

---

## Milestone 0 — Core backend libraries (from scratch) ⚪

- [ ] **T0.1 — Toolchain + skeleton** — plain **GNU Makefile** (C++20) finding apt `libpq-dev`/`libssl-dev`/`libutf8proc-dev`/`catch2` via `pkg-config` (Linux-only build → Make over CMake); `/lib` (`net router json jsonschema db httpclient`) + `/app` + `/tests`; **ASAN+UBSAN (+TSan) build** (`make asan`; TSan for the T0.9 + T0.6 pools); dev `docker-compose` Postgres. WSL2. **Done when:** `make` builds; a Catch2 test runs green under ASAN+UBSAN; the dev Postgres comes up.
- [ ] **T0.2 — `net` (HTTP/1.1 server, single-threaded)** — socket listener, request parser (CL + chunked), response writer, keep-alive (**single-threaded — thread pool deferred to T0.9**); hardening: size caps, timeout (anti-slowloris), **anti-smuggling framing** (CL+TE, dup CL, bare CR/LF/NUL, overflow); **fuzz harness**. **Done when:** malformed/smuggling vectors → 400; oversized/chunked-over-cap rejected not OOM'd; stalled conn times out; fuzzer clean for a budget.
- [ ] **T0.3 — `router`** — method+path (`:id` params) → handler; 404/405; error + warning envelope helpers. **Done when:** routing/param/404/405 tests + envelope-shaped error body pass.
- [ ] **T0.4 — `json`** — own parser+serializer (unicode escapes incl. **surrogate pairs**, numbers, bool/null), **recursion-depth cap**; **fuzz harness**. **Done when:** round-trips (incl. surrogate pairs) pass; malformed → clean error; past-cap depth rejected; fuzzer clean.
- [ ] **T0.5 — `jsonschema`** — own validator for the authored subset (types/enum/required/arrays/**`type:[…,"null"]` union**/`$ref`), reports the failing path; tested against a **generic fixture schema** (real Recipe schema is T1.2). **Done when:** valid fixture accepted; each violation class (+ null in non-nullable) rejected with path; null accepted in a nullable field.
- [ ] **T0.6 — `db` (libpq)** — connection pool, **parameterized `exec`**, result→row mapping, transaction helper, standalone `connect()` for the migration lock, and a **migration-only raw `PQexec`** path. **Done when:** parameterized round-trip; SQL-metachar bind stored literally; pool concurrency; standalone connect works.
- [ ] **T0.7 — App wiring + `/health`** — `main()` starts server, mounts router, builds db pool from env, logging + config. **Done when:** app boots; `GET /health` → 200 through our server; DB connect logged.
- [ ] **T0.8 — `httpclient`** — own HTTP/1.1 client; chunked decode; body cap; **HTTPS via OpenSSL with HOSTNAME verification (`SSL_set1_host` + `X509_V_OK`)** for OFF, plain HTTP for Ollama; timeouts. **Done when (real network):** HTTPS GET verified; **valid-CA-wrong-hostname rejected**; plain-HTTP local GET; chunked decode; fake stays the unit double.
- [ ] **T0.9 — `net` concurrency (thread pool, built LAST)** — fixed-size worker pool; accept loop dispatches each connection to a worker that owns one `db` (T0.6) connection per request; bounded queue; clean drain/join shutdown; **built + tested under TSan**. **Done when:** concurrent keep-alive clients handled with no interleaved/corrupted responses; TSan clean under load; queue bounded; clean shutdown, ASAN clean.
- [ ] **🚦 M0 review** — boundary review passes _(largest + most security-exposed part — read carefully)._

---

## Milestone 1 — Schema, DB, browse API (on the M0 libs) ⚪

- [ ] **T1.1 — Migration runner** — `schema_migrations`, per-migration transaction, session advisory lock on the dedicated connection. **Done when:** applies on boot; no re-apply; a failing migration rolls back cleanly.
- [ ] **T1.2 — `Recipe` schema + migrations + model round-trip** — authored schema (+ `definitions/Macros`); relational tables (child tables + `recipe_tags`, `ON DELETE CASCADE`, `macros_estimated` col); own JSON assembly + validator. **Done when:** migrations apply; round-trip covers grouped/to-taste/ordered+empty steps/tags+images order/`macrosEstimated`; validator rejects invalid.
- [ ] **T1.3 — Repository/service (CRUD) + PUT full-replace** — food_id picks survive; `macrosEstimated` recompute except when `macroSource=manual`; image-file GC ordering. **Done when:** CRUD + PUT-preserves-picks + per-100g null-path integration tests pass.
- [ ] **T1.4 — Browse controllers + seed** — `GET /api/recipes` (paginated summary) + `/:id` (full); seed with `food_id` null. **Done when:** list honors limit/offset; `/:id` returns schema-valid full Recipe.
- [ ] **🚦 M1 review**

---

## Milestone 2 — Frontend scaffold, browse/detail, structured form ⚪

- [ ] **T2.0 — UI mockups** (`docs/mockups/`) — screens + states agreed before build.
- [ ] **T2.1 — Angular scaffold + browse/detail** — exact-pinned Angular; dev proxy; typed API service on the error envelope; browse (paginated summaries) + detail (macros, `—`/estimated marker, `macroSource` badge, tags/favorite/notes/times, ingredient notes). **Done when:** `ng build`/`ng test` green; list+detail render a seeded recipe.
- [ ] **T2.2 — Add/Edit form + DELETE** — Reactive Forms incl. **tags/favorite/notes/times** (the only way they're set); `POST`/`PUT /api/recipes/:id`; `DELETE …/:id`→204. **Done when:** save persists; tags+favorite round-trip and are found by T5.2 filters; 422 on invalid; PUT preserves picks; delete cascades.
- [ ] **T2.3 — Manual macro entry + override.** **Done when:** manual macros persist + render; per-100g "—" with no weight.
- [ ] **🚦 M2 review** _(M1+M2 = first end-to-end vertical slice)_

---

## Milestone 3 — Paste import with selectable engine ⚪

- [ ] **T3.1 — `RuleBasedParser` + `POST /api/parse`** — social captions; **utf8proc NFC** then own UTF-8 scanning; sections/qty-unit/macro block/to-taste/parenthetical→note; **`/api/parse` returns a partial draft, never 422**. **Done when:** 3 fixtures parse (grouping, null-qty, macro block, empty steps, umlaut/NFD, `7%` trap); partial caption → draft+warning.
- [ ] **T3.2 — `LlmClient` + `LlmParser` (fallback)** — on `IHttpClient` (T0.8); Ollama native `/api/chat format=schema`; invalid → empty/partial draft + warning. **Done when:** fake-client test: valid accepted, malformed → documented fallback.
- [ ] **T3.3 — Paste screen** — textarea + engine toggle → editable preview → save. **Done when:** rule-based paste yields a pre-filled saveable form.
- [ ] **🚦 M3 review**

---

## Milestone 4 — Macros from Open Food Facts + LLM estimate ⚪

- [ ] **T4.1 — `NutritionSource` + OFF client + `foods` cache + converter** — OFF v2 `/api/v2/search` (verify limits) on T0.8 HTTPS; `foods` (UUID id PK, code UNIQUE) + FK add; local-first; per-100g mapping; unit→gram (density + piece/spoon tables). **Done when:** fake-client candidates; cache hit → no call; kJ-only converted/rejected; unit + piece/spoon conversions incl. flagged path.
- [ ] **T4.2 — Macro engine** — sum → per-serving + per-100g; unpicked/flagged → "—"; sets `macrosEstimated` when piece/spoon fed it. **Done when:** exact-grams matches hand-computed (`estimated:false`); piece/spoon full weight (`estimated:true`); flagged → "—".
- [ ] **T4.2b — `POST /api/macros/compute` + search-and-pick UI** — sets `macroSource:"ingredients"`. **Done when:** in-form search→pick→fill; compute correct + provenance set.
- [ ] **T4.3 — `POST /api/macros/estimate` + LLM button** — sets `macroSource:"llm"` + `macrosEstimated:true`. **Done when:** fake-LLM fills + asserts provenance; user can override.
- [ ] **🚦 M4 review**

---

## Milestone 5 — Media, polish, search ⚪

- [ ] **T5.1 — Image upload + lifecycle** — adds a multipart parser into `net`; server-gen filename, ≤8 MB **enforced during streaming**, magic-byte type; external URL store-only (no SSRF); owned-file GC on delete/PUT. **Done when:** valid upload renders; oversized/wrong-type/traversal rejected; external never fetched; delete/PUT unlinks owned files.
- [ ] **T5.2 — Search / filter / sort** — case-insensitive `ILIKE` (UTF-8 locale, no accent-fold, no extension); tag AND-filter; favorites; minProtein/maxCalories; sort; on the clamped paginated list. **Done when:** subset tests (incl. `Ä`↔`ä`, accent-fold-NOT-applied control), sorts, and limit-clamp pass.
- [ ] **🚦 M5 review**

---

## Milestone 6 — Deployment & docs ⚪

- [ ] **T6.1 — Docker + nginx + compose** — multi-stage Dockerfile on a **digest-pinned base**; build stage apt-installs deps; slim runtime ships `libpq5`/`libssl`/`libutf8proc`/`ca-certificates`; Postgres init with a **UTF-8 locale**; nginx `/api` + `/uploads` + `/health`; shared `uploads` volume across backend+nginx. **Done when:** `compose up` serves; browse persists; `/api/*`,`/uploads/*`,`/health` reachable; backend-written image served by nginx; in-container OFF search works (CA trust).
- [ ] **T6.2 — Docs** — env vars, Ollama pointer (verify pull tag), backup, exact apt list + digest, min build RAM, "Docker build is the source of truth". **Done when:** copy-pasteable sequence works: `compose up` → `curl …/health` 200 → POST a recipe → appears in list.
- [ ] **🚦 M6 review**

---

## Definition of done (whole phase)

- [ ] All seven milestones complete (M0 + M1–M6), each with its boundary review passed
- [ ] `docker compose up` runs the full app on a home server / VPS behind VPN or basic-auth **over TLS**
- [ ] Docs let a fresh setup reach a working app from scratch
- [ ] Plan + reviews archived to `docs/archive/` (per `docs/archive/README.md`)
