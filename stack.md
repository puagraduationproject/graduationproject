# Stack: backend, frontend and security

> Status: **agreed 2026-10-20 for the backend and frontend**, and can change
> later. Each subteam can change its own part, as long as the contracts between
> parts hold. The AI side (agent, models, training) is the AI team's choice and
> is not covered here.

## Where it runs

Cloud only for now. The app is a website, opened in the browser on any device.
A desktop app comes later.

## Frontend

| Need | Proposal | Why |
|------|----------|-----|
| Staff app | React, TypeScript, Vite | One code base for phone, tablet and laptop, opened in the browser |
| Shop pages and landing | Astro | Fast pages that search engines can read |
| Look and parts | Tailwind, shadcn/ui | Ready parts we own and can change |
| Languages | i18next | Arabic right-to-left and English |
| Loading data | TanStack Query | Caching, retries and refresh without hand work |
| Streaming answers | Server-sent events | Text first, cards when functions return |
| Tests | Vitest, Playwright | Unit tests for parts, browser tests for the main flows |

## Backend

| Need | Proposal | Why |
|------|----------|-----|
| Language and runtime | TypeScript on Node 22 | Same language as the frontend; typed |
| HTTP | Fastify | Light, fast, schema-based input checks |
| Database | PostgreSQL 16 | One schema per module, row-level security on `business_id`, transactions |
| Queries | Kysely | Typed queries; the SQL stays visible |
| Migrations | Plain SQL files per module schema | Security rules and policies stay readable |
| Outbox and background jobs | pg-boss | Runs on PostgreSQL, no extra service |
| Contracts | JSON Schema per function, event and card | One source for the API description, input checks and types |
| API description | OpenAPI, generated from the contracts | Anyone can use the functions from their pages alone |
| Module walls | Lint rules on imports | A module cannot import another module's code or tables |
| Tests | Vitest with a real PostgreSQL in a container | Includes the test that tries to read another business's data and must fail |

## Security

| Need | Proposal |
|------|----------|
| Passwords | argon2id |
| Login | Sessions in secure, HTTP-only cookies, lock-out after repeated failed tries (sessions or tokens is still open; the security member decides) |
| Permissions | Roles and permissions checked in the backend on every function call, never only in the app |
| Data between businesses | Row-level security as the last wall, plus one database user per module |
| Audit | Every write in the audit log, with who did it (member, AI or system) |
| Abuse | Rate limits on the login and the public chat |
| Secrets | Never in the code; read from the environment at start |
| Checks in CI | Dependency scanning and secret scanning on every push |

## What the choice must keep

- The contracts are the center: the backend, the app and every caller read
  the same JSON Schema
- Money as integers in cents with a currency; stock as a ledger of movements
- Every business-scoped table has `business_id` and row-level security
- Events go through the outbox in the same transaction as the change
