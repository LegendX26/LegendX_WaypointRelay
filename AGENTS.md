# AGENTS.md

Instructions for coding agents working in this repository. Humans: the same rules are in [CONTRIBUTING.md](CONTRIBUTING.md) and [docs/](docs/).

## Project

Waypoint Relay: a delivery planning system for Waypoint Group, with four roles (store manager, dispatcher, loader, driver). One responsive React PWA (`apps/web`), one NestJS API (`apps/api`), PostgreSQL 16 with Prisma, shared rules in `packages/shared`, allocation in `packages/engine`.

## Read before changing anything

1. [docs/design-deviations.md](docs/design-deviations.md): the Designathon baseline. The Figma design is the specification.
2. [docs/backend-spec.md](docs/backend-spec.md): data model, business rules, API, offline sync, database practices.
3. [docs/data-model.md](docs/data-model.md) and [docs/architecture.md](docs/architecture.md).
4. [CONTRIBUTING.md](CONTRIBUTING.md): branches and commit format.

## Git

- Create branches from `dev`: `feature/<name>` or `bugfix/<name>` (`hotfix/<name>` from `main` only for production fixes). Never commit to `main` or `dev` directly.
- Commit format: `type(scope): short description` (`feat`, `fix`, `docs`, `refactor`, `test`, `chore`). Small commits.
- **Never mention AI tool or model names** in commit messages, PR titles or descriptions, or review comments. **Never add `Co-Authored-By` lines for AI tools.**

## Code

- TypeScript everywhere. Keep code simple and consistent with the surrounding code.
- **Keep comments short and few.** Use clear names; comment only where the logic isn't obvious. No long explanatory comment blocks.
- Business rules (the 7 feasibility rules, trip time, fuel) live only in `packages/shared`. Import them; never copy them into a module or into SQL.
- No database triggers, stored procedures or functions. Constraints (FK, `UNIQUE`, `CHECK`) are fine.
- Writes that touch more than one table run in one transaction. Offline sync is idempotent on `client_uuid`. Editable rows use a `version` column.
- Every endpoint checks the role **and** the scope (own outlet, own trip, own depot).
- Times are `timestamptz` in Asia/Colombo.

## Don't

- Don't call an LLM or any external AI API from the app. Dispatcher suggestions are rule-based (backend-spec Section 3.8).
- Don't add features the design left out: admin panel, sign-up, password reset, profile settings, forecast screen, live GPS map, chat, Sinhala/Tamil, product-level order lines, invoices.
- Don't add Redis, message queues, microservices, GraphQL or WebSockets without a team decision.
- Don't invent data (items, prices, names). Use the shared datasets and the seeded S1 day.
- Don't commit datasets, `.env` files or secrets. Only `seed/**/*.csv` may be committed.

## When you change something

- Anything that differs from the Figma design → add a row to `docs/design-deviations.md`.
- Schema change → update `docs/data-model.md`. Architecture change → update `docs/architecture.md`.
- **AI disclosure:** the one place AI tools are named is `docs/ai-disclosure.md`. When AI helped with a change, note there what was used and for what.

## Before opening a PR

- Lint, typecheck and tests pass; `docker compose up` still starts the full stack with seed data.
- Once the engine exists: the seeded plan still passes `check_allocation.py`.
- The PR description says what changed and how it was tested.
