# Waypoint Relay

Delivery planning and tracking system for Waypoint Group, built by team **LegendX** for Rootcode Tech-Triathlon 2026.

One shared order-to-delivery flow for four roles: Store Manager, Dispatcher, Loader and Driver.

## Tech stack

| Layer | Choice |
|---|---|
| Frontend | React + Vite + TypeScript (PWA), Tailwind, shadcn/ui, TanStack Query, Dexie |
| Backend | NestJS (modular monolith), Prisma |
| Database | PostgreSQL 16 |
| Allocation | HiGHS with greedy fallback |
| File storage | MinIO |
| Gateway | Caddy |
| Deployment | Docker Compose on Google Cloud, Cloudflare Tunnel |
| CI/CD | GitHub Actions |

## Repository layout

```
apps/
  web/          React PWA (all four role views)
  api/          NestJS API
packages/
  shared/       Shared types, Zod schemas, constraint validator
  engine/       Allocation engine
seed/           Seed data loader
infra/          Caddy and deployment configs
docs/           Architecture, data model, AI disclosure
```

## Setup

Requires Docker and Docker Compose.

```bash
git clone https://github.com/LegendX26/LegendX_WaypointRelay.git
cd LegendX_WaypointRelay
cp .env.example .env
docker compose up
```

This starts the full stack, including the database and seed data.

## Configuration

All settings are in `.env`. See [.env.example](.env.example) for the list of variables.

## Deployed system

To be added.

## Seeded accounts

| Role | Username | Password |
|---|---|---|
| Store Manager | - | - |
| Dispatcher | - | - |
| Loader | - | - |
| Driver | - | - |

## Judge walkthrough

To be added.

## Departures from the design

See [docs/design-deviations.md](docs/design-deviations.md).

## Documentation

- [Architecture](docs/architecture.md)
- [Data model](docs/data-model.md)
- [Backend spec](docs/backend-spec.md)
- [AI disclosure](docs/ai-disclosure.md)

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md) before your first commit. Coding agents follow [AGENTS.md](AGENTS.md).

## Team LegendX

- Aditha Buwaneka (Team Leader)
