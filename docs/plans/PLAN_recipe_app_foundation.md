---
plan: recipe_app_foundation
status: approved
approvals:
  reviewer: 2026-08-24   # APPROVED after 5 external reviews + from-scratch pivot + vcpkg drop.
  human: 2026-08-24    # GATE 0 human approval — auth-ready-no-auth; SEVEN milestones (M0+M1-M6); D1 FROZEN (no more stack pivots); TLS stays hand-rolled over OpenSSL + the S1 hostname-verify fix; TLS/basic-auth caveat accepted.
---

# Plan — mise-en-place recipe app (foundation)

## Ziel
Eine selbst-gehostete Web-App für eine Person, um Rezepte zu speichern, zu
durchstöbern und anzulegen. Rezepte kommen über ein strukturiertes Formular
herein oder durch das Einfügen von Freitext (TikTok-/Instagram-Caption), der in
ein einheitliches Rezeptformat normalisiert wird. (Eingefügter Website-Text und
URL/HTML-Import sind eine spätere Phase — siehe D4.) Rezepte zeigen optionale
Bilder, einen optionalen Quell-Link und Makros pro Portion und pro 100 g. Die
Makros werden aus einer Nährwert-Datenbank aus den Zutaten berechnet, alternativ
per LLM geschätzt (Button). Paste-Parsing und Makro-Schätzung bieten je eine
wählbare Engine: regelbasiert oder ein lokales LLM (HTTP-API, z. B. Ollama).

Dieser Plan deckt das Produkt-Fundament ab — alle Kernfeatures end-to-end — über
sieben Milestones (M0 Core-Bibliotheken + M1–M6). M0 kam mit der
From-scratch-Wende dazu (D1). Es ist die erste Workflow-Phase; spätere Phasen
(erweiterte Suche, Tagging, Meal-Planning, Authentifizierung und Website-Text-/
URL-Import — siehe D4) liegen hier außerhalb des Scopes. Das Schema wird aber
auth-ready gebaut, sodass Auth eine spätere Ergänzung ist, keine Migration.

## Festgelegte Design-Entscheidungen
> Diese Entscheidungen wurden mit dem Menschen bei GATE 0 vereinbart. Ein
> C++-Backend wurde bewusst gewählt, um C++ zu lernen; und (überarbeitet
> 2026-08-24) der Mensch entschied, es from scratch zu bauen, ohne Web-Framework
> — weil der Bau der Grundlagen (ein HTTP-Server, Router, JSON, eine DB-Schicht
> über libpq, ein HTTP-Client) der Punkt ist. Das ist ein bewusster Tausch: viel
> zusätzliche Plumbing-Arbeit gegen dieses Lernen. Der Plan stellt deshalb einen
> Core-Bibliotheken-Milestone (M0) an den Anfang, damit spätere Milestones auf
> eigenen Libs stehen.

### D1 — Architektur: getrenntes Frontend + C++-Backend from scratch (Monorepo)
> Überarbeitet 2026-08-24 — der Mensch entschied, das Backend „von den Sockets
> aufwärts" zu bauen statt ein Web-Framework zu nutzen, weil der Bau der
> Grundlagen der Punkt des Lernziels C++ ist. Das tauscht viel Vorab-Plumbing
> gegen dieses Lernen, bewusst akzeptiert. Deploy-Ziel ist Linux hinter einem VPN
> (D6), was die Sicherheits-Exposition eines selbst gebauten HTTP-Servers
> eingrenzt. Entwickelt wird in WSL2 (Ubuntu), auf derselben OS-Familie wie das
> Deploy.

- **Backend:** C++17/20, kein Web-Framework — die App baut ihre eigenen kleinen
  Bibliotheken:
  - **HTTP-Server** — ein TCP-Listener über OS-Sockets, ein
    HTTP/1.1-Request-Parser (Request-Line, Header, Body, `Content-Length`/chunked),
    keep-alive. Der Server startet single-threaded (eine Verbindung zur Zeit); ein
    thread pool für Nebenläufigkeit kommt zuletzt in M0 (T0.9), sobald alles andere
    läuft — bewusst aufgeschoben, damit das Concurrency-Modell nach den Grundlagen
    gebaut wird, nicht neben ihnen. (multipart/form-data-Parsing kommt in `net`,
    sobald es zuerst gebraucht wird, bei M5/T5.1 — nicht Teil des M0-Server-Scopes.)
  - **Router** — Method + Path (mit `:id`-Params) → Handler; eine
    Request/Response-Abstraktion.
  - **JSON** — eigener Parser + Serializer (ersetzt jsoncpp) und eigene
    JSON-Schema-Validierung, so weit das `Recipe`/`Macros`-Schema es braucht
    (ersetzt valijson).
  - **DB-Schicht** — ein dünner C++-Wrapper über libpq (Connect aus einem Pool,
    parametrisiertes `exec`, Mapping der Ergebnisse auf Structs). libpq spricht das
    Postgres-Wire-Protokoll; das bauen wir NICHT nach — libpq zu nutzen heißt,
    Postgres' eigenen Client zu nutzen, kein Framework.
  - **HTTP-Client** — eigener Client über Sockets + OpenSSL für HTTPS (zu OFF und
    dem LLM; ersetzt libcurl), hinter dem `IHttpClient`-Seam (D3).
  - **Unvermeidbare Externals** (über System-Pakete / `apt` — kein
    Package-Manager): libpq (PG-Protokoll), OpenSSL (TLS), utf8proc
    (Unicode-NFC-Normalisierung für den Parser — volles NFC von Hand bräuchte
    Unicode-Tabellen, also nehmen wir diese eine kleine Lib; 4th-review Major) und
    ein Test-Framework (Catch2). Alles andere ist unseres. Kein vcpkg — der Mensch
    wählte System-Pakete und tauscht bewusst vcpkgs reproduzierbares, eingechecktes
    Version-Pinning gegen einen einfacheren Build ohne Dependency-Manager. Das ist
    nur „reproduzierbar innerhalb eines Distro-Release-Fensters", nicht exakt
    reproduzierbar (5th-review T1): ein Docker-*Tag* ist ein veränderlicher Zeiger,
    und `apt install` (ungepinnt) zieht, was der Mirror gerade liefert. Minimale
    ehrliche Härtung: die Base per Digest pinnen (`FROM debian@sha256:…`), nicht
    per Tag (T6.1) + eine dokumentierte `apt`-Liste (T6.2); optional apt-Versionen
    gegen `snapshot.debian.org` pinnen. Ein einfaches GNU-Makefile findet die Libs
    über `pkg-config` (`pkg-config --cflags --libs libpq openssl libutf8proc`);
    siehe die T0.1-Notiz, warum Make statt CMake.
