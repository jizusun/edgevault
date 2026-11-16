# 03. API surface for the MVP

## Summary

This page lists the web addresses (endpoints) the Bitwarden apps call, grouped by milestone. edgevault has to answer each of them the way the apps expect.

The list comes from three places:

- **traffic we recorded** from a real web vault talking to Vaultwarden ([POC notes](spikes/2026-11-design-pocs.md)),
- **Vaultwarden's route table**, as a reference,
- Bitwarden's public behavior.

Rows marked **seen** were observed in recorded traffic. Rows marked **ref** come only from the route table and must be confirmed with recorded traffic in M0. Every row becomes at least one test scenario.

Two things make this harder than a fixed list: **different app versions send different request shapes**, and the web vault **only talks to a server over HTTPS**.

## Client version differences

The API is not one fixed shape. Each difference below is handled as rows in a scenario table with a `client:` column (for example `android 2026.8`, `web 2026.10`, `bw-cli 2026.10`), so a refactor cannot silently drop one.

| Difference | What we know | Source |
|---|---|---|
| Registration body | Current web vault sends the split shape: `masterPasswordAuthentication {salt, kdf, masterPasswordAuthenticationHash}`, `masterPasswordUnlock {salt, kdf, masterKeyWrappedUserKey}`, `userAsymmetricKeys`, `emailVerificationToken` | seen (web 2026.10.1) |
| Older registration body | Flat shape (`masterPasswordHash`, `userSymmetricKey`, `kdf`, `kdfIterations`), sent by Android before 2026.9.0 | ref (Vaultwarden unit tests) |
| "V2" accounts | Newer clients can create accounts with a signature key pair and a new item format. Login for them needs a client-version header with a per-client minimum version | ref; M0 spike, open question 11 |
| Prelogin path | Web vault calls `/identity/accounts/prelogin/password`, not `/prelogin` | seen |
| Prelogin response | KDF settings both flat (`kdf`, `kdfIterations`, `kdfMemory`, `kdfParallelism`) and nested (`kdfSettings`), with `salt: null` | seen |
| Login response | Includes `AccountKeys.publicKeyEncryptionKeyPair` besides token fields | seen |
| Email off | `send-verification-email` returns a token in the body, which the client uses directly | seen |
| HTTPS only | The web vault rejects a server URL that is not HTTPS | seen |
| Spellings | Many operations accept `POST` and `PUT` (or `DELETE`); each spelling gets its own scenario | ref |

## M0: discovery and health

| Method | Path | Source | Notes |
|---|---|---|---|
| GET | `/api/alive` | seen | health, no auth |
| GET | `/api/now` | ref | server time |
| GET | `/api/version` | ref | server version string |
| GET | `/api/config` | seen | environment URLs, feature flags, `disableUserRegistration`; called many times |

## M1: accounts and login

| Method | Path | Source | Notes |
|---|---|---|---|
| POST | `/identity/accounts/prelogin/password`, `/identity/accounts/prelogin` | seen, ref | deterministic fake settings for unknown emails |
| POST | `/identity/accounts/register/send-verification-email` | seen | returns a token when email is off |
| POST | `/identity/accounts/register/finish` | seen | both payload shapes |
| POST | `/identity/connect/token` | seen | form-encoded; `password` and `refresh_token` grants; device fields |
| GET | `/api/devices/knowndevice` | seen | before login |
| POST | `/api/accounts/key-management/user-key-id` | seen | sent after the first login |
| GET | `/api/accounts/profile` | seen | includes `premium: true` |
| PUT, POST | `/api/accounts/profile` | ref | |
| GET, POST | `/api/accounts/keys` | ref | RSA key pair |
| GET | `/api/accounts/revision-date` | seen | polled; milliseconds; one indexed read |
| POST | `/api/accounts/verify-password` | ref | |
| POST | `/api/accounts/password` | ref | rotates security stamp |
| POST | `/api/accounts/security-stamp` | ref | deauthorize sessions |
| GET | `/api/devices` | ref | list sessions |
| GET | `/api/users/{id}/public-key` | ref | used when confirming organization members (M3) |

## M2: personal vault

