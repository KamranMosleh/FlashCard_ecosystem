# Newcomer Guide: FlashCard Ecosystem

This guide gives a practical orientation to the repository so new contributors can get productive quickly.

## 1) What this repo is

The project is an Italian-learning Telegram product with three core moving parts:

1. **Main bot service** (`flashcard-project/`) — FastAPI + aiogram app that handles Telegram updates, creates cards with Gemini, schedules spaced-repetition reviews, and persists data in MongoDB.
2. **Conjugation scraper microservice** (`WR_scraper/`) — FastAPI service that scrapes WordReference conjugations and returns structured JSON.
3. **Deployment stack** (root `docker-compose.yml` + `caddy/`) — wires bot, scraper, MongoDB, and reverse proxy (Caddy) together.

## 2) Repository map

At a high level:

- `flashcard-project/`: application code, tests, and most docs.
- `WR_scraper/`: standalone scraper API.
- `docker-compose.yml`: multi-service runtime for production/local containers.
- `docs/`: top-level operational docs (currently CI/CD focused).

If you only have 30 minutes, spend almost all of it in `flashcard-project/` first.

## 3) Main app internals (`flashcard-project/src/flashcard`)

The package is intentionally layered:

- `telegram/` → input/adapters layer (handlers, FSM, keyboards, UI formatting)
- `services/` → domain logic (expression, user, verb, LLM, consumption, algorithm)
- `db/` + `schemas/` → persistence and data modeling
- `api/` → HTTP/webhook/health endpoints
- `scheduler/` → background periodic review push loop
- `utils/` + `resources/` → cross-cutting helpers and locale strings

### Key files to read first

1. `telegram/bot.py` — bootstraps bot, middleware, router order, service wiring, scheduler startup.
2. `docs/architecture.md` — best high-level walkthrough of flow + router precedence.
3. `settings.py` — complete env/config contract and webhook validation rules.
4. `__main__.py` + `pyproject.toml` scripts — how runtime modes are selected.

## 4) Request flow mental model

A useful newcomer model:

- Telegram update arrives via **polling or webhook**.
- aiogram routes to a handler (router order matters: first match wins).
- Handler calls one or more services.
- Services may call MongoDB, Gemini, or scraper API.
- Handler formats response through UI helpers and sends Telegram message.
- For reviews, scheduler runs independently and triggers outbound cards.

## 5) Important implementation constraints

- **Router ordering is critical**. Place new command routers before `unknown` and before generic text handlers.
- **Dependency injection is name-based** in aiogram polling/webhook dispatcher data.
- **One app, two delivery modes** (`polling` vs `webhook`) via env (`TELEGRAM_DELIVERY_MODE`).
- **Background scheduler is always part of app startup**, so startup/shutdown hygiene matters.
- **HTML parse mode is global** — dynamic text should be escaped where needed.

## 6) Data and domain concepts to know

Start with these entities:

- **User profile/settings** (`users` collection): language/level/preferences/scheduling state.
- **Expression cards** (`expressions` collection): source phrase + review metrics + next due state.
- **Conjugation cache** (`conjugations` collection): scraper-backed lookup cache.

Core domain operations:

- card generation (LLM)
- card save/list/import
- spaced repetition grading/update
- scheduled review candidate selection
- story generation from vocabulary

## 7) How to run and debug locally

Recommended for first run:

1. Create `.env` from root template.
2. Install app in editable mode from `flashcard-project`.
3. Run `flashcard-bot-dev-poll`.

This avoids webhook setup complexity while you inspect handlers and services.

## 8) Where to contribute first

Good first contribution areas:

- add/update one command handler + tests
- improve one service method with better error mapping/logging
- add i18n coverage for user-facing strings
- strengthen tests around scheduler/algorithm edge cases

## 9) Suggested learning path (ordered)

1. `flashcard-project/docs/architecture.md`
2. `flashcard-project/src/flashcard/telegram/bot.py`
3. `flashcard-project/src/flashcard/telegram/handlers/` for user-facing behavior
4. `flashcard-project/src/flashcard/services/` for business rules
5. `flashcard-project/tests/unit/` to see expected behavior and mocking patterns
6. `flashcard-project/docs/contributing.md` before implementing changes
7. `WR_scraper/` only after you understand verb lookup path

## 10) Practical newcomer checklist

- [ ] Run app locally in polling mode
- [ ] Trigger `/start`, create a card, save it, grade one review
- [ ] Trace one command end-to-end (handler → service → DB call)
- [ ] Add or modify one test in `tests/unit/`
- [ ] Read router order section before adding any new handler

