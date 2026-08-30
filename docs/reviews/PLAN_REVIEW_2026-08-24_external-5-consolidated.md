# Review (5th): PLAN_recipe_app_foundation + ROADMAP — from-scratch pivot

**Date:** 2026-08-24
**Scope:** `docs/plans/PLAN_recipe_app_foundation.md` (1104 lines), `docs/ROADMAP.md`, `docs/handoff/`, `docs/reference/`, `docs/mockups/` (branch `recipe-app-foundation`, at `60ec528`)
**Method:** Own read of all docs + two independent cross-check passes (technical/security fact-check; structure/consistency/implementability). Each pass received only the sources and the task — not the other's or the reviewer's reasoning — per an independent-refutation rule. Solution proposals are included inline, as requested.

---

## Context — what changed since the 4th review

This plan pivoted **twice** since `external-4-consolidated` was folded in:
1. **From-scratch backend** — dropped the Drogon framework; the backend now builds its own HTTP/1.1 server, router, JSON parser+serializer, JSON-Schema validator, libpq DB wrapper, HTTP client (OpenSSL TLS), and multipart parser. A new **Milestone 0 (T0.1–T0.8)** was inserted for these core libraries.
2. **Dropped vcpkg** — the three remaining externals (libpq, OpenSSL, Catch2 — plus utf8proc) are now installed via system `apt` packages.

**All 4th-review findings are genuinely resolved** in the plan body — verified this pass:
`tags`/`favorite`/`notes`/times now have a write path (T2.2) and render path (T2.1); PUT recompute is suppressed when `macroSource=="manual"` (T1.3); `utf8proc` added for NFC (T0.1/T3.1); `/health` reachable via nginx (T6.1); uploads on a shared named volume (T6.1); ß/umlaut search decided as plain case-insensitive `ILIKE`, `citext` rejected (T5.2); `macroSource` set by T4.2b/T4.3; `/api/parse` returns a partial draft not `422` (T3.1); T2.0 reconciled into the plan.

---

## Overall verdict

