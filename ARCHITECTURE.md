# BizSawa Architecture

**Audience:** engineers and new contributors. **Purpose:** how the pieces fit, and where to read more.

This is the map. Detail lives next to the code and is linked below; don't duplicate it here.

## Table of Contents
1. [Overview](#1-overview)
2. [Services](#2-services)
3. [Key design decisions](#3-key-design-decisions)
4. [Request and data flow](#4-request-and-data-flow)
5. [Technology stack](#5-technology-stack)
6. [Environments and deployment](#6-environments-and-deployment)
7. [Known gaps](#7-known-gaps)
8. [Further reading](#8-further-reading)

---

## 1. Overview

BizSawa is a mobile-first, offline-first business management platform for Kenyan and East African small traders: sales/POS, inventory, invoices (with WhatsApp delivery), expenses, M-Pesa payments and analytics. See [`PRODUCT.md`](./PRODUCT.md) for the product brief.

It is a **monorepo** with a Go **modular monolith** backend and an Expo mobile client that treats local storage as the source of truth while offline.

```
┌──────────────────────────┐         ┌───────────────────────────────────┐
│ Mobile (Expo / RN)       │  HTTPS  │ Backend (Go)                      │
│ WatermelonDB (offline)   │◄───────►│ Chi API · River workers           │
│ TanStack Query           │ /api/v1 │ GORM · tenant-scoped              │
└──────────────────────────┘  + sync └───────┬───────────┬───────────────┘
                                             │           │
┌──────────────────────────┐          ┌──────▼──┐   ┌────▼────┐
│ Web (Vite + React)       │          │Postgres │   │ Redis   │
│ landing + APK download   │          │ (+River)│   │         │
└──────────────────────────┘          └─────────┘   └─────────┘
                                             │
                    Daraja (M-Pesa) · WAHA (WhatsApp) · Resend (email)
                    OpenRouter (LLM) · Google SSO · OpenTelemetry
```

## 2. Services

### Backend — `backend/`
Go modular monolith (`github.com/Codecx-Org/FinAI/backend`). Chi HTTP API and River (Postgres-backed) workers run in one process, served together with graceful shutdown.

- **Entry points** (`backend/cmd/`): `api` (the server), `migrate` (golang-migrate runner: `up | down N | version`), `mcp-server` (standalone read-only MCP host).
- **Domains** (`backend/internal/`): one package per domain: `auth`, `business`, `tenancy`, `users`, `products`, `inventory`, `customers`, `orders`, `sales`, `invoices`, `payments`, `expenses`, `taxes`, `analytics`, `chat`, `mcp`, `waha`, `sync`, `observability`, `shared`. Each follows `domain.go / handler.go / service.go / repository.go / module.go`, plus `workers.go` / `state_machine.go` where needed.
- **Port:** `:5504` (also the Fly.io internal port). Health: `/health`, `/ready`, `/metrics`.
- **Config:** environment variables; see `backend/.env.example`.
- **Deep dives:** [`backend/README.md`](./backend/README.md) and [`backend/docs/`](./backend/docs/).

### Mobile — `mobile/`
Expo / React Native (Expo Router), NativeWind/Tamagui, TanStack Query, WatermelonDB for local-first data and a sync engine under `mobile/sync/`. Package manager: bun. Android is the primary target (EAS builds).
- **Config:** `EXPO_PUBLIC_API_URL` (backend root; `/api/v1` is appended if missing). See `mobile/.env.example`.
- **Deep dives:** [`mobile/docs/`](./mobile/docs/) (`architecture-overview`, `local-first-sync`, `routing`, `state-data`, `ui-components`).

### Web — `web/`
Vite + React 19 + Tailwind 4 marketing site. Its Download section is driven by `web/src/download.json`, rewritten by the *Preview release* workflow. No backend calls.

### Load tests — `loadtests/`
k6 scenarios, bootstrap/cleanup scripts and findings. See [`loadtests/README.md`](./loadtests/README.md). Run against a preview or throwaway environment, never against production data.

## 3. Key design decisions

These are rules the code relies on. Breaking them is a design change and needs a maintainer's sign-off.

1. **Multi-tenancy is enforced, not advisory.** `TenantResolution` middleware puts the business ID (`X-Business-ID`) in the request context; GORM `BusinessScope` filters every query; the authz middleware checks the caller's role (`OWNER / MANAGER / CASHIER / VIEWER`) against an in-code policy map.
2. **Transactional outbox via River.** A service writes its rows and inserts the River job in the *same* DB transaction. There is no separate broker.
3. **Idempotency on money paths.** `X-Idempotency-Key` is required for payments and used to dedupe orders, sales and subscription payments.
4. **State machines, not status flags,** for orders, payments and invoices (`state_machine.go`, built on `qmuntal/stateless`).
5. **Offline-first clients.** The mobile app writes locally and syncs through `/sync/pull` and `/sync/push`; the server resolves conflicts and validates foreign keys. Sync contract changes must stay compatible with app versions already in the field.
6. **Backward-compatible migrations.** Migrations apply before new code serves traffic (see [`CONTRIBUTING.md`](./CONTRIBUTING.md#8-database-migrations)).

## 4. Request and data flow

Example: recording a sale that later gets paid by M-Pesa.

```
Mobile
  └─ writes sale to WatermelonDB (works offline)
  └─ sync push: POST /api/v1/sync/push
        { X-Business-ID, X-Idempotency-Key, JWT }

Backend (Chi)
  observability → RequestID → RealIP → Recoverer → Timeout
  → CORS → TenantResolution → Idempotency → JWT auth → authz
  Handler → Service → Repository (BusinessScope)
        └─ River InsertTx in the same DB tx (outbox)

River workers (async)
  order / sale / invoice / payment events
  payment worker → Daraja STK push → public POST /mpesa/* callback
  invoice worker → PDF + WhatsApp (WAHA) / email (Resend)
  analytics worker → precomputed snapshots

Mobile
  └─ sync pull: GET /api/v1/sync/pull  → local DB updated
```

Chat: `POST /api/v1/chat/business-owner` uses a tool router over business data and an LLM (OpenRouter, with fallback models) for the reply. The MCP server exposes a read-only tool registry over the same data.

## 5. Technology stack

| Layer | Technology |
|-------|------------|
| Backend language / HTTP | Go, Chi |
| ORM / DB | GORM, PostgreSQL 16, golang-migrate |
| Jobs | River (Postgres-backed) |
| Cache / misc | Redis 7 |
| Auth | JWT access + hashed refresh tokens; email/password, Google SSO, email OTP |
| Payments | M-Pesa Daraja (STK, C2B, B2C) |
| Messaging | WAHA (WhatsApp), Resend (email) |
| LLM | OpenRouter (configurable models, failover list) |
| Observability | OpenTelemetry, Jaeger, Prometheus, Grafana (compose `observability` profile) |
| Mobile | Expo, React Native, TypeScript, WatermelonDB, TanStack Query, NativeWind/Tamagui |
| Web | Vite, React 19, Tailwind 4, TypeScript |
| Tooling | bun, golangci-lint, ESLint/Prettier (mobile), oxlint (web), vitest, k6, Maestro |
| Infra | Docker Compose (local), Fly.io (backend), EAS (mobile), GitHub Actions |

## 6. Environments and deployment

| Environment | Backend | Mobile |
|-------------|---------|--------|
| Local | `make dev-up` in `backend/` (Docker Compose: api, postgres, redis; optional observability and WAHA) | `bun start` in `mobile/`, pointing `EXPO_PUBLIC_API_URL` at the backend |
| Production | Fly.io app `bizsawa-api` (region `nbo`), auto-deployed from `main` | EAS preview build on `main`; production build and public preview release via manual workflows |

Pipeline (`.github/workflows/ci.yml`), triggered on PRs to `main` and pushes to `main`:

```
PR / push → mobile lint + typecheck ─┐
            backend vet + lint + test ├─► e2e api flows ─► (main only) deploy backend (Fly)
                                      └─► (main only) EAS preview build
```

There is **no staging environment** today. Flow is feature branch → PR → CI → review → squash merge → deploy. Rules are in [`CONTRIBUTING.md`](./CONTRIBUTING.md).

Secrets: Fly secrets for the backend, GitHub Actions secrets (`FLY_API_TOKEN`, `EXPO_TOKEN`, `RELEASES_PAT`, optional `API_URL`) for CI, EAS secrets for the app.

## 7. Known gaps

Documented so contributors don't assume otherwise. Fixes are tracked separately.

- `web/` is not built or linted in CI.
- Lint gates are soft: the mobile job tolerates existing warnings and Go lint checks new issues only.
- The `mobile-unit` job is disabled, and the API e2e job passes if no e2e script is found.
- Backend scaffolds not on the hot path: `shared/eventbus`, `shared/idempotency.Store`, `shared/circuitbreaker` (see `backend/README.md`).
- No staging environment.

## 8. Further reading

- [`CONTRIBUTING.md`](./CONTRIBUTING.md): workflow, commits, PRs, migrations
- [`PRODUCT.md`](./PRODUCT.md): product brief and design principles
- [`backend/docs/`](./backend/docs/): architecture, modules, platform, interactions, operations, OpenAPI (`openapi.yaml`)
- [`mobile/docs/`](./mobile/docs/): mobile architecture, sync, routing, state
- [`loadtests/README.md`](./loadtests/README.md) and `loadtests/FINDINGS.md`
