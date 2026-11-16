# edgevault threat model

Status: **draft, written at design time**. It describes the intended MVP, not code that exists yet. It is revised in the same pull request as any change that affects it. To report a security issue, see [SECURITY.md](SECURITY.md).

The format follows the one used by Vaultwarden's threat model: an overview, expectations, an actor table, and an explicit out-of-scope list.

## Overview

edgevault is an unofficial, self-hosted server for the Bitwarden clients. It runs as a Cloudflare Worker with D1 and Durable Objects. It stores encrypted vaults and handles registration, login, folders, and items. It is not affiliated with Bitwarden, Inc. The clients come from Bitwarden. The web vault is an upstream build served by the same Worker.

## Security expectations

### Deployment

- The operator's Cloudflare account is secured (multi-factor authentication, restricted API tokens). Anyone with access to the account can read the database and Worker secrets.
- The Worker is served over HTTPS by Cloudflare on a domain the operator controls.
- Secrets are set with `wrangler secret` and never committed.

### Encryption

- Clients encrypt vault contents end to end. The server stores ciphertext it cannot decrypt and never sees the master password.
- The server enforces access to data itself and does not rely on a client to behave.
- The server **can** read what is not end-to-end encrypted: email addresses, device names and types, item types, timestamps, KDF settings, the server-side hash of the client's master password hash, refresh-token hashes, security stamps, and the token-signing secret.
- Whoever controls the Worker or the Cloudflare account can serve a modified web vault, so end-to-end encryption does not protect web vault users from them.

### Trust

Registration is closed after the first account unless the operator opens it. Account holders are still never trusted with anyone else's data.

| Actor | Can reach | Must not reach |
|---|---|---|
| Anonymous internet | Health and config endpoints, prelogin, registration (if open), login attempts, static web vault files | Any vault data; any account's existence (beyond what prelogin deliberately leaks, see below) |
| Registered user | Their own vault, folders, devices, and settings | Another user's data; raising any privilege |
| Holder of a stolen access token | What the token's user can do until it expires or the security stamp changes | Anything after the user changes their password or deauthorizes sessions |
| Holder of a stolen refresh token | A new access token, once, until the token is rotated | Reuse after rotation (the old token is rejected) |
| Operator with Cloudflare account access | Everything | No limit |
| Cloudflare as a platform | Everything stored on it | Not modeled |

### Brute force

- Failed logins are counted per account in a Durable Object and trigger increasing delays and lockout. This is strict, because the object is the single place that counts attempts for one account.
- Unknown emails are routed to a small set of shared objects that do the same work against a dummy hash, so response time and counters do not reveal whether an account exists.
- The Workers Rate Limiting binding is a coarse extra layer. Its counts are per Cloudflare location and approximate, so it is never the only defense.

## Out of scope

These are intended behavior or outside this model.

- Anything the Cloudflare account holder can do, including reading the database and secrets.
- A compromised Cloudflare account or a compromised GitHub Actions deployment pipeline. We reduce risk with least-privilege tokens and pinned actions, but we do not model it further.
- Bugs in Bitwarden's client code, including the bundled web vault. Those belong to Bitwarden. A bug our own patches add is in scope.
- Prelogin returning KDF settings for any email, with deterministic fake settings for unknown ones. Clients need this to log in.
- The server not decrypting or authenticating ciphertext. It holds no key.
- Access tokens staying valid until they expire after a client logs out. Logging out discards tokens on that device. Changing the password or deauthorizing sessions ends them everywhere.
- Rate-limit counters from the Rate Limiting binding being approximate and per location.
- Daily Free-plan quota exhaustion (D1 rows, Durable Object requests, Worker requests). It causes denial of service for the account owner. The operator can upgrade to the Paid plan.
- Log content on the operator's own Cloudflare account, beyond the redaction we apply.
- No cap on the number of items a user stores, other than D1 and per-item size limits.

## Review triggers

Update this file when a change adds an endpoint that handles credentials, adds a new actor or token type, changes how throttling works, or changes what the server can read.
