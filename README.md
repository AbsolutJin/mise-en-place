# mise-en-place

Eine selbst-gehostete Web-App für eine Person, um Rezepte zu speichern, zu
durchstöbern und anzulegen.

Rezepte kommen auf zwei Wegen herein: über ein **strukturiertes Formular** oder
durch das Einfügen einer Freitext-Caption (TikTok, Instagram), die in ein
einheitliches JSON-Rezeptformat normalisiert wird. Jedes Rezept zeigt optionale
Bilder, einen Quell-Link und Makros pro Portion und pro 100 g. Die Makros werden
aus [Open Food Facts](https://world.openfoodfacts.org/) berechnet — pro Zutat
ein Lebensmittel suchen und auswählen, lokal gecacht — mit einem Button für eine
Schätzung per lokalem LLM und einer jederzeit editierbaren manuellen Korrektur.

## Warum das Repo so aussieht

Das Backend ist in C++ von Grund auf geschrieben, ohne Web-Framework: ein eigener
HTTP/1.1-Server, Router, JSON-Parser samt Schema-Validator, eine PostgreSQL-Schicht
über libpq und ein HTTPS-Client über OpenSSL. Das ist Absicht. Der Zweck des
Projekts ist, C++ und Angular durch den Bau der Grundlagen zu lernen, nicht
möglichst schnell zu shippen. Der Preis dafür ist viel zusätzliche
Plumbing-Arbeit — ein bewusst gewählter Trade-off (siehe D1 im
[Plan](docs/plans/PLAN_recipe_app_foundation.md)).

## Stack

| Ebene | Wahl |
|---|---|
| Backend | C++20, from scratch (eigener HTTP-Server, Router, JSON, Schema-Validator, DB über libpq, HTTPS-Client); nur die unvermeidbaren Externals libpq, OpenSSL, utf8proc und Catch2 über das System-`apt`, kein Package-Manager |
| Frontend | Angular-SPA (Reactive Forms) |
| Datenbank | PostgreSQL |
| Nutrition | Open Food Facts (suchen und auswählen, gecacht) plus optional ein lokales LLM (Ollama) |
| Deployment | Docker Compose (Backend, nginx-Frontend, Postgres); läuft auf einem Home-Server oder VPS hinter VPN oder Basic-Auth über TLS |
| Dev | WSL2 (Ubuntu) |

Begründung und Detail der Stack-Entscheidungen stehen im
[Plan](docs/plans/PLAN_recipe_app_foundation.md).

## Status

Foundation-Phase: die Planung ist abgeschlossen, die Implementierung hat noch
nicht begonnen. Es existiert noch kein App-Code. Den aktuellen Fortschritt führt
die [ROADMAP](docs/ROADMAP.md); die Spezifikation steht im
[Plan](docs/plans/PLAN_recipe_app_foundation.md).

## Repository-Aufbau

```
/backend    C++-API from scratch (Milestone 0 baut zuerst die Core-Bibliotheken)
/frontend   Angular-SPA
/docs       Plan, Roadmap, Reviews, Referenz, Handoffs
docker-compose.yml
```

In dieser Phase läuft die App für eine Person und ohne Authentifizierung —
gedacht hinter einem VPN oder einem Reverse-Proxy mit Basic-Auth über TLS.
