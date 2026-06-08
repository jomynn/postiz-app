# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Postiz is a social media scheduling tool supporting 28+ platforms. Users schedule posts to a calendar, which feeds into a workflow engine that posts at the right time.

## Commands

```bash
# Start dev infrastructure (Postgres, Redis, Temporal) — required before running apps
pnpm run dev:docker

# Run all apps in parallel (frontend + backend + orchestrator + extension)
pnpm run dev

# Run only backend + frontend
pnpm run dev-backend

# Run individual apps
pnpm run dev:backend       # NestJS API on :3000
pnpm run dev:frontend      # Vite React on :4200
pnpm run dev:orchestrator  # Temporal worker on :3002

# Build
pnpm run build             # all apps
pnpm run build:backend
pnpm run build:frontend

# Database
pnpm run prisma-generate   # regenerate Prisma client after schema changes
pnpm run prisma-db-push    # push schema to DB (accepts data loss — dev only)

# Tests (must run from repo root)
pnpm test

# Lint (must run from repo root)
pnpm run lint
```

## Monorepo Structure

```
apps/
  backend/       NestJS REST API
  frontend/      Vite + React SPA
  orchestrator/  Temporal.io worker (background jobs)
  extension/     Browser extension
  sdk/           Public SDK
libraries/
  nestjs-libraries/   Shared NestJS modules, Prisma schema, integrations
  helpers/            Shared utilities and hooks (frontend + backend)
  react-shared-libraries/  Shared React components
```

### TypeScript Path Aliases

```
@gitroom/backend/*         → apps/backend/src/*
@gitroom/frontend/*        → apps/frontend/src/*
@gitroom/helpers/*         → libraries/helpers/src/*
@gitroom/nestjs-libraries/* → libraries/nestjs-libraries/src/*
@gitroom/react/*           → libraries/react-shared-libraries/src/*
@gitroom/orchestrator/*    → apps/orchestrator/src/*
```

## Backend Architecture

Layer order must always be respected:

```
Controller → Service → Repository
Controller → Manager → Service → Repository  (for complex flows)
```

- **Controllers** live in `apps/backend/src/api/` — thin, just wire HTTP to services
- **Business logic** belongs in `libraries/nestjs-libraries/src/` (not in the backend app)
- **Prisma schema** is at `libraries/nestjs-libraries/src/database/prisma/schema.prisma`
- **Social integrations** are in `libraries/nestjs-libraries/src/integrations/social/` — each platform extends `social.abstract.ts`

## Frontend Architecture

- **Routing**: file-based in `apps/frontend/src/app/` (folder = route segment)
- **Components**: `apps/frontend/src/components/`
- **UI primitives**: `apps/frontend/src/components/ui`
- **Data fetching**: always use SWR via the `useFetch` hook from `@gitroom/helpers/utils/custom.fetch`
- Each SWR call must be its own named hook — never inline or return SWR calls from object properties (violates `react-hooks/rules-of-hooks`)

### Styling

- Tailwind 3 — reference `apps/frontend/src/app/colors.scss` and `apps/frontend/tailwind.config.cjs` before writing new components
- Design tokens to use: `primary`, `secondary`, `third`–`seventh`, `input`, `inputText`, `tableBorder`, `newBgColor`, `newBorder`, `newSep`, `newBgColorInner`, etc.
- `--color-custom*` CSS variables and their `customColor*` Tailwind aliases are **deprecated** — do not use them
- Dark/light mode is controlled by the `.dark` / `.light` class on the root element

## Temporal (Orchestrator)

- **Workflows** in `apps/orchestrator/src/workflows/` — deterministic, use `proxyActivities` to call activities
- **Activities** in `apps/orchestrator/src/activities/` — where actual work happens (DB calls, external APIs)
- **Signals** in `apps/orchestrator/src/signals/` — for triggering state changes in running workflows
- Task queue name: `"main"`

## Dev Infrastructure Ports

| Service        | Port |
|----------------|------|
| Backend API    | 3000 |
| Frontend       | 4200 |
| Orchestrator   | 3002 |
| PostgreSQL     | 5432 |
| Redis          | 6379 |
| Temporal       | 7233 |
| Temporal UI    | 8080 |
| pgAdmin        | 8081 |

## Rules

- Use only `pnpm` — never `npm` or `yarn`
- Never install frontend components from npm — write native components
- Linting and tests run only from the repo root
- The system is in production — any schema or data-model change needs a migration; think carefully about backward compatibility before modifying existing behavior
