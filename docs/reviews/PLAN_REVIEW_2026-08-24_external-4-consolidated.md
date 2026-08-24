# Review: PLAN_recipe_app_foundation + ROADMAP

**Date:** 2026-08-24
**Scope:** `docs/plans/PLAN_recipe_app_foundation.md`, `docs/ROADMAP.md`, `docs/handoff/`, `docs/reference/`, `docs/mockups/`, `docs/reviews/` (branch `recipe-app-foundation`)
**Method:** Own read of all docs + two independent cross-check passes (technical fact-check vs. live sources; structure/consistency/implementability). The two passes received only the sources and the task — not each other's or the reviewer's reasoning — per an independent-refutation rule. They are complementary and converged independently on the migration-concurrency over-investment point.

---

## Overall verdict

The plan is **technically strong and ready to implement at Milestone 1 today**. It went through 12 internal + 3 external review rounds, and it shows: nearly every external-API and library claim verifiable against live sources checked out (OFF rate limits, valijson draft support, Ollama structured output, kJ/kcal, `pg_advisory_lock`, RE2). Those are exactly the areas where plans of this kind usually fail.

**However, the plan is not end-to-end internally consistent.** The last fold-ins (3rd review) introduced localized contradictions that would bite a fresh coder at **M2 / M5 / M6**. None blocks M1; none requires re-opening the architecture. The right response is **one corrective plan-text pass before M2 begins** — not a full re-review.

---

## 🔴 Real contradictions (fix before M2)

### 1. `favorite`, `tags` (+ `notes`, `prepTimeMin`, `cookTimeMin`) have no write path
The locked `Recipe` format includes these fields. But **T2.2** enumerates the add/edit form fields explicitly and **omits all five**. **T2.1** (detail render) also omits `tags`/`favorite`/`notes`/times. Meanwhile **T5.2** (GATE-0-locked) ships a favorites-only toggle and a tag AND-filter, and **D1's summary projection** carries `favorite`+`tags` for the list cards. The paste parser is explicitly forbidden from auto-tagging.

**Consequence:** No path anywhere in the plan ever sets a tag or a favorite. The T5.2 filters are dead on arrival (always empty result); the list/detail favorite star and tag chips render never-settable fields. The mockups README lists these in the form — the *intent* exists; the task specs just don't deliver it.
**Fix:** Add `tags`, `favorite`, `notes`, `prepTimeMin`, `cookTimeMin` to the T2.2 form field list and the T2.1 render list.

### 2. PUT recompute of `macrosEstimated` contradicts the "manual edit clears the flag" rule
Format (lines ~417–431): a manual macro/`totalWeightG` edit flips `macroSource` → `"manual"` **and clears `macrosEstimated` to false** (the user asserts exact values). T1.3: on PUT the backend **recomputes `macrosEstimated` from the payload, not trusting the client flag.**

**Consequence:** For the common *compute-then-override* path, ingredients still carry piece/spoon units → recompute yields `true`, overriding the user's manual `false` → state `macroSource="manual"` + `macrosEstimated=true`, i.e. exactly the "guessed shown as exact" state the flag (2nd-review Major 1) was introduced to prevent.
**Fix:** One sentence — suppress the PUT recompute when `macroSource=manual`.

---

## 🟠 Major (deploy / verify / dependency gaps)

### 3. NFC normalization is mandated (T3.1) but no normalization library is in the T1.1 manifest
T3.1 requires "NFC-normalize first" + an NFD test case. `vcpkg.json` lists only `drogon[postgres]`, `valijson`, `libcurl`, `re2`, `Catch2` (+ vendored jsoncpp). **None of these normalizes Unicode** — RE2 matches UTF-8 but does not normalize. The step and its test cannot be built as written.
**Fix:** Add `icu` or (lighter) `utf8proc` to the T1.1 manifest and name it as the normalizer.

### 4. Uploads directory not declared as a shared volume between backend and nginx
D6: the **backend** writes uploaded files (Drogon multipart → disk); **nginx** serves them (`location /uploads/`) from the frontend container. T6.1's service list names only "backend, frontend, postgres volume". Without a shared `/uploads` volume mounted into both containers, nginx can't see backend-written files (404) and uploads are lost on container recreation.
**Fix:** Declare a named `/uploads` volume mounted into both backend and frontend containers in T6.1.

### 5. `/health` is unreachable through the documented deploy, yet the smoke test curls it
nginx proxies only `/api/` and `/uploads/`. T6.2's acceptance starts with `curl /health → 200`. In prod only nginx is public (no backend host port stated) → the first step of the copy-pasteable smoke test fails.
**Fix:** Move `/health` under `/api/health`, or add an nginx `location /health`, or publish + curl a backend port in T6.2.

### 6. ß/umlaut search mechanism (T5.2) is mechanically wrong / version-dependent
T5.2 proposes "`ILIKE` under a case/umlaut-appropriate collation or `citext`". Two verified problems: `citext` only case-folds via `lower()` — it does **not** strip accents (ä↔a, ß↔ss need the `unaccent` extension); and `ILIKE` + a nondeterministic (accent-insensitive) ICU collation **is rejected by PostgreSQL before v18**. The plan pins no PG version → latent failure.
**Fix:** Decide intent. Plain case-insensitivity (Ä↔ä) works with `ILIKE` under a UTF-8 `lc_ctype`. Accent folding needs `unaccent()` (any version) or PG18+ with a nondeterministic collation. Drop `citext` as the accent answer — it isn't one.

---

## 🟡 Minor (consistency nits)

