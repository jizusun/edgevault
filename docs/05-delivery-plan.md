# 05. Delivery plan

The MVP is delivered as a sequence of small pull requests. Each one follows the TDD loop in [04-testing-strategy.md](04-testing-strategy.md#tdd-workflow). This design PR contains only documents.

## Milestones

### M0. Walking skeleton and test harness

The aim is a deployed Worker that answers `GET /api/alive`, with the whole test machinery working around it. No Bitwarden feature work beyond discovery endpoints.

Deliverables:

- Repo tooling: TypeScript strict, Vitest, `@cloudflare/vitest-pool-workers`, Biome (or ESLint), dependency-cruiser, gitleaks, CI.
- Scenario parser, printers for `sync of` and `stored vault of`, runner R1, and runner R2.
- Scenarios for `alive`, `now`, `version`, `config`.
- Harness guard CI job (PR type vs paths, coverage matrix).
- CI checks from [08-cloudflare-practices.md](08-cloudflare-practices.md): Wrangler config checks, `wrangler types` diff, migration lint, plus spell check and GitHub Actions security lint (zizmor).
- Spikes with written results in `docs/spikes/`:
  1. PBKDF2 (600,000 iterations) CPU cost on the Free plan, inline vs Durable Object.
  2. Workers Rate Limiting binding on Free.
  3. Stryker with the Workers test pool.
  4. Per-request D1 `meta` access in the in-process runner.
  5. Vaultwarden container as oracle (R3) with shared scenario files.
  6. Running Bitwarden SDK live tests against a Worker.
  7. Which registration payload shapes and account versions ("V2") current clients send; first `client:` table of scenarios.
  8. Per-PR preview environments for the contract stage.
- Staging environment deployed from `main`.

Exit criteria: CI is green with all stages that exist at this point; a red scenario can be written, seen failing, and fixed; spike results update [02-architecture.md](02-architecture.md) and the ADRs where they change a decision.

### M1. Accounts and login

Endpoints in [03-api-surface.md](03-api-surface.md#m1-accounts-and-login).

Exit criteria: `bw` CLI can register (via the new flow), log in, refresh a token, and log out against `wrangler dev` and staging. Password change rotates the security stamp and old tokens stop working. Login and prelogin scenarios include failure cases and account-enumeration checks. [THREAT_MODEL.md](../THREAT_MODEL.md) is reviewed against the implemented design and updated in this milestone's PRs. The per-account Durable Object ([ADR 0005](adr/0005-per-account-durable-object.md)) and token design ([ADR 0006](adr/0006-token-signing.md)) are implemented here.

### M2. Vault

Endpoints in [03-api-surface.md](03-api-surface.md#m2-vault).

Exit criteria: create, edit, trash, restore, delete, move, and import work and sync across two devices in a scenario. Budget scenarios for sync and import pass. Property-based invariants pass.

### M3. Delivery and hardening

- Web vault served from the Worker at a pinned upstream version.
- Rate limits, cron cleanup.
- Browser e2e (Playwright) for login, add item, second-session sync.
- Compatibility matrix with exact client versions.
- Manual exploratory checklist.
- "Deploy to your own account" documentation and a one-workflow GitHub Actions deploy.
- Nightly job set (stage 7 to 9) running.

Exit criteria: all [MVP acceptance criteria](01-scope.md#acceptance-criteria-for-the-mvp) hold.

## PR slicing

Each PR is one of four types, which CI checks against changed paths (see [protecting the harness](04-testing-strategy.md#protecting-the-harness)):

| Type | May change |
|---|---|
| `feature` | `src/`, `specs/` |
| `refactor` | `src/` only |
| `coverage` | `specs/` only |
| `dsl` | `specs/`, `test/runner/` |

A PR for one endpoint group has at least these commits, in order: scenarios (red), implementation (green), optional refactor. A reviewer can check out the first commit and see the scenarios fail.

Target PR size: under about 400 changed lines excluding generated files and scenario data.

## Branch and review rules

- `main` is protected: required checks are CI stages 1 to 5; one approving review; no force pushes.
- Conventional commit messages.
- A PR description states the type, the scenario ids added or changed, and the oracle result for new endpoints.

## Definition of done (every PR)

1. New behavior is described by scenarios or table rows, which failed before the change.
2. Unit tests exist for new domain rules, with properties where an invariant applies.
3. CI stages 1 to 5 pass; stage 6 shows no new surviving mutants in changed domain code, or each is explained.
4. New endpoints are checked against the oracle and the result is in the PR description.
5. Budgets for the touched endpoints are declared and pass.
6. Docs are updated if a decision changed, and an ADR is added for any new significant decision.
7. No secrets, no vault data, no personal data in code, fixtures, or logs.

## Risks

| Risk | Mitigation |
|---|---|
| Password hashing does not fit the Free CPU limit and the Durable Object route has a hidden cost | M0 spike 1 settles it before M1 starts; the port allows swapping the hasher without touching use cases |
| Bitwarden clients change API behavior | Pinned client versions; nightly contract job on latest; scenario per endpoint spelling |
| Oracle (Vaultwarden) behavior differs from official Bitwarden | Triage each difference; prefer official docs and SDK tests when they disagree with Vaultwarden |
| Stryker does not work in the Workers pool | Mutation testing at unit level only, plus recorded fault injection (see spikes) |
| D1 daily limits hit by a busy account on Free | Budget tests; document the upgrade path; the Paid profile removes the daily caps |
| Security bug in a young project | Threat model, strict per-account throttle, dependency audit, minimal feature set; the README says plainly that the project is unaudited |
| Durable Object quota (100,000 requests per day on Free) is exhausted and login stops | Budget test, at most two Durable Object calls per login, monitoring checklist, upgrade path |
| Clients change registration or account formats (shapes differ by client version) | `client:` column in scenario tables; nightly contract job on latest clients; oracle check |
| Web vault license or redistribution questions | Open question 4; the build downloads a pinned upstream release rather than committing it |
