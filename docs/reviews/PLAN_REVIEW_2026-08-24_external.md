# Plan Review — `PLAN_recipe_app_foundation.md`

**Date:** 2026-08-24
**Method:** Two independent review passes (one primary, one blind cross-check that received only the plan + fixtures, not the primary's reasoning). High convergence — ~11 findings surfaced by both passes independently, which is the signal they are real and not an artifact of a single reasoning chain.

---

## Verdict: CHANGES REQUIRED — not as approval-ready as the handoff claims

The plan is above-average in thoroughness, and the five prior review rounds caught real bugs (kJ/kcal factor, the total-weight invariant). **However**, there are three blocking gaps the prior reviewer missed, one factual error, and one over-promised headline feature. The existing `approvals.reviewer: 2026-08-24` stamp is over-confident; the plan should be revised before human sign-off.

---

## Correctness check of the pinned technical facts

| Claim | Verdict |
|---|---|
| `drogon[postgres]` vcpkg feature | **Correct** — feature is named `postgres`, maps to `BUILD_POSTGRESQL`, pulls `libpq`. |
| valijson header-only, in vcpkg, jsoncpp adapter | **Correct** (draft caveat — see Q4). |
| Drogon `execSqlSync` exists | **Correct** (but see M3). |
| OFF nutrient fields (`energy-kcal_100g`, `energy_100g` in kJ, `proteins/carbohydrates/fat_100g`), kJ→kcal ÷ 4.184, descriptive User-Agent requirement | **Correct**. |
| per-100 g formula `perServing × servings ÷ totalWeightG × 100` | **Mathematically correct**. |
| OFF **search** rate limit ~10/min | **Correct**. |
| OFF **product** rate limit ~100/min | **Wrong / too high** — see M1. |
| Ollama `/v1/chat/completions` + `response_format` structured output | **Partly correct but risky** — see M5. |

---

## Blocking — a fresh instance cannot cleanly implement M1/M3/M4 as written

### B1 — Storage shape is never decided
The handoff insists storage is "relational Postgres, not a JSON blob", but the plan never decides whether `ingredients` is a child table (`recipe_ingredients` with FK, ordering, a `group` column) or JSONB — nor whether `tags` / `steps` / `images` are `text[]`, JSONB, or join tables. This is *the* central M1 schema decision (T1.2 / T1.3) and it cascades directly into T5.2 (the tag AND-filter and macro-range SQL depend entirely on this choice). Without it, the coder must invent it — exactly what a self-contained plan is supposed to prevent.

### B2 — No HTTP-client seam, yet the test plan depends on mocking
libcurl is called directly (synchronous) inside `LlmClient` and the OFF client. But T3.2, T4.1, and T4.3 all verify via a "mocked HTTP/LLM/OFF response". Direct libcurl is not mockable without either an injected HTTP-client interface or a local test HTTP server — neither is specified. The test strategy assumes a seam the design does not provide; the coder hits this wall in M3/M4. Decide the seam (interface + fake, or a loopback server) in the plan.

### B3 — C++ text handling for the rule parser (UTF-8, emoji, umlauts) is unaddressed
T3.1's core job is emoji stripping (🍗💪🛒), umlaut/ß-tolerant matching (`Eiweiß`, `Hähnchen`, `Kohlenhydrate`), and `%`/`€` noise. `std::regex` is **not** UTF-8-aware and byte-matches under the default locale — this is exactly where it breaks. No UTF-8 strategy (ICU, RE2, or deliberate byte-level handling) is named, and no such library is in the T1.1 vcpkg manifest. Also lurking: `140ml Kochsahne 7%` must not read `7` as a quantity (anchor the quantity at line start). The plan treats the hardest part of the parser as trivial regex.

---

## Major — does not block the start, but will bite later

### M1 — OFF product rate limit stated wrong, and the pinned search endpoint is legacy
D5/D6 pin "~100 req/min product." The current official OFF limit for product reads is materially lower (reported by the cross-check as ~15/min, with a source) — building a backoff budget around 100 invites an IP ban. **Verify against the current OFF docs before building the backoff — do not blindly adopt 15 either.** Separately, the pinned `/cgi/search.pl?search_terms=…` is documented as "Legacy, not recommended for new integrations"; full-text search is directed to Search-a-licious. The plan hedges this only parenthetically yet still pins the deprecated path in T4.1.

### M2 — The headline "per 100 g" feature will render "—" for essentially all real recipes
`totalWeightG` auto-populates only when *every* ingredient is picked *and* every unit resolves to grams. The fixtures are dominated by exactly the units that cannot resolve: `2 Knoblauchzehen`, `1 rote Zwiebel`, `1 medium red bell pepper`, `1 TL/EL`, `tbsp`, assorted `ml` of liquids, plus to-taste rows. Piece weights aren't reliably available from OFF, and the density table covers only "common liquids". So the Goal's promise of "macros … per 100 g" will be "—" for virtually every realistic recipe. Worse, T4.2's verification uses a synthetic "fully-picked, gram-resolved" recipe, so the suite goes green while the real feature is dark. Technically safe (shows "—"), but promise ≠ reality and the verification is not representative.

### M3 — `execSqlSync` blocks the Drogon event loop; "run off the event loop" has no mechanism
D2 says DB access uses `execSqlSync` "run off the event loop." Drogon request handlers run *on* the event-loop threads by default, and `execSqlSync` blocks the calling thread. Combined with synchronous libcurl (OFF/LLM), every DB and outbound call blocks a loop thread. For a single user this is practically fine — but the plan states a property ("off the event loop") it never implements (no worker pool, no `setThreadNum` sizing). Either drop the claim or specify the mechanism. Side effect: Drogon's async value is entirely unused (see Q2).

### M4 — OFF data quality for whole foods is the real risk core
D5 justifies OFF on German coverage and declares the accuracy risk solved by "human picks." But the fixtures are almost entirely whole foods (chicken, steak, rice, onion, garlic, bell pepper) — and OFF is a *branded-product* database with sparse, uneven, user-contributed per-100 g data for exactly these. Search-and-pick removes the *auto-match* error, not the data-quality problem: the user will be picking among branded junk for "chicken breast." The risk is relocated, not eliminated.

### M5 — Ollama structured output: the chosen path is the less-reliable one
D3 banks JSON reliability on `response_format` over the OpenAI-compatible `/v1/chat/completions`, explicitly chosen over native `/api/chat`. That OpenAI `json_schema` path is reported (current bug reports) to be *ignored* for various models, while the native `format` parameter (which takes a JSON schema directly) is the robust one. valijson + fallback prevents crashes, but expect frequent fallback-to-empty-draft. Reconsider the native `format`.

### M6 — Security surface for an unauthenticated, phone-reachable write + upload app
D6 offloads all protection to a human-provisioned VPN/basic-auth caveat; nothing is enforced in-app. Specific gaps:
- **(a)** Multipart image upload — no path-traversal handling of the client filename, no concrete size/type limits ("validated" with no numbers).
- **(b)** "accept an external image URL" — unclear whether the backend *fetches* it (SSRF against the home LAN / Ollama / Postgres) or only stores it for the browser.
- **(c)** Who serves the uploads directory in prod (nginx location vs Drogon) is unspecified.
- **(d)** basic-auth *without TLS* over a phone-reachable link ships credentials in clear — the caveat names basic-auth but not HTTPS.

---

## Minor / Quality

- **Q1 — C++ test framework not chosen.** T1.1 lists "a test framework" but never names one (Catch2 / GoogleTest / doctest), affecting CMake wiring. Also no integration-DB reset strategy (truncate / rollback / recreate → determinism).
- **Q2 — Scope vs. the stated "learn C++" goal.** Six milestones covering two parse engines, an LLM client, OFF caching + unit conversion, Docker multi-stage, and nginx is a large surface for a C++ beginner. Because everything is synchronous (M3), Drogon's async value is unused. M1+M2 (scaffold + CRUD + form) already are the C++ learning core and the first end-to-end slice; the LLM parser/estimator could be trimmed out of the *foundation* scope. Worth a conscious conversation, not silent acceptance.
- **Q3 — "Pin Angular v17+" is a floor, not a pin.** A real pin is a specific version (e.g. 17.x); `v17+` is unbounded and lets a fresh build pull a newer major with different builder/test defaults — the exact drift the pin was meant to prevent.
- **Q4 — valijson JSON Schema draft unspecified.** valijson supports Draft 7 and a Draft 4 subset (not 2019-09 / 2020-12). The schema must be authored to a supported draft and carry the right `$schema`, or it silently under-validates.
- **Q5 — Reference LLM model tags are speculative** (`qwen3.5-9b`, `qwen3.8-27b`, "Gemma 4 27B"). Already labelled "illustrative"; T6.2 must verify the exact `ollama pull` tag, or a `.env.example` with an unresolvable default silently disables the LLM engine.
- **Q6 — `schemaVersion` is largely vestigial under relational storage** — a version bump becomes a SQL migration regardless; it only earns its keep at the export/interchange layer. The plan oversells it.
- **Q7 — Migration runner has no advisory lock** (safe for one instance, races if two boot concurrently). Also "idempotent" is really version-tracking, not idempotency of the DDL itself — each migration should be wrapped in a transaction and its version recorded only on success.
- **Q8 — Docker + vcpkg build cost.** The build compiles Drogon's whole dependency tree from scratch — slow and memory-hungry; on a small VPS it risks OOM, despite the "runs on a home server / VPS" promise. No binary caching / prebuilt-deps base image is mentioned.
- **Q9 — Three `Recipe` representations kept in sync by hand** (JSON Schema, C++ struct, TS type). D1 accepts this, but with no codegen step (e.g. json-schema-to-typescript) it is a drift magnet.
- **Q10 — per-100 g is derived from the *raw* ingredient-sum weight** ≠ cooked weight (water evaporates), so computed per-100 g understates the finished dish. Acceptable but undocumented.

---

## Bottom line

The plan is good — but not approval-ready. The three blocking items (storage shape, HTTP mock seam, UTF-8 parsing) are genuine "cannot implement as written" gaps that five review rounds missed, plus a factual error (OFF limit) and an over-promised headline feature (per-100 g).

Two honest meta-points:
1. The single most valuable finding (**B1**, storage shape) came from the independent pass, and that pass also corrected a factual claim (**M1**, the OFF limit) — the cross-check was not ritual here; it demonstrably improved the review.
2. If this plan runs through the `claude-tools` workflow and its `plan-reviewer` stamped APPROVE with B1–B3 open, the reviewer prompt is worth questioning — it evidently checks structure/consistency well but implementability depth and fact verification less so.
