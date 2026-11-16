# 03. API surface for the MVP

This list comes from reading the route tables of Vaultwarden (`src/api/identity.rs`, `src/api/core/*.rs`) and from the Bitwarden clients' documented behavior. It is the starting list, not the final one. **M0 turns each row into at least one scenario, and verifies it against the oracle** (see [04-testing-strategy.md](04-testing-strategy.md#validating-the-tests)). Rows the oracle disproves are corrected in the PR that finds out.

Clients call both `/api/...` and `/identity/...`. Several operations have both `POST` and `PUT` (or `DELETE`) spellings because different client versions use different ones. We implement the spellings that real clients call, and each spelling gets its own scenario so none is lost in a refactor.

## Client version differences

The API is not one fixed shape. While reading Vaultwarden's source and its unit tests (November 2026) I found these differences, and they must be handled and tested per client version:

- **Registration payloads.** Registration finish accepts at least two shapes:
  - a current shape that splits authentication and unlock data (`masterPasswordAuthentication` with a hash, salt and KDF settings, and `masterPasswordUnlock` with the wrapped user key), used by the web vault and iOS since a recent change,
  - an older flat shape (`masterPasswordHash`, `userSymmetricKey`, `kdf`, `kdfIterations`), still sent by Android before 2026.9.0.
- **"V2" accounts.** Newer clients can create accounts with an extended key-management state, including a signature key pair and a different cipher format marker. Login for those accounts requires a client-version header, and the minimum version differs by client (for example, the desktop app needs a later version than the web vault). Whether the MVP must support V2 accounts depends on what current clients create by default. This is an M0 spike and open question 11.
- **Prelogin response.** It carries the KDF settings both flat (`kdf`, `kdfIterations`, `kdfMemory`, `kdfParallelism`) and nested (`kdfSettings`), plus a `salt` that is null.
- **Multiple spellings** of the same operation (`POST` and `PUT`, `DELETE`), as noted below.

How we handle it: each payload shape and client version is a row in a scenario table with a `client:` column (for example `android 2026.8`, `web 2026.10`, `bw-cli latest`). A new client behavior becomes a new row, so it cannot be silently dropped in a refactor. These shapes come from reading another project's code and tests, so M0 confirms each one against the oracle and the Bitwarden SDK before we rely on it.

## M0: discovery and health

| Method | Path | Notes |
|---|---|---|
| GET | `/api/alive` | health, no auth |
| GET | `/api/now` | server time |
| GET | `/api/version` | server version string |
| GET | `/api/config` | feature flags and environment URLs the clients read at startup |

## M1: accounts and login

| Method | Path | Notes |
|---|---|---|
| POST | `/identity/accounts/prelogin`, `/identity/accounts/prelogin/password` | KDF settings; deterministic fake settings for unknown emails (see [02](02-architecture.md#authentication)) |
| POST | `/identity/accounts/register/send-verification-email` | new registration flow used by current clients |
| POST | `/identity/accounts/register/finish` | accepts more than one payload shape, see [client version differences](#client-version-differences) |
| POST | `/identity/connect/token` | `password` and `refresh_token` grants; creates or updates the device |
| GET | `/api/accounts/profile` | |
| PUT, POST | `/api/accounts/profile` | name and similar |
| GET | `/api/accounts/keys`, POST `/api/accounts/keys` | RSA key pair |
| GET | `/api/accounts/revision-date` | one cheap read |
| POST | `/api/accounts/verify-password` | |
| POST | `/api/accounts/password` | changes password, rotates security stamp |
| POST | `/api/accounts/security-stamp` | deauthorize sessions |
| GET | `/api/devices/knowndevice` | clients call it before login |
| GET | `/api/devices` | list sessions |

## M2: vault

| Method | Path | Notes |
|---|---|---|
| GET | `/api/sync` | profile, folders, ciphers, domains in one response |
| GET | `/api/folders`, GET `/api/folders/{id}` | |
| POST | `/api/folders` | |
| PUT, POST | `/api/folders/{id}` | |
| DELETE, POST | `/api/folders/{id}`, `/api/folders/{id}/delete` | |
| GET | `/api/ciphers`, GET `/api/ciphers/{id}` | |
| POST | `/api/ciphers`, `/api/ciphers/create` | create |
| PUT, POST | `/api/ciphers/{id}` | update |
| PUT, POST | `/api/ciphers/{id}/partial` | folder and favorite only |
| PUT, POST | `/api/ciphers/{id}/delete` | soft delete (trash) |
| DELETE | `/api/ciphers/{id}` | permanent delete |
| PUT | `/api/ciphers/{id}/restore` | |
| DELETE, POST, PUT | `/api/ciphers`, `/api/ciphers/delete` | bulk delete |
| PUT | `/api/ciphers/restore` | bulk restore |
| POST, PUT | `/api/ciphers/move` | bulk move to folder |
| POST | `/api/ciphers/purge` | delete all items (requires password) |
| POST | `/api/ciphers/import` | import of client-encrypted items and folders, chunked |
| GET, PUT | `/api/settings/domains` | equivalent domains; global list can start empty |

## M3: delivery

| Item | Notes |
|---|---|
| Web vault static assets | Served from the same Worker. API paths take priority over assets. |
| Rate limits | Strict per-account login throttle (Durable Object); coarse limit on registration and prelogin |
| Cron | Purge trash older than a configurable number of days, delete expired refresh tokens |

## Explicitly not in the MVP

Anything under organizations, collections, sends, attachments, emergency access, two-factor providers, WebAuthn, auth requests (login with device), events, admin, SSO, push, and the notifications hub. Unsupported paths return a Bitwarden-shaped error, so clients show a clear message instead of a parse failure. A scenario covers that behavior.
