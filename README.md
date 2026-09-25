# Waypoint Relay

Delivery planning and tracking system for Waypoint Group, built by team **LegendX** for Rootcode Tech-Triathlon 2026.

One shared order-to-delivery flow for four roles: Store Manager, Dispatcher, Loader and Driver.

## Tech stack

| Layer | Choice |
|---|---|
| Frontend | React + Vite + TypeScript (PWA), Tailwind, shadcn/ui, TanStack Query, Dexie |
| Backend | NestJS (modular monolith), Prisma |
| Database | PostgreSQL 16 |
| File storage | MinIO |
| Gateway | Caddy |
| Deployment | Docker Compose on Google Cloud, Cloudflare Tunnel |
| CI/CD | GitHub Actions |

## Repository layout

```
apps/
  web/          React PWA
  api/          NestJS API
packages/
  shared/       Shared types, Zod schemas, constraint validator
infra/          Caddy, deployment and compose configs
docs/           Architecture, data model, AI disclosure
```

## Getting started

Setup steps will be added once the apps are scaffolded. The target is:

```bash
cp .env.example .env
docker compose up
```

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md) before your first commit.

## Team LegendX

- Aditha Buwaneka (Team Leader)