The **plan body is in genuinely good shape** — internally consistent on the hard parts (macro provenance, storage shape, error/warning envelopes, the M0 library set), clean of old-stack assumptions, and its M0 verification steps are behavior-based, not existence-theater. It is **close to implementable but not ready to sign and start.** The blockers fall in two groups: (a) **security gaps in the hand-rolled networking code** that would ship silent vulnerabilities, and (b) **document/process inconsistencies** (a wholesale-stale ROADMAP, a wrong milestone count at the pending sign-off, two M0 tasks that aren't self-contained). None require re-opening the architecture; all are fixable in roughly a day plus one review.

The honest meta-point: the from-scratch M0 is the **largest, most security-exposed, and least-reviewed** part of the whole plan — yet it rides into implementation on internal-reviewer approval alone. The pending 5th external review is warranted precisely there.

---

## 🔴 Blocking — security in the hand-rolled networking (fix before M0 ships)

### S1. `T0.8` TLS: "certificate verification ON + SNI" does NOT verify the hostname
`SSL_CTX_set_verify(SSL_VERIFY_PEER)` checks the certificate *chain* (signature path + validity) but **not** that the cert matches the host you dialed. **SNI (`SSL_set_tlsext_host_name`) is unrelated** — it tells the server which cert to serve; it has zero effect on client verification. As written, any attacker holding *any* CA-valid cert for *any* domain passes → silent MITM on the OFF path (public internet; the VPN protects inbound, not the backend's outbound TLS).
**Fix:**
- Call `SSL_set1_host(ssl, hostname)` (or `X509_VERIFY_PARAM_set1_host()` via `SSL_get0_param()`) **before** the handshake, in addition to `SSL_VERIFY_PEER` and SNI; then assert `SSL_get_verify_result(ssl) == X509_V_OK`.
- Load the trust store explicitly: `SSL_CTX_set_default_verify_paths()` (or `load_verify_locations` at `/etc/ssl/certs`).
- **Name `SSL_set1_host` in T0.8** so the implementer doesn't stop at `SSL_VERIFY_PEER`.
- Change the verify test to a **valid-CA-but-wrong-hostname** host (e.g. `wrong.host.badssl.com`) — only that case distinguishes chain-only (vulnerable) from real hostname verification. A self-signed/expired host would pass while the code is still exploitable.

### S2. No memory-safety tooling anywhere (ASAN/UBSAN/fuzzing) for a beginner hand-rolling four untrusted-input parsers
T0.2 (HTTP parser), T0.4 (JSON parser), T0.6 (threaded pool), T5.1 (multipart) all take untrusted bytes. Buffer overreads/off-by-ones and data races are the #1 realistic bug class here, and unit tests miss exactly these. This is the highest-ROI safety net and it's absent.
**Fix:**
- Add an **ASAN+UBSAN** build config in T0.1 (`-fsanitize=address,undefined`) and run the Catch2 suite under it; add **TSan** coverage for the T0.6 pool.
- Add a **libFuzzer/AFL++** harness on the T0.2 HTTP parser and T0.4 JSON parser as a named task + verify item (a few hours of fuzzing finds crashes tests never will).

---

## 🟠 Major — process & document consistency

### P1. `ROADMAP.md` is wholesale stale — still the old Drogon/vcpkg 6-milestone plan, and it omits M0
Confirmed on disk. The ROADMAP still says: T1.1 "Drogon skeleton"; build line "`vcpkg.json` (`drogon[postgres]`, `valijson`, `libcurl`, `re2`, Catch2)"; "`setThreadNum`"; T1.2 "Draft-7 … jsoncpp … valijson"; T3.1 "NFC → RE2"; T6.1 "vcpkg binary cache"; T6.2 "vcpkg build notes". **Milestone 0 is entirely absent.** It also **falsely reports GATE 0 as passed** ("GATE 0 passed 2026-08-24", "[x] Human GATE 0 approval … status: approved") while the plan frontmatter is `human: pending`, `status: draft`. And it says "six milestones" (now seven).
**Impact:** The ROADMAP is defined as the truth-for-what's-done. An implementer trusting it would scaffold Drogon+vcpkg+jsoncpp/valijson/RE2 — the exact stack that was dropped — and believe they may start coding. Direct Truth-Triangle violation.
**Fix:** Rewrite the ROADMAP against the current plan: add an M0 section (T0.1–T0.8 + M0 review gate), strip every Drogon/vcpkg/valijson/jsoncpp/RE2/`setThreadNum` reference, set status to "planning / GATE 0 pending", correct the milestone count to seven. (~30 min.)

### P2. Plan says "six milestones" but there are seven — and the pending human sign-off inherits the wrong count
Plan line 22 ("across six milestones"), line 795 ("keep all six milestones"), and **open question #2** line 798 ("Scope: all six milestones this phase?"). Actual: M0+M1–M6 = **seven**. The extra one (M0, from-scratch core libs) is the largest and riskiest piece of work.
**Impact:** The human would be approving an understated scope by an entire milestone, at the exact gate that's pending.
**Fix:** Update the Goal, the resolved-scope note, and sign-off question #2 to "seven milestones (M0 core libraries + M1–M6)".

### P3. Two M0 tasks aren't self-contained — their acceptance criteria depend on M1 artifacts
- **T0.5** (`jsonschema`) verify says "accepts a valid **`Recipe`** and a valid **`Macros`** body" + `$ref` to `#/definitions/Macros` — but the Recipe/Macros schema is authored in **T1.2** (M1, later). A coder at T0.5 has no schema to test against.
- **T0.6/T0.7** verify against "a dev Postgres" / "logs show a successful DB connection" — but the compose Postgres service is first provisioned in **T1.1** (M1). M0 never stands up a DB.
**Impact:** Neither is implementable from the plan alone; the coder blocks or improvises.
**Fix:** Move the dev-`docker-compose` Postgres provisioning into **T0.1** (so all of M0 can talk to a DB); reword **T0.5**'s verify to use a **generic fixture schema** authored inline in M0 (types/enum/required/`$ref`), noting the real Recipe schema validates in T1.2.

### P4. Scrutiny inversion + process risk — freeze D1 and make the 5th external review a precondition
The stable recipe/CRUD logic has been reviewed ~16 passes; the genuinely novel, security-exposed M0 (hand-rolled HTTP server, OpenSSL client, JSON parser, validator, pool, multipart) has had **no external review** — yet the frontmatter reads "re-APPROVED … human signature is the only remaining step." GATE 0 has been reopened **three times post-approval**; a fourth foundational pivot would reset everything again.
**Fix / recommendation:** As conditions of signing — (1) treat the pending **5th external review as mandatory**, focused on M0; (2) get an explicit human commitment to **freeze D1** (no further stack pivots) so "ready to implement" stops being fragile.

---

## 🟠 Major — technical

### T1. Reproducibility is overclaimed — a base-image tag is mutable, apt versions drift
"Reproducibility rests on a pinned base-image tag + documented apt list" (D1/D2/T6.1/T6.2) is weaker than stated: a Docker **tag** is a mutable pointer (re-published with updates), and `apt-get install libpq-dev` (no version pin) pulls whatever the mirror serves at build time. The build is "roughly reproducible within a distro-release window," not reproducible. This is the concrete cost of dropping vcpkg's checked-in pins.
**Fix:** Pin the base image by **digest** (`FROM debian@sha256:…`), not tag — cheap, minimum honest hardening. Optionally pin exact apt versions against `snapshot.debian.org`. At minimum, reword T6.1/T6.2 to "digest-pinned base + documented apt list ≈ reproducible within a release window."

### T2. `T0.2` HTTP framing — request-smuggling / desync edge cases unaddressed
The plan handles caps, malformed→400, slowloris, but not the RFC 7230 §3.3.3 framing rules that cause CL.TE / TE.CL desync with nginx in front: reject **CL+TE both present**; reject **duplicate/conflicting `Content-Length`**; accept only `Transfer-Encoding: chunked` (else 400/501); **overflow-check** chunk-size hex and CL; reject **bare CR / bare LF / embedded NUL** in request line and headers. Practical exposure is *low* (single user, VPN, nginx normalizes, no shared cache), but it's a parser-correctness must-have and cheap now.
**Fix:** Add the above to T0.2's hardening list and add smuggling-shaped test vectors (dual CL+TE; oversized chunk-size) to its verify.

### T3. `T0.5` validator — "nullable" is not a JSON-Schema keyword (silent under-validation); plus two parser traps
D2/T0.5/T1.2 list `nullable` as a construct. Draft-7 has **no `nullable` keyword** (that's OpenAPI); nullability is `"type": ["string","null"]`. If the schema author writes `"nullable": true` and the hand-rolled validator ignores unknown keywords, it **silently under-validates** — the format has many nullable fields (`quantity`, `unit`, `group`, `foodId`, `sourceUrl`, `ownerId`, `description`, times). Also in T0.4/T0.5: (a) `\uXXXX` **surrogate pairs** (U+10000+) must be combined; (b) the recursive-descent parser needs a **recursion-depth cap** or a deeply nested body (OFF/LLM) can stack-overflow (DoS).
**Fix:** Express nullability as `type:[…,"null"]` in the authored schema and implement that union in the validator (drop "nullable" as a pseudo-keyword). Add a validator test: `null` rejected in a non-nullable field, accepted in a nullable one. Add surrogate-pair round-trip tests (T0.4) and a nesting-depth cap.

---

## 🟡 Minor

- **T5.2 `ILIKE` Ä↔ä depends on the (unpinned) Postgres locale.** `ILIKE` folds via the DB collation; under **C/POSIX** it does *not* fold non-ASCII. The official `postgres` image defaults to `en_US.utf8` so it usually works, but the dependency is unstated. **Fix:** initialize the container with a UTF-8 locale (`LANG`/`POSTGRES_INITDB_ARGS=--locale=…`) or set a UTF-8 `COLLATE` on the searched columns; note it in T5.2/T6.1.
- **T0.6 wrapper can't run multi-statement migration files.** `PQexecParams` (parameterized) runs a **single** command; migration files are multi-`;`. **Fix:** add a raw `PQexec` path documented as **migration-only, never for user input** (developer-authored SQL → no injection concern).
- **T5.1 multipart 8 MB cap must be enforced *during* streaming**, not after buffering the part (else a large upload OOMs before the check). **Fix:** state streaming enforcement, mirroring the T0.2 body cap.
- **Monorepo layout omits `/jsonschema`.** D1's `/lib` list is `/net /router /json /db /httpclient` but T0.5 builds `jsonschema` as its own module. **Fix:** add `/jsonschema` to the diagram.
- **C++ standard never pinned** ("C++17/20" throughout). Inconsistent with the exact Angular pin. **Fix:** pick one (recommend C++20) in T0.1.
- **Archive path divergence:** on disk `docs/archive/`; the workflow convention is `docs/archived/plans/`. **Fix:** pick one and align the M6 archive step + ROADMAP DoD.
- **Ingredient `note` render path not explicit in T2.1.** `group` renders (grouped display); per-row `note` (e.g. "(uncooked weight)") isn't named. **Fix:** one line in T2.1 so notes don't silently vanish on the detail page.
- **WSL2-dev vs Docker-deploy parity:** since the shipped artifact comes from the multi-stage Dockerfile, WSL parity only affects "works in dev, breaks in image" surprises. **Fix:** one sentence in T6.2 — the Docker build is the source of truth.
- **Model tags** (`qwen3.5-9b`, `qwen3.8-27b`, "Gemma 4 27B") match no confirmable registry ids — but already hedged as illustrative behind the T6.2 verify gate. No action.

---

## Checked and found sound (matters for the verdict)

- **Plan body is clean of old-stack leftovers.** Every Drogon/vcpkg/valijson/jsoncpp/libcurl/RE2 hit in the active body is deliberate contrast framing ("replaces jsoncpp", "no `re2`") or lives in the historical Reviewer-notes section. No active task assumes the framework. (The staleness is confined to the ROADMAP — P1.)
- **`macroSource`/`macrosEstimated` precedence is consistent.** A suspected contradiction between the format's "a later recompute re-derives both" and T1.3's "recompute EXCEPT when `macroSource=='manual'`" was checked and **refuted**: a "later recompute" is a user-triggered compute/estimate that flips the source *away* from manual first; a PUT that leaves source `manual` correctly honors the cleared flag. (Wording could be one clause clearer, but it's not a defect.)
- **No orphaned fields** — the 4th-review tags/favorite/notes/times gap is genuinely fixed end to end.
- **M0 verification is rigorous** — behavior-based checks (slowloris frees a worker; chunked cap without upfront CL; malformed→clear error not crash; SQL-metacharacter bind stored literally; invalid cert rejected). The strongest part of the plan.
- **Correctly specified:** parameterized libpq binds; dedicated non-pooled `connect()` for the migration advisory lock (T0.6→T1.1); SSRF closure (external URLs store-only); upload hardening (UUID filenames, magic-byte check, size cap); `ca-certificates` + runtime libs in the slim image; kJ→kcal `÷4.184`; `utf8proc` for NFC; Ollama native `/api/chat format=schema` as the more-reliable path; multipart deferral M0→T5.1 stated consistently.

---

## Recommendation

**Before signing GATE 0 and starting M0:**
1. Fix the two security blockers **S1** (TLS hostname verification) and **S2** (sanitizers + fuzzing) — fold both in as named T0.x tasks/verify items.
2. Fix the document/process majors: **P1** (rewrite the ROADMAP), **P2** (milestone count → seven), **P3** (make T0.5/T0.6/T0.7 self-contained), and adopt **P4** (5th external review on M0 as a precondition + freeze D1).
3. Fold the technical majors **T1–T3** (digest-pin, HTTP framing, `nullable`→`type:[…,"null"]`).
4. Minors can ride along in the same pass.

**On the from-scratch decision itself:** it is defensible *as a learning exercise* and the plan is honest about the trade — but the hand-rolled **HTTPS client is the most disproportionate item**: TLS is famous for silent, security-critical subtleties (S1 is exactly one), so it delivers poor learning-signal-per-risk. Worth asking whether the TLS client specifically is where "from scratch" earns its keep, or whether that one seam should use a vetted library while everything else stays hand-rolled.

> Note: this is a GitHub repo outside the weblab/SLA world, so the "findings → GitLab issue" pipeline does not apply here.
