# 02. Architecture

## Constraints that shape the design

These come from the Cloudflare docs (checked November 2026).

| Constraint | Workers Free | Workers Paid | Consequence |
|---|---|---|---|
| CPU time per request | 10 ms | 30 s default, up to 5 min | Server-side password hashing cannot run inline on Free |
| Requests per day | 100,000 | no limit | Fine for a personal vault |
| Subrequests (including D1 queries) per request | 50 | 10,000 (D1: 1,000) | Use `db.batch()`; a batch counts as one query |
| D1 database size | 500 MB | 10 GB | Fine for a personal vault |
| D1 rows read per day | 5 million | included monthly allowance | Sync cost must scale with item count, and indexes are required |
| D1 rows written per day | 100,000 | included monthly allowance | Avoid write amplification (no per-request audit rows) |
| D1 row size | 2,000,000 bytes | same | Cap encrypted cipher payload size below this |
| D1 bound parameters per statement | 100 | same | Chunk bulk inserts |
| D1 Time Travel | 7 days | 30 days | Free-plan backup story |
| Durable Objects | SQLite-backed only | SQLite-backed | Use SQLite-backed classes only |
| Durable Object CPU per request | 30 s | 30 s default, up to 5 min | Heavy work can be offloaded here on Free |
| Durable Object requests and compute | 100,000 requests and 13,000 GB-s per day; operations fail past either | 1 million requests per month included, then metered | Login makes one or two calls; exhausting the quota breaks login on Free |
| KV max value | 25 MiB | 25 MiB | Attachment ceiling without R2 |
| R2 | needs a payment method | yes | Optional attachment backend |

Unverified assumptions that the M0 spike must settle before we commit (see [05-delivery-plan.md](05-delivery-plan.md)):

