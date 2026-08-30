# mise-en-place — Roadmap

Der Fortschritts-Blick auf das Projekt: eine Liste zum Abhaken. Die Spezifikation
ist der [Plan](plans/PLAN_recipe_app_foundation.md) — dort steht zu jeder Aufgabe
das vollständige `Verify`. Hier haken wir ab, was fertig und geprüft ist.

**Legende:** `[ ]` = offen · `[~]` = in Arbeit · `[x]` = fertig und geprüft.
Eine Aufgabe ist **fertig**, wenn ihr Code geschrieben ist, ihr „Fertig, wenn"
erfüllt ist und die Änderung committet ist. Ein Milestone schließt nach dem
Review an seiner Grenze.

**Stand:** GATE 0 ist bestanden — der Plan ist freigegeben (die Approval-Daten
stehen im Frontmatter des Plans). Nächster Schritt ist die Implementierung bei
Milestone 0, Aufgabe T0.1. Der Stack ist eingefroren (D1); die maßgebliche
Beschreibung steht im [Plan](plans/PLAN_recipe_app_foundation.md).

---

## Phase 0 — Planung (abgeschlossen)

- [x] Verstehen — Intent und Repo-Kontext
- [x] Plan geschrieben — [`plans/PLAN_recipe_app_foundation.md`](plans/PLAN_recipe_app_foundation.md)
- [x] Rezeptformat und Produkt-Scope festgelegt
- [x] Fünf externe Reviews eingearbeitet ([`reviews/`](reviews/)) — inkl. der
      From-scratch-Wende, dem vcpkg-Verzicht und dem M0-Security-Durchgang
- [x] Reviewer-Freigabe (siehe Plan-Frontmatter)
- [x] Menschliche GATE-0-Freigabe (siehe Plan-Frontmatter) — auth-ready ohne
      Auth, sieben Milestones, D1 eingefroren, TLS von Hand über OpenSSL plus
      der S1-Hostname-Fix

---

## Milestone 0 — Core-Backend-Bibliotheken (from scratch)

- [ ] **T0.1 — Toolchain + Skeleton** — GNU Makefile (C++20), das die apt-Libs
      `libpq-dev`/`libssl-dev`/`libutf8proc-dev`/`catch2` über `pkg-config`
      findet (Linux-only Build → Make statt CMake); Layout `/lib`
      (`net router json jsonschema db httpclient`) + `/app` + `/tests`;
      ASAN+UBSAN-Build (`make asan`, TSan für die Pools aus T0.9 und T0.6); dev
      `docker-compose` Postgres. WSL2.
      **Fertig, wenn:** `make` baut; ein Catch2-Test läuft grün unter
      ASAN+UBSAN; die dev-Postgres kommt hoch.
- [ ] **T0.2 — `net` (HTTP/1.1-Server, single-threaded)** — Socket-Listener,
      Request-Parser (`Content-Length` und chunked), Response-Writer, keep-alive
      (single-threaded — der thread pool folgt in T0.9); Härtung: Size-Caps,
      Timeout (Anti-Slowloris), Anti-Smuggling-Framing (CL+TE, doppeltes CL,
      bare CR/LF/NUL, Overflow); Fuzz-Harness.
      **Fertig, wenn:** malformte und Smuggling-Vektoren → 400; übergroß oder
      chunked über Cap wird abgewiesen statt OOM; eine stehende Verbindung läuft
      in den Timeout; der Fuzzer bleibt sauber über ein Budget.
- [ ] **T0.3 — `router`** — Method + Path (`:id`-Params) → Handler; 404/405;
      Error- und Warning-Envelope-Helfer.
      **Fertig, wenn:** Routing-, Param-, 404- und 405-Tests plus ein
      envelope-förmiger Error-Body bestehen.
- [ ] **T0.4 — `json`** — eigener Parser + Serializer (Unicode-Escapes inkl.
      Surrogate-Pairs, Zahlen, bool/null), Rekursionstiefen-Cap; Fuzz-Harness.
      **Fertig, wenn:** Round-Trips (inkl. Surrogate-Pairs) bestehen; malformt →
      sauberer Fehler; Tiefe über dem Cap wird abgewiesen; der Fuzzer bleibt
      sauber.
