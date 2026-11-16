# 08. Cloudflare practices checklist

## Summary

Cloudflare publishes advice for building on Workers, D1, and Durable Objects. This page lists the advice that applies to us, and for each item says **how we make sure we follow it**: usually a CI check or a test, so nobody has to remember it.

The most important points: keep secrets out of code, keep each request small and fast, use indexes and batches in D1, never route everything through one Durable Object, and keep database changes backward-compatible so a deploy can roll out gradually.

Practices taken from Cloudflare's own guidance (checked November 2026), with how each one is enforced in this project. "Enforced by" names a CI check or a test where one exists, so the list does not depend on people remembering it.

Sources: [Workers best practices](https://developers.cloudflare.com/workers/best-practices/workers-best-practices/), [Rules of Durable Objects](https://developers.cloudflare.com/durable-objects/best-practices/rules-of-durable-objects/), [D1 best practices](https://developers.cloudflare.com/d1/best-practices/), [D1 migrations](https://developers.cloudflare.com/d1/reference/migrations/), [Rate Limiting binding](https://developers.cloudflare.com/workers/runtime-apis/bindings/rate-limit/), [Workers limits](https://developers.cloudflare.com/workers/platform/limits/).

## Configuration

| Practice | Enforced by |
|---|---|
| Keep `compatibility_date` current; update it on a schedule | A CI job warns when it is older than 6 months |
| Enable `nodejs_compat` | CI check on the Wrangler config. The Vitest plugin injects the flag, so tests alone would not catch a missing one |
| Enable source maps | CI check on the Wrangler config |
| Generate `Env` with `wrangler types`; do not hand-write it | CI runs `wrangler types` and fails on a diff |
| Enable Workers Logs and traces, with a sampling rate | CI check on the Wrangler config |
| Secrets only through `wrangler secret`, never in config or source | gitleaks in CI; `.dev.vars` is git-ignored |
| Bindings are not inherited across environments; declare them per environment and deploy with `--env` | Deploy script refuses to run without `--env`; the root Worker is never deployed |
| Use Workers Static Assets, not Pages | Architecture |

## Request handling

| Practice | Enforced by |
|---|---|
| Check body size before reading it (128 MB memory limit); stream when possible | Unit test for the body-limit middleware; import scenario with an oversized payload |
| No request-scoped state in module globals | Semgrep rule against module-level mutable state in `src/` |
| Always `await` or `waitUntil` promises; `waitUntil` has a 30 s limit | Oxlint type-aware `no-floating-promises` and `no-misused-promises` |
| Do not use `passThroughOnException` | Semgrep rule |
| Use bindings, not REST APIs, from inside the Worker | Code review; no Cloudflare API client in runtime dependencies |
| Use Web Crypto (`crypto.getRandomValues`, `crypto.randomUUID`); never `Math.random` for security | Semgrep rule banning `Math.random` in `src/` |
| Compare secrets with `timingSafeEqual` | Semgrep rule banning `===` on values named `hash`, `token`, `secret`, `code`; unit tests |

## D1

| Practice | Enforced by |
|---|---|
| Index every column used for filtering | Budget scenarios assert rows read |
| Use `db.batch()` to stay under the per-request query limit (50 on Free) | Budget scenarios assert query count |
| Retry write queries on transient errors, with exponential backoff and jitter. D1 already retries reads twice | Unit tests for the retry helper with fault injection |
| Run migrations by database name, not binding name | Deploy script |
| Apply migrations before the new code is live. Make schema changes additive (add, then backfill, then switch, then remove later), so old and new Worker versions can run together during a deployment | Migration lint: no `DROP` or `RENAME` in the same release as the code that stops using the thing |
| Skip read replication in the MVP. It needs the Sessions API and a bookmark that clients cannot send, so a client could read stale data right after its own write | ADR if revisited |
| Time Travel for restore: 7 days on Free, 30 on Paid | Documented in the backup guide (M5) |

## Durable Objects

| Practice | Enforced by |
|---|---|
| SQLite-backed classes only | Config check |
| Model one object per unit of coordination. Never a single global object | [ADR 0005](adr/0005-per-account-durable-object.md) |
| Deterministic ids with `getByName` | Code review |
| RPC methods, not `fetch()` | Code review |
| Initialize storage in the constructor with `blockConcurrencyWhile`, and use that call sparingly | Unit tests |
| Idempotent alarm handlers; design for unexpected shutdown | Scenario that replays an alarm and a restart |
| Use the hibernatable WebSocket API for live sync (after the MVP) | Future ADR |

## Observability

- Workers Logs and traces are on. Log fields are structured and redacted (no tokens, hashes, emails, or vault data).
- Rate-limited requests and lockouts are logged as events, so an operator can see attacks.
- A nightly job records CPU and wall time on staging and fails the budget when login, sync, or item create exceed the Free limit (except offloaded work).

## Deployment and release

- Staging deploys from `main`, production from a tag.
- Gradual deployments are used for production on plans that offer them. Because old and new versions then run at once, migrations stay additive.
- Per-PR preview environments are a candidate for the contract stage. The material I found says Durable Object storage is isolated per preview, while D1, KV, and R2 need separate bindings to be isolated. That source was a community post, so it needs a spike before we rely on it.
- Rollback is a documented command, tested on staging.
