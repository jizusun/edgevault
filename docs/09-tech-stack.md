# 09. Technology stack

## Summary

edgevault is written in **TypeScript** and runs as one **Cloudflare Worker**.

- Account and vault data lives in **D1**, Cloudflare's SQL database.
- Each account gets a small **Durable Object** that checks passwords and blocks brute-force attempts.
- Attachments go to **R2** by default, or to **KV** for people who do not want to add a card to their Cloudflare account.
- The Bitwarden web vault is served by the same Worker as static files.
- Tests run inside the real Workers runtime with **Vitest** and Cloudflare's test plugin, and browser tests use **Playwright**.

The second half of this page lists what we deliberately do **not** use, and why.

Versions were checked on the npm registry in November 2026. The repository pins exact versions, and Renovate or Dependabot proposes updates.

## What we use

### Runtime (Cloudflare)

| Piece | Used for | Free plan | Notes |
|---|---|---|---|
| Workers | HTTP API for the Bitwarden clients | yes, 10 ms CPU per request, 100,000 requests per day | One Worker per environment |
| Workers Static Assets | Serving the web vault | yes | Cloudflare recommends it over Pages for new projects |
| D1 | Users, devices, folders, items, organizations, collections, 2FA settings | yes, 500 MB per database, 5 million rows read and 100,000 written per day | [ADR 0002](adr/0002-d1-as-primary-store.md) |
| Durable Objects (SQLite-backed) | One `AccountGate` per account: password hashing on Free, strict login and 2FA throttling. Later: live sync over WebSocket | yes, 100,000 requests and 13,000 GB-s per day | [ADR 0005](adr/0005-per-account-durable-object.md) |
| R2 | Attachments and Send files (default) | own free allowance of 10 GB; needs a card on the account | [ADR 0007](adr/0007-attachment-storage.md) |
| KV | Attachments for accounts without a card (alternative) | yes, 1 GB, 25 MiB per file, 1,000 writes per day | ADR 0007 |
| Rate Limiting binding | Coarse first filter on registration and prelogin | assumed yes; M0 confirms | Approximate and per location, never the only defense |
| Cron Triggers | Purge old trash, expired tokens, stale invites | yes, 5 per account | |
| Workers Secrets | Token signing key, email HMAC key, 2FA secret encryption key | yes | Never in config files or logs |
| Workers Logs and Traces | Observability | yes | Structured and redacted |
| Email Service (optional) | Organization invites and 2FA notices by email | only to verified addresses in your own account; arbitrary recipients need Paid | Off by default; organizations work without email |

### Application code

| Piece | Version (Nov 2026) | Used for |
|---|---|---|
| TypeScript | 6.0.x (not 7.x yet) | Language, `strict` mode. TypeScript 7.0 exists, but `typescript-eslint` still only supports `<6.1`, and other tools may lag too. Revisit in M0 |
| Hono | 4.13 | Routing and middleware on Workers |
| zod | 4.6 | Validating request bodies at the HTTP boundary |
| Web Crypto (built in) | n/a | Hashing, HMAC tokens, random ids, TOTP (HMAC-SHA1), secret encryption (AES-GCM), constant-time comparison (`timingSafeEqual`) |
| Plain SQL with small typed helpers | n/a | D1 queries. Keeps every query visible, which matters for row and query budgets |

### Testing (see [04-testing-strategy.md](04-testing-strategy.md))

| Piece | Version | Used for |
|---|---|---|
| Vitest | 5.1 | Test runner |
| `@cloudflare/vitest-plugin` | 1.8 | Runs tests inside the Workers runtime with real D1, Durable Objects, KV, and R2. It replaces `@cloudflare/vitest-pool-workers`, which is deprecated |
| Our scenario parser and runners | n/a | Tests as data (`specs/`) |
| fast-check | 4.10 | Property-based tests |
| Stryker (`@stryker-mutator/core` and `vitest-runner`) | 10.1 | Mutation testing; whether it works with the Workers plugin is an M0 spike |
| Playwright | 1.65 | Browser end-to-end tests with the web vault |
| Bitwarden CLI (`@bitwarden/cli`) | 2026.10.1 | Contract tests with a real client |
| `@bitwarden/sdk-internal` | pre-release builds | Test vectors and crypto for the scenario test client. It is published only as pre-release versions, so it is a test-only dependency, pinned to an exact build |
| `otpauth` | 9.6 | Independent TOTP implementation used only in tests, to check our own TOTP code. RFC 6238 test vectors are the primary check |
| Vaultwarden container | pinned image | Reference implementation (oracle) in nightly tests, never in production |

### Tooling and CI

| Piece | Version | Used for |
|---|---|---|
| Wrangler | 4.162 | Local dev, `wrangler types`, migrations, deploys |
| Biome | 2.6 | Formatting and linting, including its `noFloatingPromises` rule |
| dependency-cruiser | 18.7 | Enforcing layer rules |
| gitleaks | latest pinned | Secret scanning |
| zizmor, actionlint | latest pinned | GitHub Actions security and syntax lint |
| typos | latest pinned | Spell check |
| GitHub Actions | n/a | CI and deploys; actions pinned to commit SHAs |
| Renovate or Dependabot | n/a | Dependency updates |

## What we do not use, and why

| Not used | Why not |
|---|---|
| Cloudflare Pages | Cloudflare now recommends Workers Static Assets for new projects |
| Cloudflare Containers | Not available on the Free plan as far as we know, and not needed |
| Hyperdrive and external databases | Everything lives on Cloudflare; D1 covers our needs |
| D1 read replication | Needs the Sessions API and a bookmark the Bitwarden clients cannot send, so a client could read stale data right after its own write |
| KV for account or vault data | No transactions or relational queries; eventually consistent |
| Queues and Workflows | No multi-step background jobs in the MVP. May be used later for backups |
| A Durable Object per user as the main database | Better isolation, but harder backups and more moving parts. Revisit after the MVP ([ADR 0002](adr/0002-d1-as-primary-store.md)) |
| An ORM (for example Drizzle) | Hides the exact queries, which we need to see to keep within D1 row and query budgets. Can be revisited |
| Rust or WebAssembly | Slower test loop and heavier setup for this team ([ADR 0001](adr/0001-typescript-on-workers.md)) |
| `@cloudflare/vitest-pool-workers` | Deprecated; renamed to `@cloudflare/vitest-plugin` |
| Jest | Vitest is the runner Cloudflare's Workers test plugin supports |
| ESLint and Prettier | Biome covers formatting and the lint rules we need, with one tool. If a rule we need is missing (for example a typed rule), we add `typescript-eslint` for that rule only |
| Cucumber or Gherkin | Allows steps without assertions; our own small DSL rejects them ([ADR 0004](adr/0004-scenario-tests-as-data.md)) |
| SMTP | Workers cannot open raw SMTP connections; email, if enabled, goes through Cloudflare Email Service |
| RSA-signed JWTs | Costs more CPU and gives no benefit while only we verify tokens ([ADR 0006](adr/0006-token-signing.md)) |
| Any Vaultwarden code | AGPL-3.0, and this is an independent implementation ([01-scope.md](01-scope.md#legal-hygiene)) |