- [ ] **T0.5 — `jsonschema`** — eigener Validator für die authored subset
      (types/enum/required/arrays/`type:[…,"null"]`-Union/`$ref`), meldet den
      fehlerhaften Pfad; getestet gegen ein generisches Fixture-Schema (das echte
      Recipe-Schema kommt in T1.2).
      **Fertig, wenn:** valides Fixture akzeptiert; jede Verletzungsklasse (plus
      null in einem non-nullable Feld) mit Pfad abgewiesen; null in einem
      nullable Feld akzeptiert.
- [ ] **T0.6 — `db` (libpq)** — Connection-Pool, parametrisiertes `exec`,
      Result→Row-Mapping, Transaktions-Helfer, standalone `connect()` für den
      Migration-Lock, ein migration-only `PQexec`-Pfad.
      **Fertig, wenn:** parametrisierter Round-Trip; ein SQL-Metazeichen als bind
      wird literal gespeichert; Pool-Nebenläufigkeit; standalone connect
      funktioniert.
- [ ] **T0.7 — App-Wiring + `/health`** — `main()` startet den Server, mountet
      den Router, baut den db-Pool aus env, Logging + Config.
      **Fertig, wenn:** die App bootet; `GET /health` → 200 durch den eigenen
      Server; der DB-Connect wird geloggt.
- [ ] **T0.8 — `httpclient`** — eigener HTTP/1.1-Client; chunked decode;
      Body-Cap; HTTPS über OpenSSL mit HOSTNAME-Verifikation (`SSL_set1_host` +
      `X509_V_OK`) für OFF, plain HTTP für Ollama; Timeouts.
      **Fertig, wenn (echtes Netz):** HTTPS-GET verifiziert; valider CA bei
      falschem Hostname wird abgewiesen; plain-HTTP-GET lokal; chunked decode;
      der Fake bleibt das Unit-Double.
- [ ] **T0.9 — `net`-Nebenläufigkeit (thread pool, zuletzt gebaut)** — fester
      Worker-Pool; der Accept-Loop verteilt jede Verbindung an einen Worker, der
      pro Request eine `db`-Connection (T0.6) besitzt; beschränkte Queue;
      sauberes drain/join-Shutdown; unter TSan gebaut und getestet.
      **Fertig, wenn:** nebenläufige keep-alive-Clients ohne verschränkte oder
      korrupte Antworten; TSan sauber unter Last; die Queue ist beschränkt;
      sauberes Shutdown, ASAN sauber.
- [ ] **M0-Review** — der Review an der Milestone-Grenze besteht (der größte und
      am stärksten sicherheitsexponierte Teil — sorgfältig lesen).

---

## Milestone 1 — Schema, DB, Browse-API (auf den M0-Libs)

- [ ] **T1.1 — Migration-Runner** — `schema_migrations`, jede Migration in einer
      eigenen Transaktion, Session-Advisory-Lock auf der dedizierten Connection.
      **Fertig, wenn:** wendet beim Boot an; kein erneutes Anwenden; eine
      fehlschlagende Migration rollt sauber zurück.
- [ ] **T1.2 — `Recipe`-Schema + Migrationen + Model-Round-Trip** — das authored
      Schema (+ `definitions/Macros`); relationale Tabellen (Kind-Tabellen +
      `recipe_tags`, `ON DELETE CASCADE`, `macros_estimated`-Spalte); eigene
      JSON-Zusammensetzung + Validator.
      **Fertig, wenn:** Migrationen wenden an; der Round-Trip deckt
      gruppierte/to-taste/geordnete-und-leere Steps, Tags- und Bild-Reihenfolge
      und `macrosEstimated` ab; der Validator weist Invalides ab.
- [ ] **T1.3 — Repository/Service (CRUD) + PUT-Full-Replace** — food_id-Picks
      überleben; `macrosEstimated` wird neu berechnet, außer bei
      `macroSource=manual`; Reihenfolge der Bild-Datei-GC.
      **Fertig, wenn:** CRUD + PUT-erhält-Picks + Per-100g-Null-Pfad-Tests
      bestehen.