- Whether WebCrypto PBKDF2 with 600,000 iterations counts against the 10 ms CPU limit on Free.
- Whether the Workers Rate Limiting binding is available on Free. (Its docs list no plan restriction, but its counts are per location and approximate either way, see [ADR 0005](adr/0005-per-account-durable-object.md).)
- Which client versions create and require "V2" accounts, and which registration payload shapes real clients send today (see [03-api-surface.md](03-api-surface.md#client-version-differences)).
- Actual CPU cost of `/api/sync` for 1,000 items.

## Shape of the system

```
 Bitwarden clients                       Cloudflare account
 (web vault, extension, bw CLI, mobile)
        |  HTTPS
        v
 +--------------------------------------------------------------+
 | Worker "edgevault"                                           |
 |                                                              |
 |  HTTP layer (Hono)   routes, validation, error mapping       |
 |        |                                                     |
 |  Application layer   use cases: Register, Login, Sync, ...   |
 |        |                                                     |
 |  Domain layer        pure TypeScript, no Cloudflare imports  |
 |        |                                                     |
 |  Ports               VaultStore, UserStore, TokenStore,      |
 |                      PasswordHasher, LoginThrottle, Clock, Ids    |
 |        |                                                     |
 |  Adapters  (chosen by the composition root from PLAN_PROFILE)|
 +--------+-----------------+------------------+----------------+
          |                 |                  |
          v                 v                  v
         D1        Durable Object "AccountGate"  KV / R2 (later)
   (users, devices,  (one per account: hashing   (attachments,
    folders, ciphers) on Free, strict throttle)   after MVP)

 Static assets (web vault) are served by the same Worker through
 Workers Static Assets; /api/* and /identity/* go to the code above.
```

### Layer rules

1. The domain layer imports nothing from Cloudflare, Hono, or the standard `fetch` API. It is plain functions and types, so it runs in any test runner.
2. Application use cases depend only on ports. Each use case is one file with one exported function.
3. Adapters implement ports. There is one adapter per port per backing service. Adapters contain no business rules.
4. Only `src/composition/` knows which adapter goes with which plan profile.
5. A dependency-cruiser rule in CI enforces rules 1 to 4.

### Repository layout (target)

```
src/
  domain/          pure rules: kdf policy, EncString validation, revision dates, cipher types
  application/     use cases (one file each)
  ports/           interfaces
  adapters/
    d1/            D1 implementations of the stores
    do/            Durable Object classes (AccountGate, later Notifier)
    webcrypto/     hashing and token signing
    memory/        in-memory fakes used only for unit tests of use cases
  http/            Hono routes, request validation (zod), Bitwarden error shapes
  composition/     wires adapters per plan profile
specs/             test data: scenario files and tables (see 04-testing-strategy.md)
test/
  runner/          the scenario parser, printers, and the three runners
  unit/            tests that are better written as code
migrations/        D1 SQL migrations
docs/
```

## Plan profiles

A profile is a set of adapter choices, not a set of `if (isFree)` branches. The domain and application layers never see the profile.

```
PLAN_PROFILE=free | paid        (set in wrangler config [vars], default: free)
```

| Concern | `free` profile | `paid` profile |
|---|---|---|
| Server-side password hash (PBKDF2 over the client's master password hash) | `AccountGate` Durable Object (one per account) hashes, with a 30 s CPU budget | Inline in the Worker with `limits.cpu_ms` raised |
| Strict login throttle (failed attempts, lockout) | `AccountGate` | `AccountGate` (same on both plans) |
| Bulk import | Chunked batches, each within 50 queries and 100 bound parameters | Same code, larger chunks |
| Coarse rate limit on registration and prelogin | Workers Rate Limiting binding (approximate, per location); fallback is a Durable Object counter | Rate Limiting binding |
| Scheduled cleanup | One Cron Trigger (Free allows 5 per account) | Same |
| Attachments (after MVP) | KV, 25 MiB max, no card needed | R2 |
| Backups | D1 Time Travel (7 days), plus optional export job | Time Travel (30 days), plus export |

The two hashers implement the same `PasswordHasher` port and share one contract test file, so they cannot drift. See [ADR 0003](adr/0003-plan-profiles-via-adapters.md) and [ADR 0005](adr/0005-per-account-durable-object.md).

### Account gate (Durable Object)

One SQLite-backed Durable Object per account, named from an HMAC of the normalized email so the id does not reveal the address. It holds only counters and lockout timestamps. Credentials stay in D1; the Worker reads the stored hash from D1 and passes it in. Unknown emails go to a fixed set of shared objects that do the same work against a dummy hash. Details and alternatives are in ADR 0005.

## Data model (D1)

All vault content is ciphertext produced by the client. The server stores it opaquely and validates only its shape.

```sql
users(
  id TEXT PRIMARY KEY,                -- UUID
  email TEXT NOT NULL UNIQUE,         -- lowercased
  name TEXT,
  password_hash BLOB NOT NULL,        -- server-side hash of the client hash
  password_salt BLOB NOT NULL,
  password_iterations INTEGER NOT NULL,
  kdf_type INTEGER NOT NULL,          -- 0 = PBKDF2-SHA256, 1 = Argon2id
  kdf_iterations INTEGER NOT NULL,
  kdf_memory INTEGER, kdf_parallelism INTEGER,
  encrypted_user_key TEXT NOT NULL,   -- EncString, "key" in the register payload
  public_key TEXT, encrypted_private_key TEXT,
  security_stamp TEXT NOT NULL,
  revision_date TEXT NOT NULL,        -- changes on any vault write
  created_at TEXT NOT NULL
);

devices(
  id TEXT PRIMARY KEY, user_id TEXT NOT NULL REFERENCES users(id),
  device_identifier TEXT NOT NULL, name TEXT, type INTEGER,
  created_at TEXT NOT NULL, last_seen_at TEXT,
  UNIQUE(user_id, device_identifier)
);

refresh_tokens(
  token_hash TEXT PRIMARY KEY,        -- SHA-256 of the token, never the token
  device_id TEXT NOT NULL REFERENCES devices(id),
  expires_at TEXT NOT NULL
);

folders(
  id TEXT PRIMARY KEY, user_id TEXT NOT NULL REFERENCES users(id),
  name TEXT NOT NULL,                 -- EncString
  updated_at TEXT NOT NULL
);
CREATE INDEX folders_user ON folders(user_id);

ciphers(
  id TEXT PRIMARY KEY, user_id TEXT NOT NULL REFERENCES users(id),
  folder_id TEXT REFERENCES folders(id) ON DELETE SET NULL,
  type INTEGER NOT NULL,              -- login, note, card, identity, ssh key
  data TEXT NOT NULL,                 -- JSON of EncStrings, size-capped
  favorite INTEGER NOT NULL DEFAULT 0,
  deleted_at TEXT,                    -- soft delete (trash)
  created_at TEXT NOT NULL, updated_at TEXT NOT NULL
);
CREATE INDEX ciphers_user ON ciphers(user_id);
```

Design notes:

- No audit-log table in the MVP, because each extra write spends the 100,000 rows-per-day Free allowance.
- `ciphers.data` is size-capped in the domain layer at a value below D1's 2,000,000 byte row limit. Oversized items return a clear Bitwarden-shaped error.
- A future `organizations` and `collection_ciphers` design can add ownership without changing `ciphers` columns. This is why `user_id` is not part of the primary key.
- Migrations are plain SQL files applied by `wrangler d1 migrations apply`. The Worker does not create schema at request time.

## Authentication

- Client does the key derivation and sends a **master password hash**. The server never sees the master password.
- The server stores a second, server-side hash of that value, so a database leak does not expose the client-side hash directly. The iteration count is stored per user.
- **Hash upgrade on login:** when the configured server-side iteration count is higher than the user's stored one, the hash is recomputed and stored after a successful login. (Vaultwarden does this too.)
- Login (`POST /identity/connect/token`) supports the `password` and `refresh_token` grants in the MVP.
- Token design is in [ADR 0006](adr/0006-token-signing.md): HMAC-signed access tokens with a purpose claim, random refresh tokens stored as hashes, rotation on use, security-stamp invalidation.
- Changing the password or calling "deauthorize sessions" changes the security stamp, which invalidates existing access tokens.
- **Account existence is not revealed.** `prelogin` returns, for an unknown email, KDF settings chosen deterministically from a short list of common settings by a keyed hash of the email, so repeated calls agree with each other and look like a real account. A single fixed default would let an attacker tell unknown emails from users with non-default settings. Login for an unknown email does the same work against a dummy hash.
- Secret comparisons use `crypto.subtle.timingSafeEqual`.
- Secrets (`JWT_SIGNING_KEY`, `EMAIL_HMAC_KEY`) are Workers Secrets. They are never in `wrangler` config, logs, or test data.

## Sync

`GET /api/sync` returns the profile, folders, ciphers, and domain settings in one response, the shape the clients expect. Costs on Free:

- Query count: 3 to 4 per sync (user, folders, ciphers, domains), well under 50.
- Rows read: about one per item, because queries use `user_id` indexes. A scenario budget test asserts this (see [budget tests](04-testing-strategy.md#budget-tests)).
- Clients also call `GET /api/accounts/revision-date` to decide whether a full sync is needed. This endpoint must stay one cheap indexed read.

## Errors

Bitwarden clients parse specific error bodies, for example `{"message": "...", "validationErrors": {...}}` and the OAuth-style errors from the token endpoint. Error shapes are part of the scenarios, not hidden in the HTTP layer, and are validated against the oracle (see [04-testing-strategy.md](04-testing-strategy.md#validating-the-tests)).

## Observability and privacy

- Workers Logs with structured JSON. A log helper redacts tokens, hashes, email addresses, and anything under `data`.
- No vault content, ciphertext, or key material in logs, metrics, or error reports.
- Each deployed environment exposes `GET /alive`. Nightly staging tests record CPU time and wall time from invocation logs.

## Security checklist (design-level)

- Secrets only in Workers Secrets, CI secrets, or `.dev.vars` (git-ignored).
- Input validated at the HTTP boundary with zod. Domain functions re-check invariants.
- Parameterized D1 statements only; a lint rule forbids string-built SQL.
- Strict per-account login throttle in the account Durable Object; coarse Rate Limiting binding on registration and prelogin ([ADR 0005](adr/0005-per-account-durable-object.md)).
- Registration closed by default after the first user, controlled by a config flag.
- Dependency audit and secret scanning run in CI.
- [THREAT_MODEL.md](../THREAT_MODEL.md) and [SECURITY.md](../SECURITY.md) are part of the first implementation PR and are updated whenever a change hits a review trigger listed there.
- The Cloudflare practices checklist in [08-cloudflare-practices.md](08-cloudflare-practices.md) is enforced by CI where possible.

## Deployment

- `wrangler.jsonc` declares the Worker, D1 and Durable Object bindings, static assets, cron trigger, observability, source maps, `nodejs_compat`, and the `PLAN_PROFILE` var. Bindings are declared per environment, and deploys always pass `--env`.
- D1 migrations are applied by database name before the new code goes live, and are additive, so two Worker versions can run together.
- GitHub Actions deploys staging on merge to `main` and production on a tag, applying D1 migrations first.
- The repository will include a documented "deploy to your own account" path using only `wrangler` and one secret.