| Method | Path | Source | Notes |
|---|---|---|---|
| GET | `/api/sync` (often `?excludeDomains=true`) | seen | profile, folders, items, collections, organizations, policies |
| GET | `/api/folders`, `/api/folders/{id}` | ref | |
| POST | `/api/folders` | ref | |
| PUT, POST | `/api/folders/{id}` | ref | |
| DELETE, POST | `/api/folders/{id}`, `/api/folders/{id}/delete` | ref | |
| GET | `/api/ciphers`, `/api/ciphers/{id}` | ref | |
| POST | `/api/ciphers`, `/api/ciphers/create` | ref | |
| PUT, POST | `/api/ciphers/{id}` | ref | update |
| PUT, POST | `/api/ciphers/{id}/partial` | ref | folder and favorite only |
| PUT, POST | `/api/ciphers/{id}/delete` | ref | trash |
| DELETE | `/api/ciphers/{id}` | ref | permanent delete |
| PUT | `/api/ciphers/{id}/restore` | ref | |
| DELETE, POST, PUT | `/api/ciphers`, `/api/ciphers/delete` | ref | bulk delete |
| PUT | `/api/ciphers/restore` | ref | bulk restore |
| POST, PUT | `/api/ciphers/move` | ref | bulk move |
| POST | `/api/ciphers/purge` | ref | requires password |
| POST | `/api/ciphers/import` | ref | chunked |
| GET, PUT | `/api/settings/domains` | ref | equivalent domains |

## M3: organizations and collections

All rows are **ref** until M0 captures the web vault's organization flow ([ADR 0008](adr/0008-organizations.md)).

| Method | Path | Notes |
|---|---|---|
| POST | `/api/organizations` | create; body carries the org key encrypted for the owner and the org key pair |
| GET, PUT, POST | `/api/organizations/{orgId}` | read and rename |
| DELETE | `/api/organizations/{orgId}` | owner only, requires password |
| POST | `/api/organizations/{orgId}/leave` | |
| GET | `/api/organizations/{orgId}/public-key`, `/keys` | org key pair |
| GET | `/api/organizations/{orgId}/users`, `/users/mini-details` | member list |
| POST | `/api/organizations/{orgId}/users/invite` | by email; accepted on registration when email is off |
| POST | `/api/organizations/{orgId}/users/{memberId}/accept` | |
| POST | `/api/organizations/{orgId}/users/{memberId}/confirm`, `/users/confirm` | owner or admin sends org key encrypted for the member |
| POST | `/api/organizations/{orgId}/users/public-keys` | public keys of members to confirm |
| GET, PUT | `/api/organizations/{orgId}/users/{memberId}` | role, collection access |
| PUT | `/api/organizations/{orgId}/users/{memberId}/revoke`, `/restore` | and bulk forms |
| DELETE | `/api/organizations/{orgId}/users/{memberId}`, `/users` | remove |
| POST | `/api/organizations/{orgId}/users/{memberId}/reinvite` | |
| GET | `/api/collections` | collections the user can see |
| GET, POST | `/api/organizations/{orgId}/collections` | list, create |
| GET | `/api/organizations/{orgId}/collections/details`, `/{colId}/details`, `/{colId}/users` | |
| PUT | `/api/organizations/{orgId}/collections/{colId}` | rename, access |
| DELETE | `/api/organizations/{orgId}/collections/{colId}` | |
| POST | `/api/organizations/{orgId}/collections/bulk-access` | |
| POST, PUT | `/api/ciphers/{id}/share`, `/api/ciphers/share` | move a personal item into the organization |
| PUT, POST | `/api/ciphers/{id}/collections`, `/collections_v2` | change an item's collections |
| POST | `/api/ciphers/bulk-collections` | |
| GET | `/api/ciphers/organization-details`, `/organization-details/assigned` | admin view |
| POST | `/api/ciphers/import-organization` | |
| GET | `/api/organizations/{orgId}/policies` | returns an empty list (policies not supported) |
| GET | billing and plan endpoints the web vault calls | fixed self-hosted values; exact list from M0 capture |

## M4: login two-factor (TOTP)

All rows are **ref** until M0 captures the flow ([ADR 0009](adr/0009-totp.md)).

| Method | Path | Notes |
|---|---|---|
| GET | `/api/two-factor` | which providers are on |
| POST | `/api/two-factor/get-authenticator` | new secret for setup (requires password) |
| POST, PUT | `/api/two-factor/authenticator` | turn on with a valid code |
| DELETE | `/api/two-factor/authenticator` | turn off (requires password) |
| POST | `/api/two-factor/get-recover` | show the recovery code (requires password) |
| POST | recovery-code login path | exact path from M0 capture |
| POST | `/identity/connect/token` | 2FA fields (`twoFactorToken`, `twoFactorProvider`, `twoFactorRemember`); exact names from M0 capture |

## M5: delivery

| Item | Notes |
|---|---|
| Web vault static assets | Served from the same Worker; API paths take priority |
| Rate limits | Strict per-account throttle (Durable Object); coarse limit on registration and prelogin |
| Cron | Purge old trash, expired refresh tokens, stale invites |

## Not in the MVP

Sends, attachments, emergency access, other two-factor providers, WebAuthn, login with device (auth requests), events, admin, SSO, groups, policies (beyond an empty list), push, and the notifications hub. Unsupported paths return a Bitwarden-shaped error, so clients show a clear message instead of failing to parse. A scenario covers that behavior.
