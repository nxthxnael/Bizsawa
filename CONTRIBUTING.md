# Contributing to BizSawa

This guide is mandatory reading for everyone who changes this repo. It keeps `main` releasable, because **a merge to `main` deploys** (see [Deployment](#6-deployment)).

For how the system fits together, read [ARCHITECTURE.md](./ARCHITECTURE.md) first.

## Table of Contents
1. [Repo layout](#1-repo-layout)
2. [Repository and remotes](#2-repository-and-remotes)
3. [Branching](#3-branching)
4. [Commits](#4-commits)
5. [Local setup and checks](#5-local-setup-and-checks)
6. [Deployment](#6-deployment)
7. [Pull requests and review](#7-pull-requests-and-review)
8. [Database migrations](#8-database-migrations)
9. [Secrets and environment variables](#9-secrets-and-environment-variables)
10. [Dependencies](#10-dependencies)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Repo layout

| Path | What | Stack |
|------|------|-------|
| `backend/` | API, workers, migrations | Go, Chi, GORM, River, PostgreSQL, Redis |
| `mobile/` | Offline-first app | Expo / React Native, WatermelonDB, bun |
| `web/` | Marketing + download site | Vite, React 19, Tailwind 4, bun |
| `loadtests/` | k6 load tests | k6, shell |
| `.github/` | CI, dependabot, templates | GitHub Actions |

Loose planning notes (`NOTES/`, `Agents_Documents/`, root `*_plan.md`) are working documents, not specs. Do not treat them as the source of truth; `ARCHITECTURE.md` and `backend/docs/`, `mobile/docs/` are.

## 2. Repository and remotes

The working repo is **`nxthxnael/Bizsawa`** (`origin`), a fork of `Chris-Mwiti/Bizsawa`. **All commits, branches and PRs live in `nxthxnael/Bizsawa`.** Nothing is pushed to, or opened against, the original repo.

```bash
git clone https://github.com/nxthxnael/Bizsawa.git
cd Bizsawa
git remote -v   # origin = nxthxnael/Bizsawa
```

To pull in changes from the original repo, add it as a read-only remote and merge them through a normal branch and PR:

```bash
git remote add upstream https://github.com/Chris-Mwiti/Bizsawa.git
git fetch upstream
git checkout -b chore/sync-upstream main
git merge upstream/main
```

Rules:
- Never push to `upstream`.
- Never commit directly to `main`; every change goes on a branch and into `main` through a PR.
- Fork Actions need no deploy secrets. Deploy steps skip when `FLY_API_TOKEN` / `EXPO_TOKEN` are unset. Do not add production secrets to a fork you do not control.

## 3. Branching

**Trunk-based:** `main` is the only long-lived branch. Branches are short-lived (aim for days, not weeks) and merge back through a PR.

Every change, however small, gets its own branch.

**Format:** `<type>/<short-description>` — lowercase, hyphen-separated. Add an issue number when one exists: `<type>/<issue>-<short-description>`.

| Type | Use for | Example |
|------|---------|---------|
| `feat/` | New feature | `feat/142-invoice-reminders` |
| `fix/` | Bug fix | `fix/sync-conflict-loop` |
| `perf/` | Performance | `perf/mobile-cold-start` |
| `refactor/` | No behaviour change | `refactor/orders-service` |
| `test/` | Tests / load tests | `test/loadtests-sales` |
| `docs/` | Docs only | `docs/contributing-rewrite` |
| `chore/` | Tooling, deps, CI | `chore/bump-expo` |

Rules:
- Branch from an up-to-date `main`.
- No direct pushes to `main`.
- Delete the branch after merge.
- Keep PRs small and single-purpose. A migration, the code that uses it, and its tests belong together; unrelated cleanups do not.

## 4. Commits

[Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <subject>

- <present-tense bullet describing one change>
- <present-tense bullet describing another>
```

Every commit has a bullet body. Bullets are present tense ("add", "remove", "fix"), one change each.

**Types:** `feat`, `fix`, `perf`, `refactor`, `test`, `docs`, `style`, `chore`.

**Scopes:** `backend`, `mobile`, `web`, `migrations`, `loadtests`, `ci`, `deps`. A backend domain package may be used instead of `backend` (`orders`, `invoices`, `sync`, `payments`, `chat`, `sales`, etc.), matching existing history (`fix(invoices): …`).

**Subject:** imperative, lowercase, no trailing period, 72 characters or fewer.

Examples (from this repo's style):

```
perf(mobile): defer sync startup work past first paint
fix(sync): stop re-pushing resolved conflicts
feat(migrations): add rollback for load-test indexes
```

Do not commit `wip`, `fix` or `update` with no context. Reference issues in the body (`Fixes #142`). **Commits carry no AI attribution or co-author trailers**; the author is the committer only. After committing, push the branch to `origin`. Squash merge uses the PR title as the commit subject, so the **PR title must follow this format too**.

## 5. Local setup and checks

Prerequisites: Git, Docker + Compose, Go (version in `backend/go.mod`; CI uses 1.25), [bun](https://bun.sh) (CI uses 1.3.10), and for mobile an Android emulator/device or iOS simulator.

### Backend

```bash
cd backend
cp .env.example .env.development    # fill in values; see section 9
make dev-up                          # builds + starts api, postgres, redis (api on :5504)
make dev-migration                   # applies migrations via the compose `migrate` profile
make dev-logs
make dev-down
```

Optional: `make dev-observability` (OTel, Jaeger, Prometheus, Grafana). To run without Docker for the API: start postgres and redis, then `go run ./cmd/migrate up` and `go run ./cmd/api` with `DATABASE_DSN` etc. set.

### Mobile

```bash
cd mobile
cp .env.example .env.development     # set EXPO_PUBLIC_API_URL to your backend
bun install --frozen-lockfile
bun start                            # or: bun run android / bun run android:lan
```

### Web

```bash
cd web
bun install
bun run dev
```

### Checks to run before opening a PR

Run what you touched. These mirror CI (`.github/workflows/ci.yml`).

| Area | Command |
|------|---------|
| Backend | `cd backend && go vet ./... && golangci-lint run ./... && go test ./... -run Test -count=1` |
| Mobile | `cd mobile && bun run lint && npx tsc --noEmit --skipLibCheck && bun run test` |
| Mobile API e2e | `cd mobile && bun run test:e2e:api` (needs a running backend; see `mobile/scripts/e2e.sh`) |
| Web | `cd web && bun run lint && bun run build` (**not yet in CI**; run it yourself) |

Backend tests need Postgres and Redis (CI uses `bizsawa_test`). Lint is gated on **no new issues**: do not add warnings to existing legacy debt.

## 6. Deployment

On push to `main`, CI runs and then:
- **Backend:** deploys to Fly.io (`bizsawa-api`, region `nbo`), then health-checks `/ready`.
- **Mobile:** starts an EAS preview Android build.
- Production mobile builds and public preview releases are **manual** workflows (`workflow_dispatch`).

Consequences for contributors:
- Every merge to `main` can ship. Do not merge anything you would not want in production.
- Merge only when CI is green. Branch protection on `main` must require the CI checks, at least one approving review, and no direct pushes.
- Migrations run **before** the new app version serves traffic, so they must be backward compatible with the currently deployed code (see section 8).
- A deploy step that prints "skipping" because a secret is missing still shows green. If you change deploy behaviour, check the job logs, not only the status.
- Rollback: revert the PR on `main` (a new PR) and let the pipeline redeploy. For migrations, see the `down` file and `backend/docs/operations.md`.

## 7. Pull requests and review

1. Rebase your branch on the latest `main` (`git fetch origin && git rebase origin/main`).
2. Run the checks in section 5.
3. Push the branch to `origin` and open a PR against `nxthxnael/Bizsawa:main`. Fill in the PR template completely and write the description in the first person ("I add...", "I fixed...").
4. CI must pass. Do not ask for review on a red PR unless you say why.

**Review requirements**
- At least **1 approval** from a code owner ([CODEOWNERS](./.github/CODEOWNERS)). Code owners are requested automatically.
- Changes to `backend/migrations/`, `.github/`, auth, payments (M-Pesa) or tenancy need an owner's explicit approval.
- While the repo has a single maintainer, self-review the diff against the checklist below before merging; once others have write access, authors do not merge their own PR without an approval.
- Address every review thread; resolve it, don't hide it.

**Reviewer checklist**
- [ ] PR title and commits follow section 4
- [ ] Scope is single-purpose; unrelated changes are split out
- [ ] Tests added or updated for new behaviour (backend logic, sync, money paths especially)
- [ ] Migrations follow section 8
- [ ] New env vars are in `.env.example` and the PR description
- [ ] No secrets, tokens or real customer data
- [ ] Multi-tenant safety: queries go through the business scope; no cross-tenant reads
- [ ] Money-moving flows stay idempotent (`X-Idempotency-Key`) and use the existing state machines
- [ ] No stray `console.log` / debug output; errors are handled, not swallowed
- [ ] No new lint warnings

**Merging:** use **squash and merge** (clean history, PR title becomes the commit). Delete the branch afterwards.

## 8. Database migrations

Migrations live in `backend/migrations/` as paired, sequentially numbered files, applied with `golang-migrate` (`go run ./cmd/migrate up`):

```
000022_add_widget_table.up.sql
000022_add_widget_table.down.sql
```

Rules:
- **Always add both `up` and `down`.** The `down` must actually undo the `up`.
- **Never edit a migration that has been merged.** Add a new one.
- Take the next free number. If two PRs collide, the later one renumbers before merge.
- Make migrations backward compatible with the running version: add columns nullable or with defaults, backfill, then tighten in a later release. Don't drop or rename a column in the same release that stops using it.
- Large index builds on big tables: use `CREATE INDEX CONCURRENTLY` where possible and say so in the PR.
- Test locally: `up`, run the tests, `down`, then `up` again.
- Destructive changes (drops, type changes, data rewrites) need an owner's approval and a written rollback plan in the PR.

## 9. Secrets and environment variables

- Never commit secrets. Only `.env.example` files are tracked; real values go in `.env.development` / `.env.production` (gitignored), Fly secrets (`fly secrets set`), or EAS secrets.
- Any new variable must be added (with an empty or safe value and a comment) to `backend/.env.example` or `mobile/.env.example` in the same PR, and listed in the PR template.
- `EXPO_PUBLIC_*` variables are **bundled into the app and readable by anyone**. Never put a private key there.
- Test keys in `ci.yml` are throwaway values for CI only; don't reuse them elsewhere.
- If you leak a secret, tell a maintainer immediately and rotate it. Deleting the commit is not enough.
- Don't paste real customer, M-Pesa or WhatsApp data into issues, PRs, tests or fixtures.

## 10. Dependencies

- Dependabot opens weekly PRs for `backend` (Go modules) and `mobile` (npm). Review them like any other PR.
- Don't add a dependency for a few lines of code. Justify new dependencies in the PR.
- Commit the lockfile with the change: `backend/go.sum`, `mobile/bun.lock`, `web/bun.lock`.
- Use `bun` for `mobile/` and `web/`. Don't add a second lockfile (`package-lock.json`, `yarn.lock`).

## 11. Troubleshooting

**I committed to `main` by accident**
```bash
git branch feat/my-change           # keep the work
git reset --hard origin/main        # put local main back to the remote state
git checkout feat/my-change
```

**Docker containers won't start**
```bash
cd backend && make dev-down && make dev-up && make dev-logs
```

**Migration failed**
Check `go run ./cmd/migrate version`. A failed migration can leave the version marked dirty. On a **local/dev** database, the simplest fix is to recreate it (`make dev-down`, remove the postgres volume, `make dev-up`). Never touch a production database without a maintainer.

---

Questions? Open a GitHub issue or ask a code owner. See [ARCHITECTURE.md](./ARCHITECTURE.md) for system design.
