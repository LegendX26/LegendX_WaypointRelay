# Contributing

## Branches

| Branch | Purpose | Created from | Merges into |
|---|---|---|---|
| `main` | Production. Deployed by tagging `v*` | - | - |
| `dev` | Integration branch, always runnable | `main` | `main` |
| `feature/<name>` | New feature | `dev` | `dev` |
| `bugfix/<name>` | Bug fix found during development | `dev` | `dev` |
| `hotfix/<name>` | Urgent fix on production | `main` | `main` and `dev` |

Use short lowercase names with hyphens, e.g. `feature/order-cutoff`, `bugfix/sync-duplicate`.

## Workflow

1. Pull the latest `dev`: `git checkout dev && git pull`
2. Create your branch: `git checkout -b feature/<name>`
3. Commit small and often
4. Push and open a Pull Request into `dev`
5. At least one teammate reviews before merging
6. Delete the branch after merge

Never push directly to `main` or `dev`.

## Commit messages

Format: `type(scope): short description`

| Type | Use for |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `refactor` | Code change without behaviour change |
| `test` | Tests |
| `chore` | Tooling, config, dependencies |

Examples:

```
feat(orders): block orders after 4 PM cutoff
fix(sync): ignore duplicate delivery uploads
docs: add data model diagram
```

## Before opening a PR

- Code builds and lint passes
- No `.env` files, secrets or raw datasets committed
- PR description says what changed and how it was tested
