# 06. Prior art

## Summary

Several projects already run a Bitwarden-compatible server, and we learned from each of them:

- **Vaultwarden** is the mature, widely used reference. It runs on a normal server, so it cannot run on Cloudflare directly, but it shows how the Bitwarden API behaves and how a serious project tests and documents security.
- **NodeWarden** and **warden-worker** already run on Cloudflare Workers. They prove it can be done, and they show how to fit within the Free plan. Neither supports organizations, and both have very little automated testing.
- Two smaller Cloudflare ports exist, but they are very new or need paid features.

edgevault takes their good ideas, and adds organizations, TOTP login, and a much stronger testing approach.

What we looked at, what we measured, and what we take from each. Numbers are from the repositories and the GitHub API in November 2026. Contributor counts are for the project's own history.

## Projects

| Project | Stack | Contributors | Tests found | Notes |
|---|---|---|---|---|
| [warden-worker](https://github.com/qaz741wsd856/warden-worker) | Rust, D1, KV by default with optional R2, MIT | 7 | 2 unit tests; none run in CI | Documents Free-plan handling in detail; no organizations |
| [NodeWarden](https://github.com/shuaiplus/nodewarden) | TypeScript, D1, R2 by default with optional KV mode, LGPL-3.0 per its README | 28 | 47 cases in 7 files (Node test runner); not run in CI; CodeQL and Semgrep run | Most active; no organizations; an organizations PR has been open, unreviewed, since September 2026 |
| [Vaultwarden](https://github.com/dani-garcia/vaultwarden) | Rust, SQLite, MySQL, Postgres or CockroachDB, AGPL-3.0 | about 220 | about 67 unit tests in `src/`; Playwright end-to-end suite across databases and SSO; nightly Bitwarden SDK live tests | The reference for API behavior; cannot run on Workers. About 70,800 stars; release 1.37.5 on Nov 1, 2026; 98 of its last 100 pull requests merged |
| [vaultwarden-cloudflare](https://github.com/cybercypher/vaultwarden-cloudflare) | Rust port of Vaultwarden, Workers, D1, KV, R2 | 1 own author, AI-assisted; contributor list is inherited from upstream | README lists an integration suite; not verified | Very new; claims many features including organizations |
| [vaultwarden-cf](https://github.com/adam0white/vaultwarden-cf) | Vaultwarden in Cloudflare Containers | 1 | none found | Needs an external database; Containers are not a Free-plan option as far as we know |

## What each one teaches

### warden-worker

Take:

- **CPU offloading to a Durable Object.** The Free plan's 10 ms CPU limit is too small for server-side PBKDF2 and for large imports. Durable Objects have a 30 s CPU default, and they are available on the Free plan with SQLite storage. Our `free` profile uses the same idea behind a port.
- **Free vs card-required storage.** KV for attachments (25 MiB, no card) and R2 (100 MB, card needed), with R2 taking priority when configured. (We use R2 as the default instead, like NodeWarden; see [ADR 0007](adr/0007-attachment-storage.md).)
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
- **Client-version awareness.** Registration accepts two payload shapes (current and an older Android one), and login for "V2" accounts needs a client-version header. Its unit tests use literal payloads per client format. We turn these into `client:` rows in scenario tables.
- **Deterministic fake KDF settings for unknown emails**, chosen by a keyed hash of the email, so prelogin does not reveal which accounts exist.
- **KDF upgrade on login**: the stored server-side hash is recomputed when the configured iteration count rises.
- **Purpose-separated tokens** (login, refresh, invite, Send, download), and constant-time comparison of secrets.
- **A written threat model** with an actor table, a list of settings that remove a protection on purpose, and an explicit out-of-scope list, plus `SECURITY.md`. Both are owned through CODEOWNERS together with CI workflows. See [../THREAT_MODEL.md](../THREAT_MODEL.md).
- **CI hygiene beyond tests**: spell check (typos), GitHub Actions security lint (zizmor), Dockerfile lint, image scan, formatting and clippy across a feature matrix.
- **Open and honest scope**: it documents behavior it does not equalize, such as login timing, and rate limits kept in memory per process and IP.

Avoid:

- Copying code. It is AGPL-3.0. This project is an independent implementation and not a strict clean-room one, because Vaultwarden's source was read during design (see [01-scope.md](01-scope.md#legal-hygiene)).
- Its in-memory, per-IP rate limits. They reset on restart and are shared by users behind one address. We use a per-account Durable Object instead ([ADR 0005](adr/0005-per-account-durable-object.md)).

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
| Organizations | Not supported | Family-sized organizations and collections in the MVP, with a reviewed access table |
| Plan support | Free-first; paid is "just remove limits" | Explicit `free` and `paid` profiles behind ports; every scenario runs on both |
| Brute-force defense | In-memory per-IP limits, or the approximate edge binding | Strict per-account throttle in a Durable Object, with the binding as a coarse extra layer |
| CI | Build and deploy; security scans in one project | Pyramid stages, harness guard, mutation testing, nightly contract and budget jobs |
| Free-plan limits | Documented in README | Enforced as budget tests |
