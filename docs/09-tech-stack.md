# 09. Technology stack

## Summary

edgevault is written in **TypeScript** and runs as one **Cloudflare Worker**.

- Account and vault data lives in **D1**, Cloudflare's SQL database. We describe the tables and write type-checked queries with **Drizzle**.
- Each account gets a small **Durable Object** that checks passwords and blocks brute-force attempts.
- Attachments go to **R2** by default, or to **KV** for people who do not want to add a card to their Cloudflare account.
- HTTP routing uses **Hono**, the web framework Cloudflare lists for building APIs on Workers.
- The Bitwarden web vault is served by the same Worker as static files.
- Tests run inside the real Workers runtime with **Vitest** and Cloudflare's test plugin. Browser tests use **Playwright**.
- Code quality checks: **Biome** (formatting and lint), **Oxlint** (type-aware lint), **Semgrep** (our security rules), plus the TypeScript compiler, knip, and dependency-cruiser.
- Tools run on **Node.js and npm**. Bun works as a package manager but cannot run our tests.

The second half of this page lists what we deliberately do **not** use, and why.

Versions were checked on the npm registry in November 2026. The repository pins exact versions, and Renovate or Dependabot proposes updates.

## What we use

### Runtime (Cloudflare)

| Piece | Used for | Free plan | Notes |
|---|---|---|---|
| Workers | HTTP API for the Bitwarden clients | yes, 10 ms CPU per request, 100,000 requests per day | One Worker per environment |
| Workers Static Assets | Serving the web vault | yes | Cloudflare recommends it over Pages for new projects |
| D1 | Users, devices, folders, items, organizations, collections, 2FA settings | yes, 500 MB per database, 5 million rows read and 100,000 written per day | [ADR 0002](adr/0002-d1-as-primary-store.md) |
| Durable Objects (SQLite-backed) | One `AccountGate` per account: hashing on Free, strict login and 2FA throttling. Later: live sync | yes, 100,000 requests and 13,000 GB-s per day | [ADR 0005](adr/0005-per-account-durable-object.md) |
| R2 | Attachments and Send files (default) | own free allowance of 10 GB; needs a card | [ADR 0007](adr/0007-attachment-storage.md) |
| KV | Attachments without a card (alternative) | yes, 1 GB, 25 MiB per file, 1,000 writes per day | ADR 0007 |
| Rate Limiting binding | Coarse first filter on registration and prelogin | assumed yes; M0 confirms | Approximate and per location, never the only defense |
| Cron Triggers | Purge old trash, expired tokens, stale invites | yes, 5 per account | |
| Workers Secrets | Token signing key, email HMAC key, 2FA secret encryption key | yes | Never in config files or logs |
| Workers Logs and Traces | Observability | yes | Structured and redacted |
| Email Service (optional) | Invites and 2FA notices | only to verified addresses in your own account; any recipient needs Paid | Off by default |

### Application code

| Piece | Version | Used for |
|---|---|---|
| TypeScript | 6.0.x | Language, `strict` mode. TypeScript 7 later ([ADR 0011](adr/0011-toolchain.md)) |
| Hono | 4.13 | Routing and middleware |
| zod | 4.6 | Validating request bodies at the HTTP boundary |
| Drizzle (`drizzle-orm`, `drizzle-kit`) | 0.45 / 0.31 | Table schema, typed queries, D1 batches, migration generation. Query builder only; sync and access checks as reviewed `sql` ([ADR 0010](adr/0010-drizzle-for-d1.md)) |
| Web Crypto (built in) | n/a | Hashing, HMAC tokens, random ids, TOTP (HMAC-SHA1), secret encryption (AES-GCM), constant-time comparison |
| Chanfana | 3.6 | **Candidate, decided in M0.** Cloudflare's OpenAPI generator and validator for Hono. A generated spec could feed the endpoint coverage check |

### Testing ([04-testing-strategy.md](04-testing-strategy.md))

| Piece | Version | Used for |
|---|---|---|
| Vitest | 5.1 | Test runner |
| `@cloudflare/vitest-plugin` | 1.8 | Tests inside the Workers runtime with real D1, Durable Objects, KV, and R2. Replaces the deprecated `@cloudflare/vitest-pool-workers` |
| Our scenario parser and runners | n/a | Tests as data (`specs/`) |
| fast-check | 4.10 | Property-based tests |
| Stryker (`@stryker-mutator/core`, `vitest-runner`) | 10.1 | Mutation testing; M0 spike 3 checks it works with the Workers plugin |
| Playwright | 1.65 | Browser end-to-end tests |
| Bitwarden CLI (`@bitwarden/cli`) | 2026.10.1 | Contract tests with a real client |
| `@bitwarden/sdk-internal` | pre-release builds | Test vectors and crypto for the test client; test-only, pinned to an exact build |
| `otpauth` | 9.6 | Independent TOTP implementation, tests only |
| Vaultwarden container | pinned image | Reference implementation (oracle), never in production |

