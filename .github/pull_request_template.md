## What does this PR do?
<!-- First person ("I add...", "I fixed..."). Clear, concise explanation. The PR title must follow Conventional Commits: type(scope): subject -->

**Related issue:** Closes #

## Areas affected
- [ ] Backend (`backend/`)
- [ ] Mobile (`mobile/`)
- [ ] Web (`web/`)
- [ ] Load tests (`loadtests/`)
- [ ] CI / infra / docs (`.github/`, Docker, Fly, EAS)

## Database migrations
<!-- Mandatory if you touched backend/migrations -->
- [ ] No database changes
- [ ] Added a paired `NNNNNN_name.up.sql` and `.down.sql`; no merged migration was edited
- [ ] Tested locally: `up` → tests → `down` → `up`
- [ ] Backward compatible with the currently deployed backend (migrations run before the new code)
- [ ] Destructive change: rollback plan below

**Migration details / rollback plan:**

## Environment variables and secrets
- [ ] No new environment variables
- [ ] New variables added to `.env.example` (backend and/or mobile) and listed here:
  - `VAR_NAME`: what it is, where to get the value
- [ ] No secrets, tokens or real customer data in the diff
- [ ] No private values in `EXPO_PUBLIC_*` variables

## Deployment impact
Merging to `main` deploys the backend (Fly.io) and starts an EAS preview build.

- [ ] Standard deploy, no manual steps
- [ ] Needs manual action before/after merge (explain):
- [ ] Sync/API contract change; older app versions in the field still work

## Testing
- [ ] Manually tested (describe what you did and on what: emulator/device/backend version):
- [ ] Backend: `go vet ./... && golangci-lint run ./... && go test ./... -run Test -count=1`
- [ ] Mobile: `bun run lint && npx tsc --noEmit --skipLibCheck && bun run test`
- [ ] Web: `bun run lint && bun run build` (not covered by CI yet)
- [ ] Tests added or updated for new behaviour
- [ ] No new lint warnings

## Checklist
- [ ] Commits (bullet bodies, no AI attribution) and PR title follow Conventional Commits
- [ ] Branch is rebased on the latest `main`
- [ ] Tenant scoping and idempotency preserved for any new or changed endpoints
- [ ] No leftover debug output or commented-out code
- [ ] Docs updated if behaviour or setup changed (`ARCHITECTURE.md`, `backend/docs/`, `mobile/docs/`)

## Screenshots / recordings
<!-- UI changes: before and after -->

---
See [CONTRIBUTING.md](../CONTRIBUTING.md) and [ARCHITECTURE.md](../ARCHITECTURE.md).
