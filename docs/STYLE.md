# Schreibstil für die menschen-lesbaren Docs

Wie wir die Dokumente schreiben, die ein Mensch wirklich liest: die README, die
ROADMAP und den Plan. Ziel ist Prosa, die ehrlich, ruhig und gut überfliegbar
ist — nicht dekoriert, nicht selbstlobend, nicht viermal dasselbe über mehrere
Dateien verteilt.

## Was das hier regelt

**Im Scope** — die menschen-lesbaren Docs, auf **Deutsch**:

- `README.md`
- `docs/ROADMAP.md`
- `docs/plans/PLAN_recipe_app_foundation.md`

**Außerhalb** — Docs, die für die Maschine geschrieben sind, bleiben wie sie sind
(und **auf Englisch**):

- `docs/handoff/*` — Kontext von Session zu Session, für Claude optimiert
- `docs/reviews/*` — historische Review-Records, authentisch belassen

## Die eine Regel, die den Inhalt schützt

Wir ändern **Form und Struktur, nie die Substanz.** Jeder technische Fakt bleibt
erhalten: die Security-Invarianten, das „Done when" jeder Aufgabe, die
eingefrorene D1-Architektur, die Design-Entscheidungen, der Approval-Record. Wenn
eine Änderung ändern würde, was ein Satz *behauptet*, ist sie hier tabu.

Der Plan ist ein Sonderfall: ein freigegebenes, gegatetes Artefakt mit
eingefrorenem Design (D1). Wir formulieren seine Prosa lesbarer, aber seine
Entscheidungen bleiben bedeutungstreu. Was ein Re-Review auslösen würde, gehört
nicht in einen Lesbarkeits-Durchgang.

## Sprache

Die Living-Docs sind **Deutsch**. Etablierte englische **Fachbegriffe bleiben
englisch** — thread pool, hostname verification, chunked, keep-alive,
`Content-Length`, Router, Handler, Migration. Kein Zwangs-Eindeutschen; ein
deutscher Kunstbegriff, den niemand sagt, ist schlechter als das englische
Original. Diese Style-Datei ist ebenfalls Deutsch, damit sie zu dem Text passt,
den sie regelt.

## Prinzipien

1. **Ein Fakt, ein Ort.** Jeder Fakt hat einen kanonischen Platz; andere Docs
   verlinken darauf, statt ihn zu kopieren. Der Stack, der Status und „D1 ist
   eingefroren" leben in *einem* Doc und werden aus den anderen referenziert —
   so können sie nicht auseinanderdriften. Konkret für dieses Repo:
   - **PLAN** — kanonisch für Stack und Design-Entscheidungen.
   - **ROADMAP** — reiner Fortschritt; verlinkt für Details in den PLAN.
   - **README** — kurze Einladung; verlinkt in PLAN und ROADMAP.
2. **Betonung ist Geld, gib sie sparsam aus.** **Fett** ist für eine echte
   Warnung oder Invariante, ein paar Mal pro Seite — nicht für Stimmung. Wenn der
   halbe Absatz fett ist, wirkt nichts davon wichtig.
3. **Keine Emoji in Prosa oder Überschriften.** Der Checklisten-Status (`[ ]` /
   `[~]` / `[x]`) trägt die Info schon; ✅ ⚪ 🚦 📋 fügen nichts hinzu, was ein
   Wort nicht kann.
4. **Ein Gedanke pro Satz.** Löse die Ketten aus Gedankenstrichen und Klammern
   auf. Wenn ein Einschub selbst einen Einschub braucht, will er ein eigener Satz
   sein.
5. **Metadaten haben einen Platz, und der ist nicht mitten im Satz.**
   Approval-Daten, Commit-Hashes, Branch-Namen gehören ins Frontmatter oder in
   eine kurze „Historie"-Notiz — nicht in die erzählende Prosa.
6. **Neutrale Stimme.** Sag, was ist, nicht wie gut es gemacht wurde. Weg mit
   „ehrlich darüber", „entschärft die Lernkurve", „ohne Kontext neu herzuleiten".
   Keine Marketing-Adjektive.
7. **Einmal sagen, dann aufhören.** Streich die Wiederholung, die einem klaren
   Satz folgt. Eine „in einem Satz"-Zusammenfassung mit fünf weiteren Zeilen war
   kein Satz.

## Vorher → Nachher

Echte Zeilen aus unseren Docs, umgeschrieben nach den Regeln oben (und dabei ins
Deutsche übertragen).

**Betonungs-Inflation**

> Vorher: GATE 0 passed, plan **approved and revised** (2026-08-25). **No app
> code yet.** We are **setting up the WSL2 dev environment** so implementation of
> **Milestone 0 / T0.1** can begin.

> Nachher: GATE 0 ist bestanden; der Plan ist freigegeben und überarbeitet
> (2026-08-25). Es gibt noch keinen App-Code — wir richten die WSL2-Dev-Umgebung
> ein, bevor Milestone 0 (T0.1) startet.

**Gedankenstrich-/Klammer-Kette**

> Vorher: Thread pool moved to LAST in M0. The `net` HTTP server is now
> **single-threaded through T0.8**; the thread pool became **new task T0.9**
> (right before the M0 review). SSL was already last (T0.8). De-risks the
> TSan/race learning curve. **T0.1–T0.8 numbers unchanged.**

> Nachher: Der thread pool rückt ans Ende von M0, als Aufgabe T0.9 direkt vor dem
> M0-Review. Der `net`-Server bleibt bis T0.8 single-threaded. Die Nummern T0.1
> bis T0.8 ändern sich nicht.

**Metadaten in der Prosa**

> Vorher: plan **approved** (`approvals.reviewer` + `approvals.human` both
> `2026-08-24`, `status: approved`).

> Nachher: der Plan ist freigegeben (die Approval-Daten stehen im Frontmatter des
> Plans).

**Emoji-Status**

> Vorher: ## Milestone 0 — Core backend libraries (from scratch) ⚪

> Nachher: ## Milestone 0 — Core-Backend-Bibliotheken (from scratch)
