# Contributing to Savile

## Branching Strategy

We use a simplified **GitHub Flow** with two main branches:

| Branch | Purpose | Who merges here |
|--------|---------|-----------------|
| `main` | **Production-ready.** Only receives merges from `develop` at phase milestones. Protected — requires PR + 1 approval. | Odytom (project lead) |
| `develop` | **Integration branch.** All feature branches merge here via Pull Requests. | Any team member via PR |

## How to Work on a Feature

```bash
# 1. Make sure you're on develop and up to date
git checkout develop
git pull origin develop

# 2. Create your feature branch
git checkout -b feature/your-task-name

# 3. Do your work, commit with clear messages
git add .
git commit -m "feat: add wallpapers endpoint with pagination"

# 4. Push your branch
git push origin feature/your-task-name

# 5. Open a Pull Request on GitHub targeting 'develop'
# Your sub-team partner reviews it

# 6. After merge, clean up
git checkout develop
git pull origin develop
git branch -d feature/your-task-name
```

## Branch Naming

| Prefix | Use for | Example |
|--------|---------|---------|
| `feature/` | New features or infrastructure | `feature/api-wallpapers-endpoint` |
| `docs/` | Documentation only | `docs/update-schema-plan` |
| `fix/` | Bug fixes | `fix/sync-timestamp-parsing` |

## Commit Message Format

Use clear, descriptive commit messages:

```
feat: add GET /api/v1/wallpapers with tag filtering
fix: correct delta sync timestamp comparison
docs: update schema_plan.md with collections table
chore: add PostgreSQL to docker-compose
```

## Sub-Team Workflow

| Sub-Team | Partner Reviews Your PRs |
|----------|--------------------------|
| Core API & Architecture | Aiso ↔ Super Jerry |
| Product & UI/UX Client | Akintoluvic ↔ Odytom |
| Data Engine (Python) | Sheriff ↔ Yazn |
| DevOps & Security | Odytom ↔ Sheriff |

## Rules

1. **Never push directly to `main`** — always go through `develop` via PR.
2. **Never commit `.env` files** — they contain secrets. Use `.env.example` for templates.
3. **Keep PRs small and focused** — one feature per PR, not a week's worth of changes in one dump.
4. **Pull `develop` before creating a new branch** — avoids merge conflicts.