- [ ] **T1.4 — Browse-Controller + Seed** — `GET /api/recipes` (paginierte
      Summary) + `/:id` (voll); Seed mit `food_id` null.
      **Fertig, wenn:** die Liste respektiert limit/offset; `/:id` liefert ein
      schema-valides volles Recipe.
- [ ] **M1-Review**

---

## Milestone 2 — Frontend-Scaffold, Browse/Detail, strukturiertes Formular

- [ ] **T2.0 — UI-Mockups** ([`mockups/`](mockups/)) — Screens + States sind vor
      dem Bau abgestimmt.
- [ ] **T2.1 — Angular-Scaffold + Browse/Detail** — exakt gepinntes Angular;
      Dev-Proxy; typisierter API-Service auf dem Error-Envelope; Browse
      (paginierte Summaries) + Detail (Makros, `—`/Estimated-Marker,
      `macroSource`-Badge, Tags/Favorit/Notizen/Zeiten, Zutaten-Notizen).
      **Fertig, wenn:** `ng build`/`ng test` grün; Liste und Detail rendern ein
      geseedetes Rezept.
- [ ] **T2.2 — Add/Edit-Formular + DELETE** — Reactive Forms inkl.
      Tags/Favorit/Notizen/Zeiten (der einzige Weg, sie zu setzen); `POST`/
      `PUT /api/recipes/:id`; `DELETE …/:id` → 204.
      **Fertig, wenn:** Speichern persistiert; Tags und Favorit machen den
      Round-Trip und werden von den T5.2-Filtern gefunden; 422 bei Invalidem;
      PUT erhält Picks; Delete kaskadiert.
- [ ] **T2.3 — Manuelle Makro-Eingabe + Override.**
      **Fertig, wenn:** manuelle Makros persistieren und rendern; per-100g „—"
      ohne Gewicht.
- [ ] **M2-Review** — M1 + M2 ergeben die erste End-to-End-Scheibe.

---

## Milestone 3 — Paste-Import mit wählbarer Engine

- [ ] **T3.1 — `RuleBasedParser` + `POST /api/parse`** — Social-Captions;
      utf8proc-NFC, dann eigenes UTF-8-Scanning; Sections/Menge-Einheit/
      Makro-Block/to-taste/Klammerausdruck→Notiz; `/api/parse` liefert einen
      partiellen Draft, nie 422.
      **Fertig, wenn:** 3 Fixtures parsen (Gruppierung, null-Menge, Makro-Block,
      leere Steps, Umlaut/NFD, die `7%`-Falle); eine partielle Caption → Draft +
      Warnung.
- [ ] **T3.2 — `LlmClient` + `LlmParser` (Fallback)** — auf `IHttpClient` (T0.8);
      Ollama nativ `/api/chat format=schema`; invalid → leerer/partieller Draft +
      Warnung.
      **Fertig, wenn:** Fake-Client-Test: valid akzeptiert, malformt → der
      dokumentierte Fallback.
- [ ] **T3.3 — Paste-Screen** — Textarea + Engine-Toggle → editierbare Preview →
      speichern.
      **Fertig, wenn:** ein regelbasiertes Paste ergibt ein vorbefülltes,
      speicherbares Formular.
- [ ] **M3-Review**

---

## Milestone 4 — Makros aus Open Food Facts + LLM-Schätzung

- [ ] **T4.1 — `NutritionSource` + OFF-Client + `foods`-Cache + Converter** —
      OFF v2 `/api/v2/search` (Limits prüfen) auf T0.8-HTTPS; `foods` (UUID-id
      PK, code UNIQUE) + FK-Ergänzung; local-first; per-100g-Mapping; Einheit→
      Gramm (Dichte- und Stück/Löffel-Tabellen).
      **Fertig, wenn:** Fake-Client-Kandidaten; Cache-Hit → kein Call; nur-kJ
      umgerechnet/abgewiesen; Einheiten- plus Stück/Löffel-Umrechnung inkl. des
      geflaggten Pfads.
