# mise-en-place

A **single-user, self-hosted web app** to store, browse, and add recipes.

Recipes can be added two ways: through a **structured form**, or by **pasting a free-text
caption** (TikTok / Instagram) that gets normalised into one **canonical JSON recipe format**.
Each recipe renders with optional images and a source link, and shows **macros per portion and
per 100 g** — computed from [Open Food Facts](https://world.openfoodfacts.org/) (search and
pick a food per ingredient, cached locally), with a **local-LLM estimate** button and an
always-editable manual override.

## Why this repo looks the way it does

The backend is written **in C++ from scratch — no web framework** (its own HTTP/1.1 server,
router, JSON parser + schema validator, PostgreSQL layer over libpq, and HTTPS client over
OpenSSL). That is a deliberate choice: the point of the project is to **learn C++ (and Angular)
by building the fundamentals**, not to ship the fastest way. It trades a lot of extra plumbing
for that learning, and the plan is honest about it.

## Stack

| Layer | Choice |
|---|---|
| Backend | **C++20, from scratch** (own HTTP server / router / JSON / schema validator / DB-over-libpq / HTTPS client); only irreducible externals — libpq, OpenSSL, utf8proc, Catch2 — via system `apt` (no package manager) |
| Frontend | **Angular** SPA (Reactive Forms) |
| Database | **PostgreSQL** |
| Nutrition | **Open Food Facts** (search-and-pick, cached) + optional local LLM (Ollama) |
| Deployment | **Docker Compose** (backend + nginx-served frontend + Postgres); runs on a home server / VPS behind a VPN or basic-auth over TLS |
| Dev | **WSL2 (Ubuntu)** |

## Status

**Foundation phase — planning complete, implementation not yet started.** The design is fully
specified and reviewed; no application code exists yet. Development follows a staged,
plan-gated workflow.

- 📋 **Plan:** [`docs/plans/PLAN_recipe_app_foundation.md`](docs/plans/PLAN_recipe_app_foundation.md) — approved
- ✅ **Progress:** [`docs/ROADMAP.md`](docs/ROADMAP.md) — checkable task tracker (Milestone 0 → M6)
- 📝 **Reviews:** [`docs/reviews/`](docs/reviews/) — the plan's review history

## Repository layout

```
/backend    C++ from-scratch API (Milestone 0 builds the core libraries first)
/frontend   Angular SPA
/docs       plan, roadmap, reviews, reference, handoff
docker-compose.yml
```

_Single-user and unauthenticated this phase — intended to run behind a VPN or a reverse proxy
with basic-auth over TLS._
