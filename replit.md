# Workspace

## Overview

pnpm workspace monorepo using TypeScript. This project is the **replit-2api-Deployer** — an OpenAI/Anthropic-compatible API proxy that forwards requests to OpenAI / Anthropic / Gemini through Replit AI Integrations.

**For deployment & re-use by another agent, read `README.md` first.** It contains the full step-by-step deployment SOP, environment requirements, model catalog, and troubleshooting playbook.

## Stack

- **Monorepo tool**: pnpm workspaces
- **Node.js version**: 24
- **Package manager**: pnpm
- **TypeScript version**: 5.9
- **API framework**: Express 5
- **Database**: PostgreSQL + Drizzle ORM
- **Validation**: Zod (`zod/v4`), `drizzle-zod`
- **API codegen**: Orval (from OpenAPI spec)
- **Build**: esbuild (CJS bundle)

## Key Commands

- `pnpm run typecheck` — full typecheck across all packages
- `pnpm run build` — typecheck + build all packages
- `pnpm --filter @workspace/api-spec run codegen` — regenerate API hooks and Zod schemas from OpenAPI spec
- `pnpm --filter @workspace/db run push` — push DB schema changes (dev only)
- `pnpm --filter @workspace/api-server run dev` — run API server locally

See the `pnpm-workspace` skill for workspace structure, TypeScript setup, and package details.