### Tooling, lint, and static analysis ([ADR 0011](adr/0011-toolchain.md))

| Piece | Version | Used for |
|---|---|---|
| Node.js LTS and npm | pinned in `.node-version` | Running tools; CI installs with `npm ci` |
| Wrangler | 4.162 | Local dev, `wrangler types`, migrations, deploys |
| Biome | 2.6 | Formatting and general lint |
| Oxlint with `oxlint-tsgolint` | 1.93 / 7.0 | Type-aware lint (`no-floating-promises`, `no-misused-promises`, `await-thenable`, and similar) |
| Semgrep | pinned | Project security rules: no `Math.random`, no `===` on secrets, no `sql.raw()`, no `passThroughOnException`, `PLAN_PROFILE` only in `src/composition/` |
| `tsc --noEmit` | with TypeScript | Type check |
| knip | 6.45 | Unused files, exports, and dependencies |
| dependency-cruiser | 18.7 | Layer rules |
| type-coverage | 2.31 | Keeps `any` from creeping in |
| gitleaks | pinned | Secret scanning |
| `npm audit`, osv-scanner | n/a | Vulnerable dependencies |
| CodeQL | GitHub | Code scanning |
| zizmor, actionlint | pinned | GitHub Actions security and syntax |
| typos | pinned | Spell check |
| GitHub Actions | n/a | CI and deploys; actions pinned to commit SHAs |
| Renovate or Dependabot | n/a | Dependency updates |

## Frameworks we looked at

Cloudflare's [framework guides](https://developers.cloudflare.com/workers/framework-guides/) list many frameworks with Workers support. For a backend that serves a ready-made web vault, only the API layer matters.

| Category | Options | Our choice |
|---|---|---|
| API routing | Hono, Chanfana (on Hono), itty-router | **Hono**; Chanfana as an M0 candidate |
| Full-stack web | RedwoodSDK (built for Cloudflare), React Router v7, Astro, SvelteKit, Nuxt, TanStack Start, Next.js through OpenNext | None. We do not render pages; the web vault is a prebuilt upstream build |
| Cloudflare libraries | Agents SDK (`agents`), `@cloudflare/actors` (Durable Object helpers) | None. Agents SDK is for AI agents; `@cloudflare/actors` is still a beta (0.0.1-beta.6) |
| D1 data layer | Drizzle, Kysely with `kysely-d1`, Prisma with `@prisma/adapter-d1` | **Drizzle** ([ADR 0010](adr/0010-drizzle-for-d1.md)) |
| Other languages | Rust (`workers-rs`), Python Workers | TypeScript ([ADR 0001](adr/0001-typescript-on-workers.md)) |

## What we do not use, and why

| Not used | Why not |
|---|---|
| Cloudflare Pages | Cloudflare recommends Workers Static Assets for new projects |
| Cloudflare Containers | Not needed, and not a Free-plan option as far as we know |
| Hyperdrive and external databases | Everything lives on Cloudflare; D1 covers our needs |
| D1 read replication | Needs a session bookmark the Bitwarden clients cannot send, so a client could read stale data right after its own write |
| KV for account or vault data | No transactions or relational queries; eventually consistent |
| Queues and Workflows | No multi-step background jobs in the MVP; may be used later for backups |
| A Durable Object per user as the main database | Harder backups and more moving parts. Revisit after the MVP |
| Drizzle relational queries (`db.query.*`) | Hide the generated SQL where we most need to see it |
| Kysely, Prisma | See [ADR 0010](adr/0010-drizzle-for-d1.md) |
| Full-stack frameworks (RedwoodSDK, Next.js, and others) | We serve a prebuilt web vault; no server rendering needed |
| `@cloudflare/actors` | Still beta |
| Rust or WebAssembly | Slower test loop and heavier setup ([ADR 0001](adr/0001-typescript-on-workers.md)) |
| Bun's test runner (`bun test`) | Cannot load `cloudflare:workers`, so it cannot run Worker tests (POC) |
| Bun as the CI package manager | Allowed locally; CI uses npm because Cloudflare's docs and our test tools are documented with Node |
| `@cloudflare/vitest-pool-workers` | Deprecated; renamed to `@cloudflare/vitest-plugin` |
| Jest | Cloudflare's Workers test plugin supports Vitest |
| ESLint with typescript-eslint | Slower, and supports only TypeScript `<6.1`. Added for a single rule only if no other tool has it |
| Prettier, dprint | Biome formats |
| `ts-prune` | Unmaintained since 2022; knip replaces it |
| Cucumber or Gherkin | Allows steps without assertions; our DSL rejects them ([ADR 0004](adr/0004-scenario-tests-as-data.md)) |
| SMTP | Workers cannot open raw SMTP connections; email goes through Cloudflare Email Service |
| RSA-signed JWTs | More CPU and no benefit while only we verify tokens ([ADR 0006](adr/0006-token-signing.md)) |
| Any Vaultwarden code | AGPL-3.0, and this is an independent implementation ([01-scope.md](01-scope.md#legal-hygiene)) |