- [ ] **T4.2 — Makro-Engine** — Summe → pro Portion + pro 100g; ungepickt/
      geflaggt → „—"; setzt `macrosEstimated`, wenn Stück/Löffel eingeflossen
      ist.
      **Fertig, wenn:** exakte Gramm treffen die handgerechnete Summe
      (`estimated:false`); Stück/Löffel volles Gewicht (`estimated:true`);
      geflaggt → „—".
- [ ] **T4.2b — `POST /api/macros/compute` + Suche-und-Auswahl-UI** — setzt
      `macroSource:"ingredients"`.
      **Fertig, wenn:** im Formular Suche → Auswahl → Füllen; Compute korrekt +
      Provenienz gesetzt.
- [ ] **T4.3 — `POST /api/macros/estimate` + LLM-Button** — setzt
      `macroSource:"llm"` + `macrosEstimated:true`.
      **Fertig, wenn:** Fake-LLM füllt + prüft die Provenienz; der Nutzer kann
      überschreiben.
- [ ] **M4-Review**

---

## Milestone 5 — Medien, Politur, Suche

- [ ] **T5.1 — Bild-Upload + Lifecycle** — ergänzt einen Multipart-Parser in
      `net`; server-generierter Dateiname, ≤ 8 MB während des Streamings
      erzwungen, Typ per Magic-Bytes; externe URL nur speichern (kein SSRF);
      GC eigener Dateien bei Delete/PUT.
      **Fertig, wenn:** valider Upload rendert; übergroß/falscher-Typ/Traversal
      abgewiesen; extern wird nie gefetcht; Delete/PUT entfernt eigene Dateien.
- [ ] **T5.2 — Suche / Filter / Sort** — case-insensitives `ILIKE` (UTF-8-Locale,
      keine Accent-Faltung, keine Extension); Tag-AND-Filter; Favoriten;
      minProtein/maxCalories; Sort; auf der geklammerten paginierten Liste.
      **Fertig, wenn:** Teilmengen-Tests (inkl. `Ä`↔`ä`, Kontrolle dass NICHT
      accent-gefaltet wird), Sorts und Limit-Clamp bestehen.
- [ ] **M5-Review**

---

## Milestone 6 — Deployment & Docs

- [ ] **T6.1 — Docker + nginx + Compose** — mehrstufiges Dockerfile auf einer
      per-Digest gepinnten Base; die Build-Stufe apt-installiert die Deps; das
      schlanke Runtime liefert `libpq5`/`libssl`/`libutf8proc`/`ca-certificates`;
      Postgres-Init mit UTF-8-Locale; nginx `/api` + `/uploads` + `/health`;
      geteiltes `uploads`-Volume über Backend und nginx.
      **Fertig, wenn:** `compose up` liefert aus; Browse persistiert; `/api/*`,
      `/uploads/*`, `/health` erreichbar; ein vom Backend geschriebenes Bild wird
      von nginx ausgeliefert; OFF-Suche im Container funktioniert (CA-Trust).
- [ ] **T6.2 — Docs** — Env-Vars, Ollama-Pointer (Pull-Tag prüfen), Backup,
      exakte apt-Liste + Digest, minimaler Build-RAM, „der Docker-Build ist die
      Wahrheit".
      **Fertig, wenn:** die copy-paste-bare Sequenz funktioniert: `compose up` →
      `curl …/health` 200 → ein Rezept POSTen → erscheint in der Liste.
- [ ] **M6-Review**

---

## Definition of Done (ganze Phase)

- [ ] Alle sieben Milestones fertig (M0 + M1–M6), jeder mit bestandenem
      Grenz-Review
- [ ] `docker compose up` fährt die komplette App auf einem Home-Server oder VPS
      hinter VPN oder Basic-Auth über TLS
- [ ] Die Docs führen ein frisches Setup von Grund auf zu einer laufenden App
- [ ] Plan + Reviews sind nach `docs/archive/` archiviert (gemäß
      [`archive/README.md`](archive/README.md))
