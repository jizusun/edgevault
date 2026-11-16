# 06. Prior art

What we looked at, what we measured, and what we take from each. Numbers are from the repositories and the GitHub API in November 2026. Contributor counts are for the project's own history.

## Projects

| Project | Stack | Contributors | Tests found | Notes |
|---|---|---|---|---|
| [warden-worker](https://github.com/qaz741wsd856/warden-worker) | Rust, D1, MIT | 7 | 2 unit tests; none run in CI | Documents Free-plan handling in detail |
| [NodeWarden](https://github.com/shuaiplus/nodewarden) | TypeScript, D1, R2 or KV, LGPL-3.0 per its README | 28 | 47 cases in 7 files (Node test runner); not run in CI; CodeQL and Semgrep run | Most active; no organizations |
| [Vaultwarden](https://github.com/dani-garcia/vaultwarden) | Rust, SQLite, MySQL or Postgres, AGPL-3.0 | 100+ | about 67 unit tests in `src/`; nightly Bitwarden SDK live tests | The reference for API behavior; cannot run on Workers |
| [vaultwarden-cloudflare](https://github.com/cybercypher/vaultwarden-cloudflare) | Rust port of Vaultwarden, Workers, D1, KV, R2 | 1 own author, AI-assisted; contributor list is inherited from upstream | README lists an integration suite; not verified | Very new; claims many features including organizations |
| [vaultwarden-cf](https://github.com/adam0white/vaultwarden-cf) | Vaultwarden in Cloudflare Containers | 1 | none found | Needs an external database; Containers are not a Free-plan option as far as we know |

## What each one teaches

### warden-worker

Take:

- **CPU offloading to a Durable Object.** The Free plan's 10 ms CPU limit is too small for server-side PBKDF2 and for large imports. Durable Objects have a 30 s CPU default, and they are available on the Free plan with SQLite storage. Our `free` profile uses the same idea behind a port.
- **Free vs card-required storage.** KV for attachments (25 MiB, no card) and R2 (100 MB, card needed), with R2 taking priority when configured.
- **Static assets for the web vault**, with API paths routed to the Worker, and a pinned upstream web vault version.
- **Built-in rate limiting** on login, registration, and prelogin using the Workers Rate Limiting binding, degrading gracefully if the binding is missing.
- **Honest scope statement** (personal vault; no organizations).

Avoid:

- Almost no tests and no test step in CI.
- D1 2 MB row limit leaking into product behavior (text Send size) without a stated cap and a test.

### NodeWarden

Take:

- **Wide feature coverage for a personal vault**: multi-user with invite codes, passkey login, backup to S3 or WebDAV, device management, import formats.
- **A compatibility test for vault shape that runs the real production SQL** (in an in-memory SQLite) rather than only mocks.
- **A contributor guide that lists cross-file checks** (schema, backup, locale files). We keep the idea of a checklist, but enforce it in CI.
- **Security scanning in CI** (CodeQL, Semgrep, actionlint).

Avoid:

- Tests that are not part of CI, and a guide that recommends only a type-check and build.
- Mocking storage methods in tests, which couples tests to implementation.
- A package script that points to a test file that does not exist. Our coverage-matrix check and CI catch this class of drift.
- Large unreviewed community PRs (an organizations PR touching 92 files has sat open since September 2026). Our PR size target and slicing exist to avoid this.

### Vaultwarden

Take:

- **The route table** as the starting list of endpoints (see [03-api-surface.md](03-api-surface.md)).
- **The oracle idea**: its workflow `sdk-live-tests.yml` runs the Bitwarden SDK's live end-to-end tests against the server every night. The SDK publishes test vectors and live tests that any compatible server can use.
- **A test-only seeder** that exists only in a build with an `unsafe_test_mode` feature and is guarded by credentials. We want the same property: test-only code must not be reachable in production builds, enforced by a build flag and a dependency-cruiser rule.
- **Both `POST` and `PUT`/`DELETE` spellings** for many endpoints, because real clients differ.

Avoid:

- Copying code. It is AGPL-3.0; we implement from observed behavior and public docs.

### vaultwarden-cloudflare and vaultwarden-cf

Take:

- Evidence that a fuller port is possible in principle. We do not rely on their claims.

Avoid:

- Depending on a single-author, AI-assisted port for a password manager without independent tests.
- Container-based hosting if the Free plan is a goal.

## Where this project is different

| Area | Existing Workers projects | edgevault |
|---|---|---|
| Specification | Prose README, few tests | Executable scenario files that are the spec |
| Mocks | Stubbed storage in some tests | Real D1 and Durable Object in-process; oracle for expectations |
| Plan support | Free-first; paid is "just remove limits" | Explicit `free` and `paid` profiles behind ports, each tested |
| CI | Build and deploy; security scans in one project | Pyramid stages, harness guard, mutation testing, nightly contract and budget jobs |
| Free-plan limits | Documented in README | Enforced as budget tests |