- **Frontend:** Angular-SPA (Angular CLI, `ng build` → statisches Bundle), als
  statische Dateien ausgeliefert; ruft das Backend über HTTP unter `/api/*` auf.
  Gewählt, um ein robustes, strukturiertes Framework zu lernen; seine Reactive
  Forms passen besonders gut zum dynamischen Zutaten-Zeilen-Formular und zur
  editierbaren Paste-Preview. Eine exakte Angular-Version vorab in `package.json`
  pinnen (ein konkretes `17.x.y`, nicht „v17+") — eine Untergrenze wie `^17` lässt
  einen frischen Build ein neueres Major mit anderen Builder-/Test-Defaults ziehen,
  genau den Drift, den ein Pin verhindert.
- **Same-origin-Strategie (kein CORS):** in dev proxyt `ng serve` (:4200) `/api` →
  Backend über `frontend/proxy.conf.json` (`ng serve --proxy-config`); in prod
  liefert nginx das statische Bundle und reverse-proxyt `/api/*` ans Backend (siehe
  T6.1). Same-origin, also braucht der Browser kein CORS. Falls später ein
  Cross-Origin-Pfad dazukommt, ergänzt die eigene HTTP-Schicht die
  CORS-Response-Header.
- **Monorepo-Layout:**
  ```
  /backend    C++-API from scratch (GNU-Makefile; libpq/OpenSSL/utf8proc/Catch2 über System-apt-Pakete)
    /lib          eigene Bibliotheken: /net (Sockets+HTTP-Server) /router /json /jsonschema /db /httpclient
    /app          /controllers  /models  /services  /migrations
    /tests
  /frontend   Angular-SPA
  /docs
  docker-compose.yml
  ```
- **API-Konventionen (2nd-review Major 4 + 5 — einmal festgelegt, überall
  genutzt):**
  - **Error-Envelope:** jede Nicht-2xx-Antwort hat eine JSON-Form —
    `{ "error": { "code": "<machine_slug>", "message": "<human text>", "details": <any?> } }`
    — damit der typisierte Frontend-Service (T2.1) genau eine Sache hat, gegen die
    er codet.
  - **Warning-Envelope (Erfolgspfad — 3rd-review minor #1):** ein `200` mit einer
    weichen Warnung nutzt **eine** Form — `{ "data": <payload>, "warning": { "code":
    "<slug>", "message": "<text>" } }` — für alle drei Degradations-Flows
    (`/api/foods/search` nur-Cache; der Parse-LLM-invalid-Draft, T3.2; der
    LLM-Makro-Fallback, T4.3). Fehlt `warning`, ist das Ergebnis sauber.
  - **Status-Codes:** `400` fehlerhafter Request; `422` schema-invalider
    `Recipe`-/Makro-Body (Fehler unseres eigenen Validators, M0 `jsonschema` — mit
    den fehlerhaften Pfaden in `details`); `404` unbekannte `:id`; `504`
    Upstream-Timeout. **OFF-Throttling-Verhalten für `/api/foods/search` (3rd-review
    minor #2):** existiert ein lokaler Kandidat → `200` + Cache-Ergebnisse +
    `warning`; sonst den Upstream-Zustand durchreichen — OFF gab `429` → `429`; OFF
    nicht erreichbar → `502`; OFF-Timeout → `504`. LLM nicht erreichbar / Timeout auf
    `/api/parse` und `/api/macros/estimate` ist KEIN 5xx: es degradiert auf den
    `200` + Warnung-Draft-Pfad (der Best-Effort-/leere editierbare Draft aus
    T3.2/T4.3), sodass ein ausgefallenes LLM die Eingabe nie blockiert (konsistent
    mit D6) — der T3.2/T4.3-„invalid output"-Fallback und ein Transport-Fehler
    teilen sich diesen Pfad. Mutationen nutzen den Error-Envelope: `PUT`/`DELETE`
    zielen auf `/api/recipes/:id` (damit die `404`-auf-`:id`-Regel gilt), und ein
    erfolgreiches `DELETE` gibt `204` zurück.
  - **Liste vs. Detail + Pagination:** `GET /api/recipes` liefert eine
    leichtgewichtige Summary-Projektion (id, title, erstes Bild, `favorite`, Makros
    pro Portion + `macrosEstimated`, damit die Liste geschätzte Makros ebenfalls
    markieren kann, tags) — NICHT die voll aus Kind-Tabellen zusammengesetzten
    Objekte — und ist paginiert über `?limit=&offset=`; `limit` ist standardmäßig 50
    und wird auf ein hartes serverseitiges Maximum (z. B. 100) geklammert, sodass ein
    großes Client-`limit` die Grenze nicht aushebeln kann (3rd-review minor #5). Das
    volle kanonische `Recipe` (alle Kind-Tabellen zusammengesetzt) gibt es nur unter
    `GET /api/recipes/:id`. Das begrenzt die Browse-Query, wenn der Bestand wächst.
- **Akzeptierter Trade-off:** zwei Build-Systeme und zwei Deploys; der
  `Recipe`-Typ wird über C++/TS NICHT automatisch geteilt — drei Repräsentationen
  (JSON-Schema, C++-Struct, TS-Typ) werden in dieser Phase von Hand synchron
  gehalten, ein bekanntes Drift-Risiko (Q9). Gemildert durch D2 (ein JSON-Schema als
  Single Source of Truth in `docs/`, auf beiden Seiten validiert) und den
  T1.2-Round-Trip-Test; ein Codegen-Schritt (z. B. `json-schema-to-typescript` für
  den TS-Typ) ist eine saubere spätere Ergänzung, falls der Drift wehtut.

### D2 — Storage: PostgreSQL (eigener libpq-Wrapper), mit JSON als Austauschformat
- **PostgreSQL** über unsere eigene DB-Schicht auf libpq (D1). Gewählt statt
  SQLite, weil der Mensch später Auth / Multi-User ergänzen will, wofür Postgres die
  robustere Basis ist; der zusätzliche Container ist unter Docker Compose günstig.
  Die DB-Schicht bietet eine kleine synchrone API — Connect (aus unserem eigenen
  Connection-Pool), parametrisiertes `exec` (`$1,$2…`-Binds, nie string-konkatenierte
  SQL → keine Injection) und Result→Struct-Mapping — synchron gehalten, damit es beim
  Lernen zugänglich bleibt.
- **Threading (eigener thread pool) — zuletzt in M0 gebaut (T0.9):** der Server
  läuft bis T0.8 single-threaded (accept → handle → respond, eine Verbindung zur
  Zeit), was für eine Person genügt und die ganze M0-Scheibe zum Laufen bringt, bevor
  Nebenläufigkeit dazukommt. T0.9 fügt dann den thread pool hinzu: der HTTP-Server
  (D1) verteilt jeden Request an einen Worker-Thread aus unserem Pool; ein Worker
  besitzt für den Request eine libpq-Connection, und die blockierenden `exec`-/
  HTTPS-Aufrufe laufen auf diesem Worker, nie auf dem Accept-Loop. Für eine Person
  reicht ein kleiner Pool (z. B. 4–8 Worker); wir behaupten KEIN non-blocking I/O —
  einen Worker zu blockieren ist in dieser Größenordnung in Ordnung. (Genau hier
  lehrt das Selberbauen das Concurrency-Modell, das ein Framework versteckt hätte —
  deshalb kommt es zuletzt, wenn Sockets/Parsing/Routing/DB schon laufen, nicht
  verstrickt mit den Basics.)
- **Dependencies über System-Pakete (kein vcpkg):** die einzigen Externals sind
  libpq, OpenSSL, utf8proc und Catch2, installiert mit `apt` (`libpq-dev`,
  `libssl-dev`, `libutf8proc-dev`, `catch2`/`libcatch2-dev`) in dev (WSL) und in der
  Docker-Build-Stufe. Das Makefile findet sie mit `pkg-config`. Kein Framework, kein
  Dependency-Manager. (Reproduzierbarkeit ruht auf gepinnten Distro-/
  Base-Image-Versionen gemäß D1; der Build kompiliert KEINES davon aus dem Quellcode,
  also ist er schnell und leicht — Q8.)
- **JSON + Schema-Validierung sind UNSERE (D1):** eigener Parser/Serializer und
  eigene Validierung des `Recipe`/`Macros`-Schemas. Das Schema in `docs/` bleibt die
  Single Source of Truth, aber da wir es selbst validieren, sind wir nicht an den
  unterstützten Draft einer Library gebunden — wir schreiben es zu einer klaren, in
  sich konsistenten Teilmenge: object/array/string/number/enum/required, und
  Nullability ausgedrückt als `"type":[…,"null"]`-Union — NICHT als
  `nullable`-Keyword (das ist OpenAPI, nicht JSON-Schema; ein unbekanntes Keyword
  würde die vielen nullable-Felder still unter-validieren — 5th-review T3) +
  `definitions/Macros` über `$ref`. Unser Validator implementiert genau diese
  Teilmenge. Das trägt jeden „schema-validated"-Schritt (T2.x, T3.2, T4.3). (Löst die
  alte valijson-/Draft-7-Beschränkung ab.)
- **Migrationen:** einfache SQL-Dateien in `/backend/migrations`, beim Start von
  einem kleinen versions-verfolgenden Runner angewendet, der die angewendeten
  Versionen in einer `schema_migrations`-Tabelle festhält. Jede Migration läuft in
  ihrer eigenen Transaktion, und ihre Version wird nur bei Erfolg festgehalten (ein
  Fehlschlag rollt nur diese Migration zurück, nicht die schon committeten).
  **Sicherheit bei nebenläufigem Boot (Q7 + 2nd-review Major 3 — für spätere
  Skalierung behalten, vom Single-Container-Deploy nicht gebraucht):** zwei
  gleichzeitig bootende Backend-Instanzen sind im heutigen D6-Single-Container-Compose
  unmöglich; diese Mechanik wird bewusst behalten, damit ein späteres
  Multi-Instanz-Deploy sicher ist, kein stilles Gold-Plating (4th-review meta). Der
  Runner muss seinen Lock halten und all seine Arbeit auf einer dedizierten
  libpq-Connection erledigen (nicht auf einer aus unserem Request-Pool) — ein
  `pg_advisory_lock` von einer gepoolten Connection könnte den Lock, die Migrationen
  und den Unlock auf verschiedenen Connections landen lassen und so nicht
  serialisieren. Konkret: einen session-level `pg_advisory_lock` auf dieser einen
  dedizierten Connection nehmen, über alle Per-Migration-Transaktionen halten, am Ende
  freigeben. (Eine einzelne umschließende Transaktion wird NICHT genutzt — die gäbe
  ein Alles-oder-nichts-Rollback über alle Migrationen, eine andere Granularität; und
  ein Per-Migration-`pg_advisory_xact_lock` wird vermieden, weil er bei jedem Commit
  einer Migration freigibt und das Race zwischen den Migrationen wieder öffnet.) Der
  „schon angewendet?"-Check liest `schema_migrations`, während der Lock gehalten wird,
  sodass zwei bootende Instanzen serialisieren. („Idempotent" = sicher, den Runner via
  Versions-Tracking erneut auszuführen; die DDL selbst muss nicht idempotent sein.)
  Keine ORM-Codegen-Magie.
- **Relationale Speicherform (B1 — entschieden, keine JSONB-Blobs):**
  - `recipes` — eine Zeile pro Rezept: Skalar-Felder (`title`, `description`,
    `source_url`, `servings`, `prep_time_min`, `cook_time_min`, `total_weight_g`,
    `favorite`, `notes`, `schema_version`, `owner_id`, `created_at`, `updated_at`) +
    die Makro-Spalten pro Portion (`cal`, `protein`, `carbs`, `fat`) + `macro_source`
    + `macros_estimated` (boolean; hält das `macrosEstimated`-Flag fest, damit der
    „estimated"-Marker ein save→reload überlebt — ein gespeichertes Feld, nicht
    abgeleitet wie per-100 g).
  - `recipe_ingredients` — Kind-Tabelle, FK `recipe_id`, ein expliziter
    `position`-Integer für die Reihenfolge, und Spalten `group_label` (nullable
    Section-Label — die Spalte heißt `group_label`, NICHT `group`, ein
    Postgres-reserviertes Wort; das JSON-Feld bleibt `group`), `name`, `quantity`
    (nullable), `unit` (nullable), `food_id` (nullable UUID; der FK → `foods.id` wird
    später in T4.1 ergänzt, wenn die `foods`-Tabelle existiert — die Spalte entsteht
    hier in M1 OHNE das Constraint), `note` (nullable). Hier leben die
    nullable-quantity-/to-taste-Zeilen und die Gruppierung.
  - `recipe_steps` — Kind-Tabelle, FK `recipe_id`, `position`, `text`. (Geordnet;
    kann für video-only-Captions leer sein.)
  - `recipe_images` — Kind-Tabelle, FK `recipe_id`, `position`, `url`.
  - `recipe_tags` — Join-Tabelle, FK `recipe_id`, `tag`, plus ein `position`, damit
    die Reihenfolge des `tags[]`-Arrays round-trippt (unique auf `(recipe_id, tag)`).
    Der T5.2-Tag-AND-Filter ist `... WHERE tag = ANY($tags) GROUP BY recipe_id HAVING
    count(*) = $n`.
  - Alle Kind-/Join-Tabellen tragen `FK recipe_id … ON DELETE CASCADE`, sodass
    `DELETE /api/recipes/:id` (T2.2) das ganze Aggregat sauber entfernt.
  - Begründung: geordnete/abgefragte Collections sind echte Zeilen (saubere
    Reihenfolge, der Tag-Filter und die Makro-Range-SQL in T5.2, und späteres FTS
    funktionieren alle), kein opakes JSONB. Das kanonische `Recipe`-JSON wird an der
    API-Grenze aus diesen Tabellen zusammengesetzt (D2s Austausch-Schicht); JSON ist
    das Wire-Format, diese Tabellen sind die Persistenz.
- **Auth-ready Schema (keine Auth in dieser Phase implementiert):** ein
  `users`-Tabellen-Stub und ein nullable `owner_id`-FK auf `recipes`. Noch erzwingt es
  nichts; es existiert, damit Auth eine spätere Ergänzung ist, keine Schema-Migration
  von Live-Daten.
- Eine `foods`-Tabelle (in T4.1, M4 erstellt) cacht die
  Open-Food-Facts-Einträge, die der Nutzer gewählt hat. Sie hat einen
  Surrogat-UUID-`id`-Primary-Key (auf den sich `foodId: uuid?` im `Recipe`-Format
  bezieht) mit dem OFF-Barcode `code` als `UNIQUE`-Spalte (nicht als PK) und speichert
  Name, Makros pro 100 g (kcal/protein/carbs/fat), `lang` und `fetched_at`, sodass
  wiederholte Lookups kein Netz brauchen (D5). `recipe_ingredients.food_id`
  referenziert `foods.id`; eine Cache-Hit-Auflösung kann über `id` oder den unique
  `code` nachschlagen.
- Das einheitliche Rezeptformat (`Recipe`) ist einmal als JSON-Schema in `docs/`
  definiert (die Single Source of Truth), gespiegelt von einem C++-Struct (unsere
  eigene JSON-(De)Serialisierung) und einem TS-Typ. Es wird (von unserem eigenen
  Validator) an jeder API-Grenze validiert, auf der Ausgabe des Paste-Parsers, auf der
  LLM-Ausgabe, und für den Export genutzt. So gilt „alles wird einheitliches JSON" auf
  der API-/Austausch-Schicht; Postgres ist das Persistenz-Detail. Das Format trägt eine
  `schemaVersion` — unter relationaler Speicherung verdient sie sich ihren Platz vor
  allem auf der Export-/Austausch-Schicht (eine Änderung der Speicherform ist ohnehin
  eine SQL-Migration), also wird sie behalten, aber nicht überverkauft (Q6). (Eine
  YAML-Import-/Export-Bequemlichkeit ließe sich später ergänzen; JSON bleibt
  kanonisch.)

### D3 — Lokale-LLM-Integration (steckbar, hinter einem HTTP-Client-Seam)
- **HTTP-Client-Seam (B2 — Testbarkeit):** ausgehendes HTTP läuft durch ein
  winziges `IHttpClient`-Interface (`get`/`post` → Status + Body), mit unserer eigenen
  Client-Implementierung (Sockets + OpenSSL für HTTPS — D1) für die Produktion und
  einer Fake-/Stub-Implementierung für Tests. Das ist der Seam, von dem die gemockten
  HTTP-Verifikationen in T3.2, T4.1 und T4.3 abhängen — der echte Client wird in Tests
  nie getroffen. Blockierendes Verhalten läuft auf einem Worker-Thread (D2).
- `LlmClient` (auf dem `IHttpClient`-Seam) ruft ein lokales LLM. Konfiguration über
  env: `LLM_BASE_URL` (Server-Root, Default `http://localhost:11434`), `LLM_MODEL`
  (kein eingebackener Default — nur Dokumentation; wenn ungesetzt, ist die LLM-Engine
  nicht verfügbar und der Toggle deaktiviert, nie ein stiller Rateversuch), optional
  `LLM_API_KEY`.
- **Endpoint + strukturierte Ausgabe (gebaut in T3.2/M3, wiederverwendet in
  T4.3/M4):** standardmäßig Ollamas natives `/api/chat` mit dem `format`-Parameter auf
  das `Recipe`-/Makro-JSON-Schema gesetzt — Berichten zufolge ist das native `format`
  (schema-erzwungen) zuverlässiger als der OpenAI-kompatible
  `/v1/chat/completions`-+-`response_format: json_schema`-Pfad, den mehrere Modelle
  ignorieren. Die OpenAI-kompatible Oberfläche bleibt als konfigurierbarer Fallback für
  Nicht-Ollama-Server. So oder so wird jede Antwort mit unserem eigenen Validator (M0
  `jsonschema`) gegen das Schema validiert, mit dem dokumentierten sanften Fallback (ein
  leerer/partieller Draft in die editierbare Preview + eine Warnung — siehe D4/T3.2) bei
  invalider Ausgabe. Keine reine Prompt-Nötigung.
- Das LLM ist aus, solange der Engine-Toggle es nicht wählt, sodass die App voll
  nutzbar ist, ohne dass ein LLM läuft.
- **Keine Übersetzung:** das LLM parst Captions nur in Schema-JSON und bewahrt die
  Originalsprache (Deutsch bleibt Deutsch). Die OFF-Suche kommt mit deutschen Begriffen
  bereits klar, also ist Deutsch↔Englisch-Übersetzung in dieser Phase ausdrücklich
  außerhalb des Scopes.
- **Das Modell ist nicht hart codiert** — zur Laufzeit über `LLM_MODEL` gewählt; ein
  Modellwechsel ist eine env-Änderung + Neustart (das Modell muss in Ollama gepullt
  sein), keine Code-Änderung, kein Rebuild. Dokumentierter Referenz-Default:
  `qwen3.5-9b` (schnell; bestes Deutsch der Familie, und die Fallback-Engine, also zählt
  Tempo). Höhere Qualität: `qwen3.8-27b` für unordentliche Captions. Diese Tags sind
  illustrativ (sie spiegeln die lokalen Launcher-Labels des Nutzers, keine bestätigten
  Ollama-Registry-IDs) — T6.2 muss den exakten `ollama pull`-Tag verifizieren, damit der
  `.env.example`-Default tatsächlich auflöst. Falls strikte JSON-Treue trotz
  strukturierter Ausgabe je zum Flaschenhals wird, ist Gemma 4 27B (Q4_K_M) eine
  notierte Alternative. (Begründung: 2026er-Benchmarks sehen Qwen3 als führend für
  Nicht-Englisch/Deutsch, während JSON-Zuverlässigkeit hier aus dem
  Structured-Output-Modus des Servers + unserer eigenen Schema-Validierung + dem Fallback
  kommt, nicht aus Modell-Gehorsam.)

### D4 — Paste-Parsing: zwei Engines hinter einem Interface
- **Scope dieser Phase: nur Social-Captions** — TikTok-/Instagram-Freitext-Captions
  (die drei Fixtures). **Außerhalb des Scopes:** eingefügter Rezept-Website-Text und
  URL-/HTML-Fetching (schema.org/`Recipe` JSON-LD). Das ist eine saubere spätere
  Ergänzung hinter demselben `RecipeParser`-Interface und wird jetzt NICHT gebaut.
- `RecipeParser`-Interface mit zwei C++-Implementierungen:
  - `RuleBasedParser` — **die Best-Effort-Primär-Engine**: das Alltags-Arbeitstier,
    das die festgelegten Caption-Muster voll abdeckt (Sections→`group`,
    Menge/Einheit-Regex, Makro-Block, to-taste-Zeilen, Klammerausdruck→Notiz, Hashtag-/
    Emoji-Stripping). Offline, kostenlos, deterministisch; die App ist voll nutzbar, ohne
    dass ein LLM läuft.
    - **UTF-8 ist ein First-Class-Anliegen (B3), erledigt durch eigenes Scanning —
      keine Regex-Engine.** Captions stecken voller Multibyte-Inhalt — Emoji (🍗💪🛒),
      Umlaute/ß (`Eiweiß`, `Hähnchen`, `Kohlenhydrate`), `%`/`€`. Konsistent mit der
      From-scratch-Entscheidung (D1) und den minimalen Externals (kein `re2`) macht der
      Parser sein eigenes UTF-8-bewusstes Scanning: zu Unicode-Codepoints dekodieren,
      Emoji/Symbole über Codepoint-Bereiche strippen (keine Byte-Hacks), an Whitespace/
      Interpunktion tokenisieren und Einheiten/Labels/Mengen als Tokens matchen —
      Mengen-Matches am Zeilen-/Token-Anfang verankert, damit `140ml Kochsahne 7%` die `7`
      nicht als Menge liest. Deutsche Einheitenwörter (`TL`/`EL`/`Stück`/`Prise`) und
      Makro-Labels (`Eiweiß`/`Kohlenhydrate`/`Fett`) werden als ganze Tokens gematcht.
  - `LlmParser` — **der Fallback** für unordentliche Captions, die die Regeln
    verfehlen: schickt den eingefügten Text + das `Recipe`-JSON-Schema ans lokale LLM,
    bittet um schema-valides JSON, validiert es. „Fallback" heißt hier nutzer-gewählt (der
    Mensch schaltet auf die LLM-Engine, wenn die Regeln schwach abschneiden) — es gibt
    KEINE automatische Regel→LLM-Übergabe. Wenn das LLM invalides/unparsbares JSON liefert,
    ist das dokumentierte Verhalten, einen Best-Effort- oder leeren Draft mit einer Warnung
    in die editierbare Preview zu bringen (nie ein stilles Speichern, nie ein Auto-Retry mit
    der anderen Engine); der Mensch korrigiert ihn dann in der Preview.
- **UI:** auf dem Paste-Screen wählt der Nutzer die Engine, sieht das geparste Ergebnis
  in einem editierbaren Preview-Formular (nutzt das M2-Formular wieder), korrigiert alles
  Nötige und speichert dann. Parsen speichert nie direkt — der Mensch bestätigt immer.

### D5 — Makros: aus Zutaten (Suche & Auswahl), LLM-Schätzung oder manuell (immer überschreibbar)
- **Nährwert-Quelle: Open Food Facts (OFF)** hinter einem
  `NutritionSource`-Interface. Gewählt statt USDA, weil der Nutzer aus deutschen
  Rezepten kocht: OFF ist mehrsprachig mit starker deutscher Abdeckung und einer
  Textsuche-API, während USDA nur englisch/US-zentriert ist (seine *Zahlen* sind
  universell, aber seine *Namen* passen nicht zu deutschen Zutaten, und deutschtypische
  Lebensmittel wie Quark/Schmand fehlen). Das Interface lässt einen Seam, um USDA (saubere
  generische Vollwert-Werte) später als zweite Quelle zu ergänzen. OFF ist
  ODbL-lizenziert — für privaten, selbst-gehosteten Gebrauch in Ordnung; eine kleine Zeile
  „Data from Open Food Facts (ODbL)" wird in der UI gezeigt.
- **Bekanntes Datenqualitäts-Risiko (M4 — verlagert, nicht beseitigt):** OFF ist eine
  Marken-Produkt-Datenbank mit spärlichen, ungleichmäßigen, nutzer-beigetragenen
  Per-100-g-Daten für generische Vollwert-Lebensmittel (Hähnchenbrust, Rumpsteak, Reis,
  Zwiebel) — genau das, woraus diese Fixtures bestehen. Suche-und-Auswahl beseitigt den
  *Auto-Match*-Fehler, aber der Nutzer wählt für ein Vollwert-Lebensmittel weiterhin unter
  Marken-Einträgen, und die Qualität schwankt. Milderungen (alle schon im Flow): die
  Auswahl-UI bevorzugt Produkte mit vollständigen Nährwerten / einem Nutrition-Grade
  (T4.2b), manuelles Überschreiben ist immer verfügbar (ein generischer Vollwert-Wert, dem
  der Nutzer traut), und der USDA-Seam (saubere generische Werte) ist die vorgesehene
  künftige Zweitquelle für genau diese Lücke. Als echte Limitierung dokumentiert, kein
  gelöstes Problem.
- **Suche-und-Auswahl pro Zutat (Mensch im Loop):** statt einen Match automatisch zu
  raten (das größte Genauigkeits-Risiko des vorigen Plans), sucht der Nutzer OFF für jede
  Zutat und wählt das richtige Lebensmittel; `quantity × per-100 g` → Makros. Das entfernt
  den riskanten unscharfen Auto-Match.
- **OFF-API-Kontrakt (M1 + 2nd-review Major 2 — Endpoint/Service korrekt festgelegt):**
  diese sind unterschiedlich und dürfen nicht vermengt werden. **Primäre Textsuche = die
  klassische OFF-API-v2-Suche, `GET https://world.openfoodfacts.org/api/v2/search`**
  (dokumentiert, stabil). **Search-a-licious** ist der *neuere, separate* Suchdienst auf
  einem *eigenen Host* (`https://search.openfoodfacts.org`, eigener `/search`-Endpoint),
  der v2 irgendwann ablösen soll — ein gültiger künftiger Tausch hinter dem
  `NutritionSource`-Interface, NICHT dasselbe wie `/api/v2/search`. Das alte
  `/cgi/search.pl` ist deprecated („not recommended for new integrations") — nur
  Last-Resort-Fallback. Produkt-Lesungen per Barcode. OFF verlangt einen beschreibenden
  `User-Agent` (z. B. `mise-en-place/0.1 (contact)`) — ein generischer/leerer UA wird
  gedrosselt/blockiert, also setzt unser HTTP-Client ihn immer — und OFF erzwingt
  Rate-Limits mit 429. **Die alte „~100/min Produkt"-Zahl NICHT hart codieren — sie ist
  falsch/zu hoch** (das aktuell berichtete Produkt-Limit ist deutlich niedriger, ~15/min;
  Suche ~10/min). **T4.1 muss die exakten aktuellen Limits gegen die Live-OFF-Docs
  bestätigen** und den Backoff konservativ dimensionieren (ein zu großzügiges Budget lädt
  einen IP-Bann ein). Der Client setzt den UA und behandelt 429 mit exponentiellem Backoff.
- **Per-100-g-Nährstoff-Mapping (festgelegt — vermeidet die kJ/kcal-Falle):**
  `energy-kcal_100g` für Kalorien lesen (Fallback `energy_100g ÷ 4.184`, da `energy_100g`
  in kJ ist), und `proteins_100g` / `carbohydrates_100g` / `fat_100g`. **Produkte ohne die
  `*_100g`-Nährwerte sind nicht auswählbar** und werden geflaggt — nie mit Null gefüllt.
  (~die Hälfte der OFF-Produkte hat keine vollständigen Nährwertdaten.)
- **Cache: erst lokal suchen, OFF nur wenn unzureichend.** Die Suche matcht zuerst die
  lokale `foods`-Tabelle und ruft OFF nur, wenn die lokalen Kandidaten dünn sind; jedes
  *gewählte* Lebensmittel wird persistiert (Schlüssel OFF-Barcode `code`, unique, mit
  `lang` + `fetched_at`), sodass **das Auflösen eines schon gewählten Lebensmittels kein
  Netz braucht**. Mit der Zeit baut das die eigene geprüfte lokale Teilmenge des Nutzers
  auf — schnell, offline wiederverwendbar, OFFs ungleichmäßige Qualität absorbierend (jeder
  Eintrag einmal bei der Auswahl geprüft).
- **Einheit→Gramm-Umrechnung:** OFF liefert per 100 g, also müssen Mengen in Gramm
  aufgelöst werden. Massen-Einheiten (g/kg) sind exakt. **Volumen-Einheiten
  (ml/l/cup/tbsp/tsp) sind dichteabhängig** — OFFs `serving_size` ist Freitext und ergibt
  selten eine brauchbare Dichte, also nutzt der Converter eine kleine eingebaute
  Dichte-Tabelle für gängige Flüssigkeiten (Wasser, Milch, Öl…), fällt nur als letztes
  Mittel auf Wasser-Äquivalent (1 g/ml) zurück und flaggt Volumen-Zutaten ohne Dichte
  (geflaggt, nicht still genullt). Makros bleiben immer überschreibbar.
- **Stück/Löffel-Auflösung (M2 — damit per 100 g bei echten Rezepten nicht dunkel
  bleibt):** die Fixtures sind von Stück- und Löffel-Einheiten dominiert
  (`2 Knoblauchzehen`, `1 rote Zwiebel`, `1 TL`, `2 tbsp`), die die strikte Regel unten
  unaufgelöst ließe — was per 100 g für praktisch jedes echte Rezept auf „—" setzt. Um das
  zu beheben, trägt der Converter zusätzlich eine kleine Stück-Gewichts-Tabelle (z. B. 1
  Zehe ≈ 5 g, 1 Zwiebel ≈ 150 g, 1 Ei ≈ 60 g, 1 Paprika ≈ 150 g) und feste Löffel-Volumen
  (TL/tsp ≈ 5 ml, EL/tbsp ≈ 15 ml → Gramm über die Dichte-Tabelle). Er darf OFFs
  `serving_size` NUR nutzen, wenn es klar als Einzelstück-Gewicht parst; er darf OFFs
  `product_quantity` NICHT nutzen — das ist die *Packungs*-Menge (z. B. 500 g für einen
  Beutel Reis), NICHT ein Pro-Stück-Gewicht (2nd-review Major 2). Das sind approximative
  Defaults, klar überschreibbar. Einheiten ohne Tabellen-Eintrag und ohne Dichte bleiben
  geflaggt (nicht still genullt).
- **Ehrlichkeit approximativer Makros (2nd-review Major 1 — durch den M2-Fix
  eingeführt):** die Stück/Löffel-Tabelle speist SOWOHL den Gewichts-Nenner ALS AUCH die
  Gramm jeder Zutat, also sind, wenn sie beiträgt, die gezeigten Makros pro Portion *und*
  per 100 g approximativ. Um geratene Zahlen nicht als exakt zu präsentieren, trägt das
  Rezept `macrosEstimated: true`, sobald irgendein Stück/Löffel-Tabellen- (oder
  serving_size-abgeleitetes) Gewicht in die Rechnung floss, und die UI markiert diese Makros
  als „estimated". Das stellt die Ehrlichkeit wieder her, die das strikte „—" gab, ohne bei
  echten Rezepten dunkel zu werden. **Dasselbe Flag wird auf dem LLM-Schätzungs-Pfad gesetzt
  (T4.3 + 3rd-review Major):** LLM-geschätzte Makros und Gesamtgewicht sind ihrer Natur nach
  approximativ, also `macrosEstimated: true`, und die UI markiert sie nach Speichern/Neuladen
  als geschätzt, nicht nur während der Eingabe — plus ein `macroSource`-Badge
  (ingredients/llm/manual), damit die Provenienz das Neuladen ohnehin überlebt.
- **LLM-Schätzungs-Button:** fragt das lokale LLM nach Makros pro Portion (und einem
  geschätzten Gesamtgewicht), wenn der DB-Lookup unvollständig ist oder der Nutzer es
  vorzieht.
- **Manuell:** der Nutzer kann Makros immer eintippen/überschreiben.
- Die App speichert **Makros pro Portion + Portionszahl + Gesamt-Rezeptgewicht (g)**.
  **Per 100 g ist ein abgeleiteter Wert** — `perServing × servings ÷ totalWeightG × 100`,
  nicht unabhängig persistiert — in T1.2 fixiert, damit er nicht driften kann. `totalWeightG`
  wird nur automatisch gefüllt, **wenn *jede* Zutat gewählt ist *und* jede Einheit zu Gramm
  auflöst**; ist eine Zutat ungewählt oder hat eine unauflösbare (geflaggte) Einheit, ist das
  Gesamtgewicht partiell, also zeigt per 100 g **„—"** (derselbe Pfad wie manuelle/
  LLM-Einträge ohne Gewicht) statt eines still falschen Werts. Mit der
  M2-Stück/Löffel-Auflösung oben erreichen gängige Rezepte jetzt *tatsächlich* ein volles
  Gewicht; das „—" ist das ehrliche letzte Mittel, nicht der Normalfall. Makro-Felder:
  Kalorien, Protein, Carbs, Fett (erweiterbar).
- **Roh- vs. gekochtes Gewicht (Q10 — dokumentierter Vorbehalt):** `totalWeightG` ist die
  **Summe der rohen Zutaten-Gewichte**, nicht das Gewicht des fertigen Gerichts (Wasser
  verdampft beim Kochen), also ist das abgeleitete per 100 g *pro 100 g Roh-Input* und
  untertreibt das gekochte Gericht leicht. Für diese Phase akzeptabel; in UI/Docs sichtbar
  gemacht, damit die Zahl nicht für Nährwerte auf Kochgewicht-Basis gehalten wird. Der Nutzer
  kann `totalWeightG` mit einem gemessenen Kochgewicht überschreiben, wenn er per 100 g auf
  Kochgewicht-Basis will.

### D6 — Medien, Quell-Link, Deployment
- **Bilder (M6 — konkrete Limits, nicht nur „validiert"):** optionale Uploads über
  unseren eigenen multipart/form-data-Parser (in `net` bei T5.1 ergänzt) → Disk-Volume,
  per URL referenziert. Das Backend generiert seinen eigenen Dateinamen (UUID + validierte
  Extension) und traut dem Client-Dateinamen nie (kein Path-Traversal); es erzwingt Größe
  ≤ 8 MB und Content-Type ∈ {jpeg, png, webp}, per Magic-Bytes verifiziert, nicht nur per
  Header. Das Uploads-Verzeichnis wird in prod von nginx ausgeliefert (ein `location
  /uploads/`-Block), nicht von der App.
- **Externe Bild-URL (M6b — kein SSRF):** die Option „externe Bild-URL" ist
  nur-speichern — die URL wird gespeichert und vom Browser gerendert; das Backend holt sie
  NICHT. Das schließt das SSRF-Loch (ein serverseitiger Fetch könnte das Heim-LAN / Ollama /
  Postgres treffen).
- **Quell-Link:** optionales URL-Feld, auf der Rezeptseite als Link gezeigt
  (nur-speichern).
- **Deploy:** Docker Compose — Backend (mehrstufiger C++-Build → schlankes Runtime),
  Frontend (statischer Build, von nginx ausgeliefert), Postgres (mit einem Volume); Ollama
  ist der eigene Dienst des Nutzers, referenziert über `LLM_BASE_URL`. Läuft auf einem
  Home-Server / VPS. Braucht ausgehendes Netz für erstmalige Open-Food-Facts-Lookups (danach
  gecacht, den OFF-Rate-Limits unterworfen); optional Ollama für LLM-Features. **Graceful
  Degradation:** ist OFF nicht erreichbar, liefert die Suche nur-Cache-Ergebnisse mit einer
  klaren Meldung, und manuelle + LLM-Makro-Eingabe funktionieren weiter — ein Netzausfall
  blockiert die Rezept-Eingabe nie.
- **Build-Kosten (Q8 — jetzt minimal):** mit `apt`-System-Paketen installieren
  libpq/OpenSSL/Catch2 als vorgebaute Binaries — sie werden NICHT aus dem Quellcode
  kompiliert — also ist das OOM-Risiko des alten Drogon-/vcpkg-from-source-Builds im
  Wesentlichen weg. Nur unser eigener Code kompiliert. (Base-Image-apt-Layer cachen von
  selbst; die apt-Liste + minimalen Build-RAM in T6.2 dokumentieren.)
- **Auth-Annahme (M6c):** die App ist in dieser Phase unauthentifiziert (eine Person).
  Weil sie vom Handy aus mit vollem Schreib- + Upload-Zugriff erreichbar ist, MUSS sie hinter
  einem VPN oder einem Reverse-Proxy mit Basic-Auth *über TLS/HTTPS* laufen — Basic-Auth ohne
  TLS schickt Credentials im Klartext über eine handy-erreichbare Verbindung, also ist
  **HTTPS Teil des Vorbehalts, nicht optional** — bis In-App-Auth kommt. Das Schema ist
  auth-ready (D2).

## Einheitliches Rezeptformat (`Recipe`) — festgelegte Feldliste
> Bei GATE 0 festgelegt und gegen echte deutsche + englische TikTok-Captions
> validiert (drei durchgearbeitete Beispiele: eine video-only-Caption mit vorberechneten
> Makros, ein gruppiertes Rezept mit mehreren Sections, und ein englisches mit Steps +
> einer Aufbewahrungs-Notiz). Das JSON-Schema in `docs/` ist die Single Source of Truth,
> formalisiert in T1.2; das C++-Struct und der TS-Typ spiegeln es. Weil das Format eine
> `schemaVersion` trägt, ist jede Verfeinerung, die beim Bau des Paste-Parsers (T3.x)
> auftaucht, eine versionierte Migration, kein Redesign — die Feldliste kann sich bewusst
> weiterentwickeln, ohne gespeicherte Rezepte zu brechen.

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
**Verworfen / eingefaltet** (bei Bedarf später via `schemaVersion` wieder aufgreifen):
`cuisine` und `category` → in `tags` einfalten; `difficulty` → ausgelassen (subjektiv,
geringer Nutzen); `yield` → `servings` (numerisch) ist maßgeblich für die Makro-Rechnung;
ein numerisches `rating` → ersetzt durch das boolesche `favorite`.

---

## Milestone 0 — Core-Backend-Bibliotheken (from scratch)
> Neuer Milestone aus der From-scratch-Wende vom 2026-08-24 (D1). Baut das Plumbing,
> das ein Framework uns gegeben hätte, damit spätere Milestones Bibliotheken zum
> Draufstehen haben. Alles hier ist isoliert unit-getestet; noch keine Rezept-Logik.
- **T0.1** Toolchain + Projekt-Skeleton: ein einfaches **GNU-Makefile** (**C++20**
  pinnen — 5th-review minor, ein Standard, nicht „C++17/20"), das die **System-(apt-)
  Pakete** `libpq-dev`, `libssl-dev`, `libutf8proc-dev`, `catch2`/`libcatch2-dev` über
  **`pkg-config`** findet (`pkg-config --cflags --libs libpq openssl libutf8proc`; kein
  vcpkg, kein Framework) + das Layout `/backend/lib` (`/net /router /json /jsonschema /db
  /httpclient`) + `/app` + `/tests` (D1) + ein Catch2-Test-Target. **Warum Make, nicht
  CMake:** der Build ist **Linux-only** (dev in WSL2, Deploy auf Linux hinter einem VPN —
  D6), also bringt CMakes Hauptnutzen (Cross-Plattform-Generierung) hier nichts, während ein
  handgeschriebenes Makefile einfacher ist und die tatsächlichen Compile-/Link-Aufrufe
  sichtbar hält — was zum Ziel „die Grundlagen lernen" passt (D1). Falls je ein
  Windows-Target dazukommt, ist der Rückgriff auf CMake ein sauberer späterer Wechsel. **Eine
  `ASAN+UBSAN`-Build-Config (`-fsanitize=address,undefined`), unter der die Catch2-Suite läuft
  (5th-review S2 — das wichtigste Sicherheitsnetz beim Handbauen von vier
  Untrusted-Input-Parsern), als Makefile-Target (z. B. `make asan`), plus TSan-Abdeckung für
  die Concurrency-Arbeit (der T0.9-thread-pool und der T0.6-db-Connection-Pool).** Außerdem
  **hier den dev-`docker-compose`-Postgres-Dienst aufstellen** (nach vorn gezogen, damit ganz
  M0 mit einer DB reden kann — 5th-review P3). Entwickelt wird in **WSL2 (Ubuntu)**. *Verify:*
  `make` baut; ein trivialer Catch2-Test läuft grün **unter ASAN+UBSAN** (`make asan`); der
  dev-Postgres-Container kommt hoch.
- **T0.2** `net` — TCP-Socket-Listener + **HTTP/1.1-Request-Parser** (Request-Line,
  Header, Body via `Content-Length` **und** chunked) + Response-Writer + keep-alive.
  **Vorerst single-threaded** — eine Verbindung zur Zeit; der **thread pool ist auf T0.9
  verschoben** (D2), damit diese Task auf korrekte Socket-Behandlung und HTTP-Parsing ohne
  Nebenläufigkeit fokussiert bleibt. Härtung: Header-Größe/-Anzahl **und Gesamt-Body-Größe
  cappen (auch *während* des chunked-Decodings erzwungen, wo es kein Vorab-`Content-Length`
  gibt)**; Malformtes mit `400` abweisen; ein **Socket-Read-/Idle-Timeout** (Anti-Slowloris).
  **RFC 7230 §3.3.3 Framing / Anti-Smuggling (5th-review T2):** **sowohl `Content-Length` als
  auch `Transfer-Encoding` vorhanden** abweisen, **doppeltes/widersprüchliches
  `Content-Length`** abweisen, nur `Transfer-Encoding: chunked` akzeptieren (sonst `400/501`),
  **Overflow-Check** von Chunk-Size-Hex und `Content-Length`, und **bare CR / bare LF /
  eingebettetes NUL** in Request-Line und Headern abweisen. **Den Parser fuzzen (5th-review
  S2):** ein libFuzzer/AFL++-Harness auf dem HTTP-Parser ist Teil dieser Task. *Verify:*
  Unit-Tests parsen wohlgeformte und malformte Requests (partiell, übergroße Header, falsches
  `Content-Length`, **CL+TE beide vorhanden, doppeltes CL, übergroße Chunk-Size, bare CR/LF**);
  ein Integrationstest macht einen echten localhost-Request; übergroßer Body/Header (inkl. eines
  chunked-Body über dem Cap) werden abgewiesen, nicht OOM'd; eine stehende Verbindung wird bei
  Timeout geschlossen; **der Fuzz-Harness läuft sauber über ein gesetztes Budget**.
- **T0.3** `router` + Request/Response-Abstraktion: `method + path` (mit `:id`-Params) →
  Handler; unbekannter Pfad → `404`, falsche Methode → `405`; die **Error-Envelope- +
  Warning-Envelope-Helfer** (D1). *Verify:* Routing-Tests inkl. `:id`-Extraktion, `404`/`405`
  und ein envelope-förmiger Error-Body.
- **T0.4** `json` — eigener Parser + Serializer (Objekte, Arrays, Strings **mit
  Unicode-Escapes inkl. `\uXXXX`-Surrogate-Pairs, zu U+10000+-Codepoints kombiniert —
  5th-review T3**, Zahlen, `true`/`false`/`null`), Round-Trip zu einem kleinen DOM/Value-Typ,
  mit einem **Rekursionstiefen-Cap**, damit ein tief verschachtelter Body (von OFF/LLM) keinen
  Stack-Overflow (DoS) auslösen kann. **Den Parser fuzzen (5th-review S2):** ein
  libFuzzer/AFL++-Harness ist Teil dieser Task. *Verify:* Round-Trip-Tests inkl. verschachtelter
  Strukturen, Unicode-Escapes, **Surrogate-Pairs**, große/Edge-Zahlen; malformter Input → ein
  klarer Parse-Fehler (nie ein Crash); eine **Tiefe über dem Cap wird abgewiesen**, kein Crash;
  **der Fuzz-Harness läuft sauber über ein gesetztes Budget**.
- **T0.5** `jsonschema` — eigener Validator für die **authored subset**, die die App
  nutzt (Typen, `required`, `enum`, Arrays, **Nullability als `"type":[…,"null"]`-Union — NICHT
  ein `nullable`-Keyword, das OpenAPI ist, nicht JSON-Schema, und still unter-validieren würde;
  5th-review T3**, und `$ref` auf `#/definitions/Macros`), meldet den fehlerhaften Pfad.
  Getestet gegen ein **generisches Fixture-Schema, inline in M0 verfasst**
  (Typen/enum/required/`$ref`) — das echte `Recipe`/`Macros`-Schema wird später in T1.2
  verfasst, also darf M0 nicht davon abhängen (5th-review P3). *Verify:* akzeptiert ein valides
  Fixture-Dokument; weist jede Verletzungsklasse (falscher Typ, fehlendes required, falsches
  enum, falsches `$ref`-Ziel, **`null` in einem non-nullable Feld**) mit dem Pfad ab, und
  **akzeptiert `null` in einem `[…,"null"]`-Feld**.
- **T0.6** `db` — libpq-Wrapper: ein **Connection-Pool**, **parametrisiertes `exec`**
  (`$1,$2…`-Binds — SQL wird nie string-konkateniert), Result→Row-Mapping und ein
  Transaktions-Helfer; plus ein **standalone (nicht gepooltes) `connect()`**, das der
  Migration-Runner (T1.1) nutzt, um seinen Advisory-Lock auf einer dedizierten Connection zu
  halten (D2); plus ein **rohes `PQexec`, dokumentiert als migration-only, nie für User-Input**
  (Migrationsdateien sind mehr-Statement, was parametrisiertes `PQexecParams` nicht ausführen
  kann — Entwickler-verfasste SQL, also kein Injection-Bedenken; 5th-review minor). *Verify:*
  gegen ein dev-Postgres liefert ein parametrisierter Round-Trip Zeilen; ein Wert mit
  SQL-Metazeichen, als **Bind** übergeben, wird literal gespeichert/zurückgegeben (Injection
  inert); der Pool gibt Connections unter nebenläufiger Nutzung aus und nimmt sie zurück; eine
  standalone-Connection kann außerhalb des Pools geöffnet werden.
- **T0.7** App-Wiring: `main()` startet den `net`-Server auf einem Port, mountet den
  `router`, baut den `db`-Pool aus env, ergänzt strukturiertes Logging + Config;
  `/health`-Endpoint. *Verify:* die App bootet; **`GET /health` → 200 durch unseren eigenen
  Server**; Logs zeigen eine erfolgreiche DB-Connection.
- **T0.8** `httpclient` — der **eigene HTTP/1.1-Client**, der den `IHttpClient`-Seam
  erfüllt (D3), genutzt von den LLM- + OFF-Clients (T3.2/T4.1/T4.3). Scope: über Sockets
  verbinden; einen HTTP/1.1-Request schreiben; die Response inkl. **chunked-Transfer-Decode**
  lesen; Response-Body-Größen-Cap; **HTTPS über OpenSSL** für OFF (öffentliches Internet);
  **plain HTTP** für den lokalen Ollama-Pfad; ein Read-/Connect-**Timeout**.
  - **TLS muss den HOSTNAME verifizieren, nicht nur die Chain (5th-review S1 — kritisch):**
    `SSL_VERIFY_PEER` allein prüft nur die Zertifikats-*Chain*; **SNI
    (`SSL_set_tlsext_host_name`) verifiziert NICHTS** (es sagt dem Server nur, welches Cert er
    servieren soll). Ohne Hostname-Verifikation MITMt jeder Angreifer mit *irgendeinem*
    CA-validen Cert den OFF-Pfad. Also: **`SSL_set1_host(ssl, host)`** (oder
    `X509_VERIFY_PARAM_set1_host` via `SSL_get0_param`) **vor** dem Handshake aufrufen,
    `SSL_VERIFY_PEER` + SNI setzen, den Trust-Store laden
    (`SSL_CTX_set_default_verify_paths`) und **`SSL_get_verify_result == X509_V_OK` asserten**.
  - *Verify (echtes Netz, aus der Default-Unit-Suite herausgehalten):* ein HTTPS-`GET` zu
    einem bekannt guten Host gelingt; **ein Host mit validem CA, aber FALSCHEM Hostname (z. B.
    `wrong.host.badssl.com`) wird abgewiesen** (das ist der Fall, der chain-only von echter
    Hostname-Verifikation unterscheidet — ein selbst-signierter/abgelaufener Host würde
    durchgehen, während der Code noch exploitierbar ist); ein **plain-HTTP**-GET zu einem
    lokalen Test-Server funktioniert; eine chunked-Response dekodiert korrekt; der
    Fake-`IHttpClient` bleibt das, was die T3.2/T4.1/T4.3-Unit-Tests nutzen.
- **T0.9** `net` **Nebenläufigkeit — thread pool (zuletzt in M0 gebaut, bewusst).** Alles
  bis T0.8 läuft single-threaded; diese Task fügt das Concurrency-Modell hinzu, um das es beim
  From-scratch-Ziel wirklich geht (D2), jetzt wo Sockets/Parsing/Routing/DB/HTTP-Client alle
  isoliert funktionieren. Scope: ein **fester Worker-Thread-Pool** (z. B. 4–8 Worker,
  env-konfigurierbar); der Accept-Loop übergibt jede akzeptierte Verbindung an einen Worker;
  **ein Worker besitzt eine libpq-Connection aus dem `db`-Pool (T0.6) für die Lebensdauer des
  Requests**, und die blockierenden `exec`-/HTTPS-Aufrufe laufen auf dem Worker, nie auf dem
  Accept-Loop (D2). Die Work-Queue beschränken; den Pool sauber herunterfahren (In-Flight
  drainen, Worker joinen). **Unter TSan bauen + testen** (die Config in T0.1 aufgestellt) —
  dies und der T0.6-Connection-Pool sind die zwei Stellen, an denen Data-Races leben können.
  *Verify:* der Server behandelt **nebenläufige** keep-alive-Clients korrekt (keine
  verschränkten/korrupten Antworten); **TSan ist sauber** unter nebenläufiger Last; der Pool
  beschränkt seine Queue, statt unbegrenzt zu wachsen; ein sauberes Shutdown drainet und joint
  ohne Leak (ASAN sauber).
- **M0-Review** — der `workflow:review` an der Grenze besteht _(die Bibliotheken sind das
  Fundament, auf dem alles andere steht — eine sorgfältige Lektüre wert)._

## Milestone 1 — Schema, DB, Browse-API (auf den M0-Libs)
- **T1.1** **Migration-Runner** (auf der M0-`db`-Lib) mit einer
  `schema_migrations`-Tabelle: einfache SQL-Dateien, jede Migration in ihrer **eigenen
  Transaktion** (Version nur bei Erfolg festgehalten), der Runner hält einen **session-level
  `pg_advisory_lock` auf einer dedizierten libpq-Connection über alle
  Per-Migration-Transaktionen hinweg** (am Ende freigegeben), sodass nebenläufige Boots
  serialisieren (Q7 + 2nd-review Major 3, gemäß D2); `docker-compose` mit einem Postgres-Dienst
  für dev. *Verify:* Migrationen wenden beim Boot an; ein erneuter Lauf wendet nicht erneut an;
  eine absichtlich fehlschlagende Migration lässt `schema_migrations` unverändert (die
  Transaktion dieser Migration rollt zurück).
- **T1.2** `Recipe`-JSON-Schema in `docs/` (Source of Truth — die **festgelegte
  Feldliste** oben), verfasst zu der **in sich konsistenten Teilmenge, die unser eigener
  Validator implementiert** (D2 — nicht an den Draft einer Library gebunden). **Außerdem den
  Makro-Body als benanntes Sub-Schema `definitions/Macros` verfassen** (die
  `{calories,protein,carbs,fat}`-Form), referenziert von `macrosPerServing` via `"$ref":
  "#/definitions/Macros"`; `POST /api/macros/compute` und `POST /api/macros/estimate`
  validieren ihre Request-/Response-Makro-Bodies dagegen, und das Ollama-`format` für den
  Makro-Schätzer (D3) nutzt dasselbe Sub-Schema — sodass der D1-`422`-Pfad und jeder
  „schema-validated **macro** body"-Schritt ein verfasstes Schema hat, keine zweite Source of
  Truth (3rd-review minor #3). SQL-Migrationen für die **in D2/B1 entschiedene relationale
  Form**: `users`-Stub; `recipes` (nullable `owner_id`, Skalar-Felder, **Makro-Spalten pro
  Portion cal/protein/carbs/fat, `total_weight_g`, `macro_source`, `macros_estimated`**);
  **Kind-Tabellen `recipe_ingredients`** (FK, `position`, nullable
  `quantity`/`unit`/`group_label`/`food_id`/`note` — **`food_id` ist hier eine einfache nullable
  UUID-Spalte, noch kein FK** (die `foods`-Tabelle landet in T4.1), und die Section-Spalte ist
  `group_label`, nicht das reservierte Wort `group`; sprach-neutrales Einheiten-Vokabular),
  **`recipe_steps`** (FK, `position`, `text`), **`recipe_images`** (FK, `position`, `url`);
  **Join-Tabelle `recipe_tags`** (FK, `tag`, `position`) — alle Kind-/Join-FKs **`ON DELETE
  CASCADE`**; C++-Model-Structs + **unsere eigene JSON-(De)Serialisierung (M0 `json`)**, die
  **das kanonische `Recipe`-JSON aus diesen Tabellen zusammensetzt/emittiert** (beachte die
  JSON↔Spalten-Namens-Maps, z. B. `macrosPerServing.calories` ↔ Spalte `cal`) + eine
  Validierungsfunktion mit dem **M0-`jsonschema`-Validator**. *Verify:* Migrationen wenden auf
  einer frischen DB an; ein Unit-Test round-trippt ein `Recipe`-Struct↔JSON (über die
  Kind-Tabellen) — inkl. eines **gruppierten, to-taste-Zutaten-Rezepts** (null quantity/unit),
  geordneter `steps`, eines **leeren-`steps`**-Rezepts, **`tags[]` + `images[]` mit erhaltener
  Reihenfolge** **und `macrosEstimated: true`, das den Round-Trip überlebt** — und der Validator
  **weist ein schema-invalides Dokument ab** und akzeptiert ein valides.
- **T1.3** Recipe-Repository/Service (create, read, list, update, delete) über die **M0-
  `db`-Lib** (parametrisiertes `exec`) — schreibt/liest über die Eltern- + Kind-Tabellen in
  einer Transaktion; per-100-g-abgeleitete Berechnung. **Update-(PUT-)Strategie (2nd-review
  Blocker): ein Full-Replace innerhalb einer Transaktion** — der Client PUTet das *komplette*
  Rezept (das M2-Formular hält bereits jeden `foodId` der Zutaten, also round-trippt es sie),
  und der Service ersetzt die Kind-Zeilen aus diesem Payload und weist `position` aus der
  Array-Reihenfolge neu zu; **ein gewählter `food_id` überlebt eine Bearbeitung**, weil der
  Payload ihn trägt (ein bloßes delete-and-reinsert, das `food_id` fallen ließe, wird
  ausdrücklich abgelehnt). **Bei PUT berechnet das Backend `macrosEstimated` aus der Zutaten-/
  Gewichts-Auflösung des Payloads neu — AUSSER wenn `macroSource=="manual"`, wo das Flag des
  Clients unverändert übernommen wird (4th-review Red #2):** eine manuelle Bearbeitung behauptet
  exakte Werte und löscht das Flag (gemäß der Format-Präzedenz-Regel), also darf ein Recompute
  über noch-approximative Zutaten-Einheiten es **nicht** wieder auf `true` kippen und den
  Zustand „geraten als exakt gezeigt" wieder einführen. Recompute gilt nur für die
  `ingredients`-/`llm`-Quellen. **Bild-Datei-GC (3rd-review minor #6):** weil `ON DELETE
  CASCADE` die `recipe_images`-Zeilen entfernt, liest der Service **zuerst die eigenen
  Bild-Dateinamen**, löscht dann das Rezept, dann unlinkt er die backend-eigenen Dateien
  (best-effort, bei Fehler geloggt — eine verwaiste Datei ist eine Warnung, nie ein
  fehlgeschlagener Request); bei PUT diffed er alte vs. neue Bild-URLs und unlinkt die
  fallengelassenen **eigenen** Dateien nach dem Commit (externe-URL-Bilder sind nur-speichern —
  nie angefasst). *Verify:* Integrationstests laufen gegen ein **dediziertes Test-Postgres**
  (Compose-Dienst; Migrationen vor der Suite angewendet; **jeder Test truncatet die
  Rezept-Tabellen** zum Determinismus — Q1) für CRUD inkl. eines Multi-Group-/Multi-Step-Rezepts,
  **einer PUT-Bearbeitung, die `food_id`-Picks erhält und Zutaten umsortiert**, plus ein
  Unit-Test für per 100 g (inkl. des „kein Gewicht" → null-Pfads).
- **T1.4** REST-Controller `GET /api/recipes` (**paginierte Summary-Liste** —
  `?limit=&offset=`, Summary-Projektion gemäß D1, nicht die voll aus Kind-Tabellen
  zusammengesetzten Objekte) und `GET /api/recipes/:id` (volles kanonisches `Recipe`); seede
  2–3 Beispiel-Rezepte (**mit `food_id` auf null gelassen**, damit der T4.1-FK-Add keine Waisen
  findet). *Verify:* ein Integrationstest trifft beide Endpoints — die Liste liefert Summaries,
  die `limit`/`offset` respektieren, `/:id` liefert ein schema-valides volles `Recipe`.

## Milestone 2 — Frontend-Scaffold, Browse/Detail, strukturiertes Formular
- **T2.0** UI-Mockups (Design vor dem Bau) — die Schlüssel-Screens in `docs/mockups/`
  mocken (Browse/Liste + Suchleiste, Detail mit Makro-Tabellen/`—`/estimated +
  `macroSource`-Badge, Add/Edit-Formular inkl. Tags/Favorit/Notizen/Zeiten, Paste-Import
  editierbare Preview, OFF-Suche-und-Auswahl, Empty-/Error-/Warning-States). *Verify:* jeder
  Screen hat einen abgestimmten Mock, damit T2.1–T2.3 gegen ein entschiedenes Design bauen.
  _(Diese Task ist der Owner des Mockup-Schritts im Plan, den die ROADMAP als T2.0 führt — die
  zwei stimmen jetzt überein; 4th-review minor.)_
- **T2.1** Angular-CLI-Scaffold (**exakt gepinnte Version `17.x.y`**, nicht `^17` — Q3) +
  **dev-`proxy.conf.json`** (`/api` → Backend) + typisierter API-Service (`HttpClient`), der
  **gegen den D1-Error-Envelope codet** (eine `{error:{code,message,details}}`-Form) +
  Browse-Listen-Seite (**die paginierte Summary-Liste konsumierend**) + Detail-Seite, die
  Titel, Bild, Quell-Link, Zutaten, Steps, beide Makro-Tabellen rendert, **und `tags` (Chips),
  `favorite` (Stern), `notes`, `prepTimeMin`/`cookTimeMin` (4th-review Red #1 — diese Felder
  existieren im Format und werden in T5.2 gefiltert, also müssen sie rendern), und die
  per-Zeile-`note` jeder Zutat (z. B. „(uncooked weight)"), damit Notizen nicht still
  verschwinden (5th-review minor)** (**den „estimated"-Marker zeigend, wenn `macrosEstimated` —
  sowohl auf der Detail-Seite als auch in der Browse-Liste, deren Summary-Projektion das Flag
  bereits trägt — plus ein kleines `macroSource`-Badge (ingredients/llm/manual) auf der
  Detail-Seite, damit die Provenienz das Neuladen überlebt; **das Badge wird weggelassen, wenn
  `macrosPerServing` ungesetzt/all-zero ist**, sodass ein makro-loses Rezept keines zeigt**).
  *Verify:* `ng build` besteht und `ng test` läuft grün mit **ChromeHeadlessNoSandbox**
  (Chromium in der Test-Umgebung installiert); gegen die laufende API (über den dev-Proxy)
  rendern die Liste + eine Detail-Seite ein geseedetes Rezept mit beiden Makro-Spalten
  (Component-Test wo praktikabel).
- **T2.2** Add/Edit-Formular mit Angular **Reactive Forms** (Titel, Beschreibung,
  servings, Gewicht, dynamisches Zutaten-Zeilen-`FormArray`, steps, Quell-URL, Bilder,
  Makro-Felder, **`tags` (Chips hinzufügen/entfernen — der *einzige* Weg, Tags zu setzen, da
  der Parser nie auto-taggt), `favorite` (Toggle), `notes`, `prepTimeMin`, `cookTimeMin`** —
  4th-review Red #1: ohne diese wären die T5.2-Tag-/Favorit-Filter von Anfang an tot) →
  `POST`/**`PUT /api/recipes/:id`** mit **serverseitiger Validierung** im Backend
  (**schema-invalid → `422`** mit den fehlerhaften Pfaden im Error-Envelope; **unbekannte id →
  `404`** — die id steht in der PUT-URL, gemäß D1); **`PUT` schickt das komplette Rezept**,
  damit `food_id`-Picks den Full-Replace überleben (T1.3). Plus ein **`DELETE
  /api/recipes/:id`**-Controller (der das T1.3-Repository-`delete` freilegt; Kind-Zeilen
  kaskadieren), verdrahtet mit einer Delete-Aktion in der UI mit einer Bestätigung. *Verify:*
  eine valide Eingabe persistiert und erscheint im Browse; **ein mit Tags + Favorit
  gespeichertes Rezept round-trippt und wird dann von den T5.2-Tag-/Favorit-Filtern
  gefunden**; **ein schema-invalider POST liefert `422`** mit dem Error-Envelope; **eine
  PUT-Bearbeitung erhält gewählte `food_id`s**; **das Löschen eines Rezepts entfernt es (und
  seine Kind-Zeilen) aus `GET /api/recipes`** (Backend-Validierung + Delete-API-Test + ein
  Formular-Validierungs-Component-Test).
- **T2.3** **Manuelle Makro-Eingabe + Override** ins Formular verdrahten (self-contained;
  noch keine Nährwert-DB). *Verify:* ein mit manuell eingegebenen Makros gespeichertes Rezept
  persistiert und rendert beide Makro-Spalten; per 100 g zeigt „—", wenn kein Gewicht angegeben
  ist. _(Auto-„compute from ingredients" ist auf M4 verschoben — siehe T4.2b — weil der
  Compute-Pfad nicht existiert, bis die Nährwert-DB kommt.)_

## Milestone 3 — Paste-Import mit wählbarer Engine
- **T3.1** `RecipeParser`-Interface + `RuleBasedParser` in C++ + `POST /api/parse`. Scope
  ist **nur Social-Captions** (TikTok/Instagram); die regelbasierte Engine ist der
  **Best-Effort-Primär**-Parser (LLM ist der Fallback in T3.2). Konkrete zu behandelnde Muster
  (aus den durchgearbeiteten deutschen + englischen Caption-Beispielen): **Zutaten-Sections**
  (`🍗 Für das Hähnchen:` / `Crispy Beef Strips` → jeder Zeilen-`group`); **Menge/Einheit-Regex**
  über das sprach-neutrale Vokabular (`600 g`, `140 ml`, `2 Knoblauchzehen`→`Stück`, `1 TL`,
  `2 tbsp`; ein bloßes `tsp black pepper` defaultet auf quantity 1); **Klammerausdrücke →
  `note`** (`(diced)`, `(uncooked weight)`, `(tenderises the beef)`); **servings** aus Prosa
  (`4 Portionen`, `Serves 4`); ein **Makro-Block** über Label-Synonyme (`kcal`/`calories`;
  `Eiweiß`/`Protein`/`P`; `Kohlenhydrate`/`Carbs`/`C`; `Fett`/`Fat`/`F`) in beliebiger
  Reihenfolge → `macrosPerServing`, `macroSource: "manual"`; **Hashtag-Wände + Emoji gestrippt**
  (nie auto-getaggt); **to-taste-Zeilen** (`Salz + Pfeffer`, `Petersilie zum garnieren`) → null
  quantity/unit; und jeder **Aufbewahrungs-/Aufwärm-Block → `notes`**. Steps können fehlen
  (video-only). **UTF-8-Behandlung gemäß D4/B3: den Input zuerst NFC-normalisieren — via
  `utf8proc`** (`utf8proc_NFC`; eingefügte Captions kommen evtl. NFD-dekomponiert an, z. B. `ä`
  = `a`+U+0308, was Ganz-Token-Matches für `Eiweiß`/`Hähnchen` und das
  Codepoint-Range-Emoji-Stripping brechen würde; volles NFC braucht Unicode-Tabellen, also ist
  das die eine Stelle, wo wir eine externe Lib nutzen — 4th-review Major), dann **unser eigenes
  UTF-8-Scanning (kein `std::regex`, keine Regex-Engine), Codepoint-Range-Emoji-/
  Symbol-Stripping und zeilen-anfang-verankerte Mengen**, damit `140ml … 7%` die `7` nicht
  falsch liest. **Partial-Parse-Verhalten (4th-review minor):** `/api/parse` **liefert immer
  einen Draft, nie `422`** — eine Caption, die die Regeln nur teilweise parsen (z. B. kein
  Titel), ergibt einen partiellen `Recipe`-Draft (200 + der Warning-Envelope), der in der
  editierbaren Preview landet, damit der Mensch ihn vervollständigt; strikte Schema-Validierung
  gilt beim **Speichern** (T2.2), nicht beim Parsen. *Verify:* Unit-Tests parsen die drei
  repräsentativen Captions in die erwarteten strukturierten Felder, inkl. Gruppierung,
  null-quantity-Zeilen, dem extrahierten Makro-Block, leeren Steps, **korrekter Emoji-/
  Umlaut-Behandlung (`Eiweiß`, `Hähnchen`), einer NFD-dekomponierten Input-Variante (utf8proc
  NFC) und dem `7%`-kein-Menge-Fall**; eine absichtlich partielle Caption liefert einen
  partiellen Draft + Warnung, kein `422`.
- **T3.2** `LlmClient` (auf dem **`IHttpClient`-Seam** aus D3/B2 — die echte Impl ist der
  T0.8-`httpclient`, plain-HTTP für lokales Ollama) + `LlmParser` (Ollama nativ `/api/chat`
  `format`=schema standardmäßig, OpenAI-kompatibler Fallback; Schema-Validierung; Fallback bei
  invalid). Dokumentierter Fallback = bei invalidem/unparsbarem LLM-JSON einen **Best-Effort-
  oder leeren Draft in die editierbare Preview mit einer Warnung** zurückgeben (kein Auto-Retry,
  kein stilles Speichern). *Verify:* Unit-Test mit dem **Fake-`IHttpClient`** (kein Live-LLM,
  kein echtes Netz) prüft, dass valides JSON akzeptiert wird und malformtes JSON diesen
  dokumentierten Fallback auslöst — ein leerer/partieller Draft plus ein Warn-Flag, keine
  Exception und kein gespeicherter Record.
- **T3.3** Paste-Screen: Textarea, Engine-Toggle (regelbasiert / lokales LLM), Parse →
  **editierbares Preview-Formular** (nutzt das M2-Formular wieder) → speichern. *Verify:* das
  Einfügen eines Samples mit der regelbasierten Engine erzeugt ein vorbefülltes, editierbares
  Formular, das korrekt speichert.

## Milestone 4 — Makros aus Open Food Facts (Suche & Auswahl) + LLM-Schätzung
- **T4.1** `NutritionSource`-Interface + **OFF-Client** (auf dem **`IHttpClient`-Seam**
  aus D3/B2 — die echte Impl ist der T0.8-`httpclient`, **HTTPS mit Cert-Verifikation** zum
  öffentlichen OFF-Host → **OFF-API-v2-Suche `/api/v2/search`** primär (**nicht**
  Search-a-licious, das ein separater Service/Host ist — siehe D5 Major 2), das alte
  `/cgi/search.pl` als Last-Resort-Fallback; **beschreibender `User-Agent`**; 429/Backoff) +
  Migration, die die **`foods`-Cache-Tabelle (Surrogat-UUID-`id`-PK, `code` UNIQUE)** erstellt
  und **das FK-Constraint `recipe_ingredients.food_id → foods.id`** auf der schon aus T1.2
  existierenden Spalte ergänzt (keine neue Spalte) + `GET /api/foods/search?q=` (**erst lokale
  `foods`, live OFF nur wenn unzureichend**, dann gewählte Ergebnisse persistieren) + das
  per-100-g-Nährstoff-Mapping (`energy-kcal_100g`, sonst `energy_100g ÷ 4.184`;
  proteins/carbohydrates/fat `_100g`) + ein **Einheit→Gramm-Converter mit der Dichte-Tabelle
  UND der M2-Stück-Gewichts-/Löffel-Volumen-Tabelle**. **Zuerst die aktuellen OFF-Rate-Limits
  gegen die Live-Docs bestätigen (M1)** — die alte „~100/min"-Zahl nicht hart codieren — und den
  Backoff konservativ dimensionieren. *Verify:* Unit-Test mit dem **Fake-`IHttpClient`** liefert
  Kandidaten; **das Auflösen eines schon gewählten Lebensmittels (per `food_id`/Barcode) macht
  keinen API-Call** (Cache-Hit); ein **kJ-only-Mock** wird umgerechnet (oder abgewiesen), nie
  roh summiert; Einheiten-Umrechnungs-Tests (g/kg/ml/l + **Stück-Einheiten wie
  `Knoblauchzehen`/`Zwiebel` und Löffel `TL`/`EL`**) inkl. des „kein Tabellen-Eintrag, keine
  Dichte → geflaggt"-Pfads.
- **T4.2** Makro-Engine: für Zutaten mit einem gewählten `foodId` `quantity × per-100 g`
  summieren → Totale → pro Portion + per 100 g. Zutaten mit **keinem Pick**, ein Produkt **ohne
  `*_100g`-Nährwerte** oder eine **unauflösbare Einheit** werden der UI sichtbar gemacht (nie
  genullt), und jede davon macht `totalWeightG` partiell → per 100 g rendert **„—"**. **Setzt
  `macrosEstimated: true`, sobald ein Stück/Löffel-Tabellen- (oder serving_size-)Gewicht in die
  Rechnung floss** (2nd-review Major 1), damit die UI diese Makros als geschätzt statt exakt
  markieren kann. *Verify:* (a) ein **all-exact-grams**-Rezept berechnet pro Portion + per 100 g
  gegen **handgerechnete Erwartungswerte** (nicht die eigenen Konstanten des Converters —
  vermeidet die Tautologie des früheren ±5%-Tests) mit `macrosEstimated:false`; (b) ein
  **Stück/Löffel**-Rezept berechnet ein volles `totalWeightG` **und setzt
  `macrosEstimated:true`**; (c) ein Rezept mit einer ungepickten/geflaggten Zutat meldet sie und
  zeigt per 100 g als „—".
- **T4.2b** `POST /api/macros/compute` (setzt **`macroSource: "ingredients"`** bei einem
  Compute — 4th-review minor, damit das T2.1-Badge zuverlässig ist) + die
  Frontend-**Suche-und-Auswahl-UI** (pro Zutat: Suchbox → Kandidatenliste — **Produkte mit
  vollständigen Nährwerten / einem Nutrition-Grade bevorzugend** — → Auswahl → Makros füllen;
  ins M2-Formular verdrahtet). *Verify:* im Formular listet die Suche nach einer Zutat
  (gemockt/live OFF) Kandidaten, eine Auswahl füllt ihre Makros, pro Portion + per 100 g rechnen
  korrekt, wenn alle Zutaten gewählt und gramm-aufgelöst sind, **und das Ergebnis trägt
  `macroSource:"ingredients"`** (Backend-Compute-Unit-Test + ein Formular-Component-Test für den
  Pick-Flow).
- **T4.3** `POST /api/macros/estimate` + „Estimate with local LLM"-Button →
  `LlmMacroEstimator` (nutzt `LlmClient` auf dem **`IHttpClient`-Seam** wieder; schema-validiert;
  liefert auch ein geschätztes Gesamtgewicht), füllt Makro-Felder zur Nutzer-Prüfung **und setzt
  `macroSource: "llm"` + `macrosEstimated: true`** (LLM-Ausgabe ist eine Schätzung — 3rd-review
  Major; 4th-review minor: es muss `macroSource` setzen, damit das T2.1-Badge zuverlässig ist).
  *Verify:* Unit-Test mit dem **Fake-`IHttpClient`** (kein Live-LLM) füllt Makro-Felder **und
  prüft, dass das gefüllte Rezept `macroSource:"llm"` + `macrosEstimated: true` trägt**; der
  Nutzer kann vor dem Speichern noch überschreiben (was `macroSource` → `manual` kippt).

## Milestone 5 — Medien, Politur, Suche
- **T5.1** Bild-Upload-Endpoint — **diese Task ergänzt einen
  `multipart/form-data`-Parser in die `net`-Lib** (aus M0 verschoben, hier zuerst gebraucht) →
  Disk-Volume + externe-URL-Option (**nur-speichern, kein serverseitiger Fetch — M6b/SSRF**).
  Konkrete Validierung gemäß D6/M6: **server-generierter Dateiname** (UUID + Extension,
  Client-Dateiname ignoriert — kein Path-Traversal), **Größe ≤ 8 MB *während* des Streamings
  erzwungen** (abweisen, wenn die Bytes ankommen, nicht nach dem Puffern des ganzen Parts —
  sonst OOMt ein großer Upload vor dem Check; 5th-review minor, spiegelt den T0.2-Body-Cap),
  **Content-Type ∈ {jpeg,png,webp} per Magic-Bytes verifiziert**. **Dateien sind
  lifecycle-verwaltet (3rd-review minor #6):** der Uploads-Service **besitzt** die
  UUID-benannten Dateien, die er geschrieben hat, und unlinkt jede eigene Datei, die keine
  `recipe_images`-Zeile mehr referenziert — bei Rezept-Löschung und beim PUT-Bild-Diff (gemäß
  der T1.3-Reihenfolge); externe-URL-Bilder besitzen keine Datei. *Verify:* ein valides Bild
  hochladen hängt es an und rendert; **übergroß, falscher Typ und ein gebauter
  Path-Traversal-Dateiname werden alle abgewiesen**; eine externe Bild-URL wird gespeichert und
  **nie vom Backend gefetcht** (API-Test prüft, dass kein ausgehender Request erfolgt); **das
  Löschen eines Rezepts (und das Ersetzen eines Bildes via PUT) entfernt die entsprechende
  eigene Datei vom Uploads-Volume, und ein externe-URL-Bild wird nie angefasst**.
- **T5.2** Browse-**Suche/Filter/Sort** über `GET /api/recipes`-Query-Params + SQL (bei
  GATE 0 festgelegt), auf die **paginierte Summary-Liste** gelegt (`limit`/`offset`, D1):
  **Titel-Text**-Suche — **einfache Case-Insensitivity** via **`ILIKE` unter einem
  UTF-8-`lc_ctype`** (`Ä`↔`ä`, `Huhn`↔`huhn`); **keine Accent-Faltung** (`Hahnchen`↔`Hähnchen`
  sind NICHT gleich) und **keine Extension** — braucht keine gepinnte PG-Version (4th-review
  Major #6; entschieden case-insensitive-only. `citext` wird ausdrücklich abgelehnt — es
  `lower()`-faltet nur, strippt keine Accents; Accent-Faltung via `unaccent` ist eine verschobene
  spätere Option). **`ILIKE`s `Ä`↔`ä`-Faltung hängt von einem UTF-8-DB-Locale ab (nicht
  C/POSIX) — der Postgres-Container wird mit einem initialisiert (T6.1); 5th-review minor.**
  Substring auf `title`, optional `description`; **Tag-Filter** (Multi-Select, **AND**-Semantik);
  ein **Nur-Favoriten**-Toggle (`favorite = true`); **Makro-Filter** `minProtein` + `maxCalories`
  auf den Makro-Spalten pro Portion; und **Sort** nach neueste (`createdAt` desc, Default), Titel
  A–Z oder höchstes Protein. _(Volltextsuche über Zutaten/Steps ist auf eine spätere Phase
  verschoben — braucht Postgres-FTS.)_ *Verify:* Such-Tests liefern die erwartete Teilmenge aus
  geseedeten Daten für eine Titel-Query (**ein Case-Insensitivity-Fall, `Ä`↔`ä`; und eine
  Kontrolle, die prüft, dass Accent-Faltung NICHT angewandt wird**), einen Tag-AND-Filter, den
  Favoriten-Toggle und eine `minProtein`/`maxCalories`-Range; jede Sort-Reihenfolge bestätigen
  **und dass `limit`/`offset` paginieren — und dass ein `limit` über dem Max geklammert wird,
  nicht wörtlich übernommen** (3rd-review minor #5).

## Milestone 6 — Deployment & Docs
- **T6.1** Mehrstufiges **Dockerfile** fürs C++-Backend (Build → schlankes Runtime), auf
  einer **per-Digest gepinnten Base (`FROM …@sha256:…`, nicht ein veränderlicher Tag —
  5th-review T1)**; der **Postgres-Dienst wird mit einem UTF-8-Locale initialisiert**
  (`POSTGRES_INITDB_ARGS=--locale`/`LANG=…utf8`), damit die T5.2-`ILIKE`-`Ä`↔`ä`-Faltung
  garantiert ist, nicht dem Default überlassen (5th-review minor); die Build-Stufe macht
  **`apt install` von `libpq-dev`/`libssl-dev`/`libutf8proc-dev`/`catch2`** (vorgebaut,
  schnell). Das **schlanke Runtime-Image MUSS die Runtime-Libs (`libpq5`, `libssl`,
  `libutf8proc`) und `ca-certificates` enthalten** — unser eigener OpenSSL-HTTPS-Client (T0.8)
  verifiziert das OFF-Cert gegen den System-Trust-Store, also scheitert ohne CA-Certs (oder die
  Runtime-Libs) der OFF-Pfad beim Deploy, obwohl er in dev bestand. + Angular-`ng
  build`-Static-Bundle, von nginx ausgeliefert mit einem **`location /api/ { proxy_pass →
  backend }`**-Block, einem **`location /uploads/`**-Block **und einem `location /health` (oder
  `/api/health` exponieren)**, damit der T6.2-Smoke-Test erreichbar ist (4th-review Major #5 —
  nginx ist die einzige öffentliche Fläche; ein bloßes `/health` würde 404en) +
  `docker-compose.yml` (Backend, Frontend, Postgres-Volume **und ein benanntes
  `uploads`-Volume, in BEIDE Container gemountet — Backend (schreibt) und Frontend/nginx
  (liefert)** — 4th-review Major #4: ohne ein geteiltes Volume 404t nginx backend-geschriebene
  Dateien und Uploads gehen bei Recreate verloren; `LLM_BASE_URL` → externes Ollama) +
  `.env.example`. *Verify:* `docker compose up` baut und liefert die App aus; Browse funktioniert
  gegen ein persistiertes Postgres-Volume; `/api/*`, `/uploads/*` und **`/health`** sind über
  nginx erreichbar; **ein vom Backend geschriebenes hochgeladenes Bild wird von nginx
  ausgeliefert** (geteiltes Volume funktioniert); **eine OFF-Suche aus dem laufenden
  Backend-Container gelingt (CA-Trust funktioniert)**.
- **T6.2** `docs/`-Nutzung + Config (Env-Vars, auf Ollama zeigen, Postgres-Backup,
  C++-Build-Notizen **inkl. der exakten `apt install`-Paketliste (libpq, OpenSSL, utf8proc,
  Catch2) — unter Nutzung der echten, versions-spezifischen Paketnamen des gepinnten
  Base-Images** (z. B. `libssl3`, die `libutf8proc`/Catch2-Pakete des Releases), nicht der
  generischen Platzhalter — und das **per-Digest gepinnte Base-Image** (T1),
  Angular-Build-Notizen **und der minimale Build-RAM** — Q8). Beachten, dass **der mehrstufige
  Dockerfile-Build die Source of Truth fürs ausgelieferte Artefakt ist** — WSL-dev betrifft nur
  „läuft in dev, bricht im Image"-Überraschungen (5th-review minor). **Den exakten
  Ollama-Pull-Tag verifizieren** für das dokumentierte `LLM_MODEL` (`ollama list`) und einen
  auflösbaren Tag in `.env.example` legen. *Verify:* eine **konkrete copy-paste-bare Sequenz**
  aus den Docs gelingt: `docker compose up` → `curl …/health` liefert 200 → ein Beispiel-Rezept
  `POST`en → es erscheint in `GET /api/recipes`.

---

## Offene Fragen an den Menschen (GATE 0)
_Diese Session gelöst (alle festgelegt): `Recipe`-Feldliste; Browse-/Such-Scope (T5.2);
Paste-Parser-Tiefe + Quell-Scope (D4 — nur Social-Captions); LLM-Modell-Politik +
keine-Übersetzung (D3); Milestone-Reihenfolge (Deploy bleibt M6); **per-100-g-Lücke (M2) →
eine Stück-Gewichts-/Löffel-Volumen-Tabelle ergänzen, damit sie für echte Rezepte auflöst**.
Verbleibende Sign-off-Fragen:_
1. **Auth-Seam-Default:** OK, das auth-ready Schema (users-Stub + nullable `owner_id`) jetzt
   mit **noch nicht implementierter Auth** aufzunehmen, gemäß D2/D6? **→ APPROVED 2026-08-24:
   ja.**
2. **Scope:** alle **sieben Milestones (M0 Core-Bibliotheken + M1–M6)** in dieser Phase?
   _(5th-review P2.)_ **→ APPROVED 2026-08-24: ja, alle sieben.**
3. **D1 einfrieren?** _(5th-review P4.)_ **→ APPROVED 2026-08-24: ja, D1 ist EINGEFROREN** —
   keine weiteren Framework-/Package-Manager-/From-scratch-Scope-Wenden. Der **TLS-Client
   bleibt von Hand über OpenSSL** (mit dem T0.8-S1-Hostname-Verify-Fix); der
   `IHttpClient`-Seam hält einen Library-Tausch günstig, falls je nötig.
_Alle drei Sign-off-Fragen sind BEANTWORTET; GATE 0 ist GESCHLOSSEN (`status: approved`)._

_Stack: Angular-SPA + **C++-Backend from scratch (kein Framework — eigener HTTP-Server/Router/
JSON/Schema/DB-über-libpq/HTTP-Client; Externals = libpq, OpenSSL, Catch2)** + PostgreSQL._

## Reviewer notes
_(newest round first)_

**Post-approval human-directed revision — build tool + M0 sequencing (2026-08-25)**
After GATE 0, an experienced-C++ friend reviewed the ROADMAP and made two points the human
chose to adopt. **Neither touches the frozen D1 architecture** (no framework, package-manager,
or from-scratch-scope change), so the standing GATE 0 approval is not reopened by them — they
are a build-tooling swap and an intra-M0 task reorder:
1. **CMake → plain GNU Makefile.** The build is Linux-only (WSL2 dev, Linux deploy — D6), so
   CMake's cross-platform payoff buys nothing while adding ceremony; a hand-written Makefile is
   simpler and keeps compile/link visible (suits the learn-the-fundamentals goal). Updated: D1
   (layout comment + `pkg-config` line), D2 (Makefile locates libs), T0.1 (Makefile + `make`/
   `make asan`, with the rationale). CMake stays a clean later switch if a Windows target is
   ever added.
2. **Thread pool moved to LAST in M0.** The friend's advice — "hang SSL and multithreading on
   the end if you're really hand-rolling it, else it's too much." SSL was already last (T0.8);
   the thread pool was front-loaded in T0.2. Now the server runs **single-threaded through
   T0.8** and the pool becomes **new task T0.9** (right before the M0 review), built once the
   rest works — de-risking the TSan/race learning curve. Updated: D1 + D2 (single-threaded
   first, pool at T0.9), T0.2 (thread pool removed), **new T0.9** (pool + concurrent dispatch,
   worker-owns-a-db-connection, TSan-verified), T0.1 (TSan now covers the T0.9 pool + the T0.6
   db pool). T0.1–T0.8 numbers unchanged.
_The friend also suggested dropping Docker; the human **deferred** that to M6 (dev Postgres
stays in Docker for now; the deploy-Docker question is revisited when M6 is reached) — no plan
change made for it yet._

**5th external review — consolidated, targets the from-scratch M0 (2026-08-24)**
(`docs/reviews/PLAN_REVIEW_2026-08-24_external-5-consolidated.md`): confirmed **all 4th-review
fixes genuinely resolved** and the plan body **clean of old-stack assumptions**; verdict "close
but not ready to sign — blockers in the hand-rolled networking + doc/process." All folded:
- _🔴 S1 TLS verifies only the chain, not the HOSTNAME (`SSL_VERIFY_PEER`+SNI ≠ hostname check)
  → silent MITM on OFF_ → *Fixed*: T0.8 pins **`SSL_set1_host` + `SSL_get_verify_result==X509_V_OK`**;
  verify uses a **valid-CA-wrong-hostname** host (`wrong.host.badssl.com`).
- _🔴 S2 no memory-safety tooling for 4 hand-rolled untrusted-input parsers_ → *Fixed*: T0.1
  adds **ASAN+UBSAN (+TSan for the pool)**; T0.2 + T0.4 get **libFuzzer/AFL++ harnesses**.
- _🟠 P1 ROADMAP wholesale stale (Drogon/vcpkg/6-milestone, falsely says GATE 0 passed)_ →
  *Fixed*: `docs/ROADMAP.md` rewritten to the current plan (M0, apt, seven milestones, GATE 0
  pending).
- _🟠 P2 "six milestones" but there are seven (M0+M1–M6)_ → *Fixed*: Goal + scope note +
  sign-off Q2.
- _🟠 P3 T0.5/T0.6/T0.7 depended on M1 artifacts_ → *Fixed*: compose Postgres moved to T0.1;
  T0.5 tests a generic fixture schema (real schema in T1.2).
- _🟠 P4 freeze D1 + make this 5th review a precondition_ → sign-off **Q3 (freeze D1)** added.
- _🟠 T1 reproducibility overclaimed_ → *Fixed*: **digest-pin** the base image (D1/T6.1/T6.2).
- _🟠 T2 HTTP request-smuggling framing_ → *Fixed*: T0.2 adds RFC 7230 §3.3.3 rules
  (CL+TE, dup CL, bare CR/LF/NUL, overflow) + test vectors.
- _🟠 T3 `nullable` isn't a JSON-Schema keyword; surrogate pairs; recursion cap_ → *Fixed*:
  D2/T0.5 use `type:[…,"null"]`; T0.4 combines surrogate pairs + depth cap.
- _minors_ → ILIKE UTF-8 locale (T6.1/T5.2); `PQexec` migration-only path (T0.6); multipart
  streaming cap (T5.1); `/jsonschema` in layout; **C++20** pinned (T0.1); ingredient `note`
  render (T2.1); WSL-vs-Docker parity note (T6.2).
- _open decision surfaced to the human_ → the review questions whether the **hand-rolled TLS
  client** specifically is worth it (poor learning-signal-per-risk); left for the human at
  sign-off (default: keep from-scratch + the S1 fix).
**Confirm pass:** reviewer verified all ten change-sets **present, correct, and consistent**
(S1 TLS hostname verify sound; S2 sanitizers/fuzzing; ROADMAP clean of old stack; seven
milestones; M0 self-contained; digest pin; anti-smuggling; `type:[…,"null"]`; minors) and
M0→M6 ordering intact. One blocking finding — the freshly-rewritten ROADMAP claimed reviewer
approval before this pass granted it — resolved by stamping `approvals.reviewer: 2026-08-24`
now (the pass's intended output), so ROADMAP + frontmatter agree (reviewer-approved,
human-pending). One non-blocking (a D3 "M5" mislabel → "T3.2/M3, T4.3/M4") also fixed. **Human
GATE 0 signature is the only remaining step; do NOT stamp `approvals.human` until the user
signs.**

**4th external review — consolidated (2026-08-24)** (`docs/reviews/PLAN_REVIEW_2026-08-24_external-4-consolidated.md`):
verdict "ready at M1 today; one corrective pass before M2." Ran against the **pre-pivot** plan
(cites Drogon/vcpkg/RE2/valijson), so some findings were stale; the **live** ones (unaffected by
the from-scratch/vcpkg changes) were all folded in:
- _🔴 tags/favorite/notes/times had no write path (T2.2 omitted them → T5.2 filters dead)_ →
  *Fixed*: added to T2.1 render + T2.2 form (+ round-trip/filter verify).
- _🔴 PUT recompute of `macrosEstimated` fought the "manual edit clears it" rule_ → *Fixed*:
  T1.3 suppresses recompute when `macroSource=="manual"`.
- _🟠 NFC had no mechanism (worse post-pivot: no libs)_ → **Decided (human): add `utf8proc`**
  (apt) — the one external for the parser; D1/D2/T0.1/T3.1/T6.1/T6.2 updated.
- _🟠 uploads not a shared volume_ → *Fixed*: T6.1 declares a named `uploads` volume in both
  backend + nginx.
- _🟠 `/health` unreachable through nginx_ → *Fixed*: T6.1 adds `location /health`.
- _🟠 ß/umlaut search wrong (citext/collation/PG-version)_ → **Decided (human): plain
  case-insensitive `ILIKE`** (no extension, any PG version); citext rejected; accent-folding
  deferred. T5.2 updated.
- _minors_ → PUT URL `:id` (T2.2), `macroSource` set by T4.2b(`ingredients`)/T4.3(`llm`),
  `/api/parse` returns a partial draft not `422` (T3.1), ROADMAP T2.0 reconciled into the plan,
  migration-concurrency marked "kept for future scaling" (D2). Handoff §3 + example-recipes
  fixture note fixed in their own files.
- _stale (already resolved by the pivot; noted, not acted)_ → NFC-lib-in-vcpkg, valijson
  `$schema`, Drogon `DbClient connectionNumber=1`, Drogon/vcpkg first-build risk.
**Confirm pass: APPROVE** — all seven corrective edits verified present, correct, and
consistent; M0→M6 ordering holds; 1 non-blocking folded (badge omitted when macros unset).
Reviewer signature re-stamped `2026-08-24`; **human GATE 0 signature is the only remaining step.**

**Drop vcpkg (2026-08-24)** — after the from-scratch pivot was reviewer-approved, the human
chose to **drop vcpkg too and use system packages (`apt`)** for the three externals
(libpq/OpenSSL/Catch2). A build-system change (not architecture): reviewer stamp reset again.
Edits: D1 (externals via apt, no package manager; reproducibility now rests on pinned
base-image/distro versions + a documented apt list), D2 (dependencies-via-apt note, CMake
`find_package`/`pkg-config`), Q8 (build cost now minimal — prebuilt binaries, nothing compiled
from source), T0.1 (CMake finds apt packages), T6.1 (pinned base image + `apt install` in the
build stage + runtime libs `libpq5`/`libssl` and `ca-certificates`), T6.2 (document the apt
list + base tag). **Knowing trade recorded:** loses vcpkg's checked-in version pinning; the
human accepted this. **Reviewer confirm: APPROVE** — no active-body vcpkg references remain,
reproducibility (pinned base image + apt list) coherent, runtime image ships `libpq5`/`libssl`/
`ca-certificates`; 1 non-blocking (use the base image's real version-specific apt names in T6.2)
folded into T6.2. Reviewer signature re-stamped `2026-08-24`; **human GATE 0 signature is the
only remaining step.**

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
