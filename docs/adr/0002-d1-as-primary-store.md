# ADR 0002: D1 as the primary store

Status: proposed

## Context

Vault data is relational and small per user: users, devices, folders, ciphers. We need it to work on the Free plan and to be backed up.

## Decision

Use Cloudflare D1 for all durable account and vault data. Use SQL migrations managed by `wrangler d1 migrations`. Use Durable Objects only for work that needs a bigger CPU budget or stateful connections (password hashing, later WebSocket notifications), not as the main store.

## Reasons

- Free allowances fit a personal vault: 500 MB per database, 5 million rows read and 100,000 rows written per day.
- Time Travel gives point-in-time restore: 7 days on Free, 30 days on Paid.
- SQL gives transactions through `db.batch()` and indexes that control rows read, which we assert in budget tests.

## Consequences

- 2,000,000 byte row limit: cipher payloads are capped below it, with a clear error.
- 100 bound parameters per statement: bulk operations are chunked.
- 50 queries per request on Free: use cases design their queries up front and use batches.
- A single D1 database processes queries one at a time, which is fine for personal use and a known ceiling for large multi-user deployments.
- Writes are scarce on Free (100,000 rows per day), so we avoid audit-log writes in the MVP.

## Alternatives considered

- **Durable Object SQLite per user:** good isolation and no daily row caps on the Paid plan, but a larger design change, harder backup, and more moving parts than the MVP needs. Revisit after the MVP.
- **KV:** no relational queries or transactions.