- **PUT URL disagreement:** T2.2 writes `PUT /api/recipes` (no `:id`); D1 + ROADMAP write `…/:id`. T2.2 itself requires "unknown id → 404", which needs the id in the URL. Self-contradictory.
- **`macroSource` never set by compute/estimate tasks:** T3.1 sets `"manual"`, but no task states T4.2 sets `"ingredients"` or T4.3 sets `"llm"`. The T2.1 detail badge then displays nothing reliable.
- **ROADMAP invents T2.0 (UI mockups)** not present in the plan. The ROADMAP calls itself a faithful tracker but here tracks scope the plan lacks. Reconcile which document owns the mockup step.
- **`/api/parse` behavior on a partial rule-based parse is unpinned:** only the LLM-invalid path (T3.2) defines the "200 + editable draft + warning" fallback. A partial rule-based parse (e.g. no title) would be `422`'d under strict boundary validation, though it's meant to land in the editable preview. Pin that `/api/parse` returns a partial draft, not 422.
- **valijson does not read `$schema`:** D2's "matching `$schema`, else under-validation" misdescribes the mechanism (the draft is fixed by the `SchemaParser` Version enum; Draft 7 is the default). Outcome is fine; the cause-and-effect is misleading — a coder may waste time expecting auto-draft-selection.
- **Handoff §3 is stale:** describes LLM via OpenAI-compatible `/v1/chat/completions`, but D3 defaults to native Ollama `/api/chat format=schema`. The handoff is the "read first" doc — could mislead the next implementer.
- **Fixture teaches tag extraction the parser doesn't implement:** `example-recipes.md` Example 1 says "`(Meal Prep)` → a tag", but T3.1 has no tag rule (tags are user-curated).
- **Dedicated-connection acquisition unspecified:** the clean way is a separate `DbClient` with `connectionNumber=1`, not a pinned pool connection (which reopens the race D2 fixes). Also add a "do not use a Drogon *fast* DbClient with `execSqlSync`" note (self-deadlock).
- **Model tags** (`qwen3.5-9b`, `qwen3.8-27b`, "Gemma 4 27B") match no known Ollama registry ids — but already hedged as "illustrative" behind the T6.2 verify gate, so risk is contained.
- **Plan omits per-step `Status:` blocks / Breaking-Changes section** (house template); status lives entirely in the ROADMAP → two sources of truth. Path drift: ROADMAP DoD says `docs/archive/` while elsewhere `docs/archived/plans/`.

---

## Scope / meta observations

- **Migration-concurrency is over-invested.** Flagged independently by both cross-check passes: the dedicated-connection `pg_advisory_lock`-held-across-per-migration-transactions machinery guards against two concurrently booting backend instances — impossible in the described single-container compose deploy. Correct and cheap, but consumed disproportionate review energy (Rounds 6–9). Mark it as "kept for future scaling" rather than silent gold-plating.
- **Backups are documentation-only (T6.2)** — the only data-loss protection for a curated store is prose guidance. Acceptable this phase, but a conscious choice.
- **Planning-to-code ratio = everything-to-nothing.** 903-line plan, 15 rounds, 0 lines of code. If the stated goal is "learn C++", C++ isn't learned in the plan — T1.1 should be built now.
- **The workflow plan-reviewer was unreliable.** The handoff admits it: every external review found blockers *after* the internal reviewer said APPROVE. This consolidated cross-check found, *after* three external reviews, one more factual blocker (NFC lib) and one structural blocker (tags/favorite). Do not trust the automated review alone.
- **Three hand-synced `Recipe` representations** (JSON Schema, C++ struct, TS type), no codegen — drift risk accepted as Q9, but the T1.2 round-trip test covers only C++↔JSON; the TS side is untested.
- **Steep learning curve + first-build risk:** Drogon via vcpkg compiles the whole dependency tree from scratch (slow, OOM risk on a small VPS, acknowledged as Q8). T1.1 (toolchain) is realistically the hardest step, not the easiest.

---

## What is genuinely strong

- The hard traps are handled correctly: kJ/kcal mapping, OFF v2 `/api/v2/search` vs. Search-a-licious, `product_quantity` forbidden as per-piece weight, NFC + RE2 + line-anchored quantities, SSRF (external URLs store-only), magic-byte upload validation, TLS-mandatory caveat, `food_id`→`foods.id` FK sequencing.
- Relational schema is well-pinned (positions, cascade, `group_label` reserved-word fix, per-100 g **derived** and locked in T1.2).
- T4.2 verify uses **hand-computed** expected values instead of the converter's own constants — avoiding the tautology the earlier ±5% test had.
- The review loop **verifiably converged** (3+6 → 1+5 → 0+1 blocking/major), and the fold-ins are actually present in the body.

### Verified correct against live sources (technical pass)
OFF rate limits (15/min product, 10/min search) · OFF endpoint/service split · OFF nutrient mapping · valijson Draft 7 default + `definitions`/`$ref` · Ollama native `format` more reliable than OpenAI-compat `response_format` (ignored by some models) · `pg_advisory_lock` session-scope design · `std::regex` not UTF-8-aware → RE2 · `group` reserved word → `group_label` · Drogon `execSqlSync` blocking + `setThreadNum` + `drogon[postgres]` feature required · Angular exact-pin + `proxy.conf.json` + `ChromeHeadlessNoSandbox` · per-100 g formula `perServing × servings ÷ totalWeightG × 100`.

---

## Recommendation

1. **Before M2:** one corrective plan-text pass (not a full re-review) for the two red contradictions + the four major points. All localized edits.
2. **M1 can start immediately** — none of its tasks are affected. Precondition: a working C++ toolchain (CMake / vcpkg / Docker).
3. Carry the Handoff §3 (LLM) and ROADMAP↔plan divergences (T2.0, path drift) along.

> Note: this is a GitHub repo outside the weblab/SLA world, so the "findings → GitLab issue" pipeline does not apply here.
