# Spike: design-phase proofs of concept (November 2026)

## Summary

Before changing the design, I ran small experiments to replace guesses with measurements:

- **Password hashing does not fit in a Free-plan Worker.** One server-side hash takes about 100 ms of CPU. The Free limit is 10 ms. Hashing must move to a Durable Object on Free.
- **Signing tokens is cheap.** HMAC signing and checking costs about 0.015 ms, so it fits easily.
- **Real clients behave differently from older docs.** The current web vault sends the newer registration shape, calls some endpoints the design had missed, and refuses to talk to a server over plain HTTP.
- **Bun can install packages and launch our tests, but cannot run them itself.** `bun test` cannot load Cloudflare's runtime modules.
- **Both Biome and Oxlint catch an un-awaited promise**, the most common Workers bug pattern.
- **Organizations could not be captured through the browser yet.** A pop-up in the web vault blocked the form. The organization endpoint list comes from reading Vaultwarden's route table instead, and M0 must confirm it with real traffic.

Nothing here is production code. The experiments ran in throwaway folders and are not committed.

## 1. CPU cost of crypto in the Workers runtime

### Setup

- A minimal Worker run by `wrangler dev --local` (Wrangler 4.162, local `workerd`), compatibility date 2026-11-01.
- Endpoints that run Web Crypto operations N times per request. Timing is the wall time of the HTTP request from `curl` on the same machine (8 cores), so it includes a small HTTP overhead. A no-op request took about 4 ms.

### Results

| Operation | Per request (wall time) |
|---|---|
| No-op | about 4 ms |
| PBKDF2-SHA256, 5,000 iterations | 7 to 10 ms |
| PBKDF2-SHA256, 100,000 iterations | 21 to 24 ms |
| PBKDF2-SHA256, 600,000 iterations | 104 to 106 ms |
| 10 × PBKDF2-SHA256, 600,000 iterations | 976 ms |
| 1,000 × HMAC-SHA256 sign and verify | 15 ms |
| 100 × RSA-2048 verify | 7 ms |
| Node.js `pbkdf2Sync`, 600,000 iterations (reference) | 113 ms |

### What it means

- **Hashing at 600,000 iterations costs about 100 ms.** That is ten times the Free plan's 10 ms CPU limit per request.
- A Cloudflare community post measured a similar local CPU time (about 103 ms per derivation at 600,000 iterations) ([source](https://community.cloudflare.com/t/cloudflare-workers-pbkdf2-600-000-iterations-and-cpu-limits/980189)). An older community answer says the runtime counts Web Crypto CPU time ([source](https://community.cloudflare.com/t/run-node-js-for-worker/98983/10)). Cloudflare's docs say only that waiting on I/O does not count.
- Conclusion: on the Free plan, server-side hashing runs in the per-account Durable Object, which has a 30 s CPU budget ([ADR 0005](../adr/0005-per-account-durable-object.md)).
- HMAC token signing and checking is about 0.015 ms per pair, which supports [ADR 0006](../adr/0006-token-signing.md).

### Still open

- Local timing is not hosted CPU time. M0 measures the same Worker deployed on a Free account and reads CPU time from Workers Logs.
- The server-side iteration count is a separate choice from the client's. The client already does 600,000 iterations before sending its hash. A lower server-side count would be cheaper, but it weakens protection of a leaked database. This is open question 15.

## 2. What a current web vault sends

### Setup

- Vaultwarden (`vaultwarden/server:latest`, web vault "Vaultwarden Web 2026.10.1", server config version 2026.6.0) in Docker, as a reference implementation only.
- A small Node.js HTTPS proxy in front of it that recorded every `/api` and `/identity` request and response.
- Playwright (Chromium) drove the web vault: register two users, log out, log in.

### Findings

1. **The web vault refuses plain HTTP.** Over `http://localhost` it showed "Insecure URL not allowed. All URLs must use HTTPS." Our HTTP test runner must serve TLS locally.
2. **Endpoints called during register and login:**

   | Method | Path | Notes |
   |---|---|---|
   | GET | `/api/config` | Called many times |
   | POST | `/identity/accounts/register/send-verification-email` | With email sending off, the response body is a token the client uses directly in the next step |
   | POST | `/identity/accounts/register/finish` | Response `{"object":"registerFinish"}` |
   | POST | `/identity/accounts/prelogin/password` | Not `/prelogin` |
   | POST | `/identity/connect/token` | Form-encoded `grant_type=password` with `deviceType`, `deviceIdentifier`, `deviceName`, `client_id=web`, `scope=api offline_access`; later `grant_type=refresh_token` |
   | GET | `/api/sync?excludeDomains=true` | |
   | GET | `/api/accounts/revision-date` | Polled; returns a number in milliseconds |
   | POST | `/api/accounts/key-management/user-key-id` | Sent right after the first login |
   | GET | `/api/accounts/profile` | |
   | GET | `/api/devices/knowndevice` | Before login |

3. **Registration body** uses the split shape:
   `masterPasswordAuthentication {salt, kdf {kdfType, iterations}, masterPasswordAuthenticationHash}` and `masterPasswordUnlock {salt, kdf, masterKeyWrappedUserKey}`, plus `userAsymmetricKeys {publicKey, encryptedPrivateKey}` and `emailVerificationToken`.
4. **Login response** includes `AccountKeys` with `publicKeyEncryptionKeyPair`, besides the usual token fields. Access tokens lasted 7,200 s.
5. **Sync and profile response** includes, among others: `premium: true`, `accountKeys`, `userDecryption.masterPasswordUnlock`, `organizations`, `organizationsNew`, `policies`, `policiesNew`, `collections`, `sends`. `premium: true` matters: Bitwarden clients only show TOTP codes stored in items to premium users, so our server must report premium for every user.

### Organization flow (not captured)

The "New organization" page loaded, but a modal ("You can't autofill passwords without the browser extension") overlapped the form, and the automated submit stayed disabled in three attempts. I stopped there instead of fighting the UI. The organization endpoint list in [03-api-surface.md](../03-api-surface.md) comes from Vaultwarden's route table and must be confirmed with captured traffic in M0.

Vaultwarden's source shows that when email is off, an invited member is marked "accepted" straight away, and the owner then confirms them. A family can use organizations without any email.

## 3. Facts checked online for this update

| Question | Answer | Source |
|---|---|---|
| Is R2 usable without a card? | No. R2 has its own monthly free allowance (10 GB, 1 million writes, 10 million reads) regardless of Workers plan, but turning it on needs a payment method because usage past the allowance is billed | [R2 pricing](https://developers.cloudflare.com/r2/pricing/), [community answer](https://community.cloudflare.com/t/why-using-r2-free-tier-involves-giving-card-info/945179) |
| KV limits on Free | 1 GB storage, 25 MiB per value, 100,000 reads and 1,000 writes per day | [KV limits](https://developers.cloudflare.com/kv/platform/limits/) |
| Can a Free-plan Worker send email? | Not to arbitrary recipients. Sending to verified destination addresses in your own account is free on all plans | [Email Service pricing](https://developers.cloudflare.com/email-service/platform/pricing/) |
| Which test plugin? | `@cloudflare/vitest-pool-workers` is deprecated and renamed to `@cloudflare/vitest-plugin` (1.8.2, supports Vitest 4 and 5) | npm registry |
| TypeScript 7? | TypeScript 7.0.2 is out, but `typescript-eslint` accepts only `>=4.8.4 <6.1.0`, so the wider tooling may not support 7 yet | npm registry |

## 4. Bun and linters

### Bun with Workers

A minimal Worker that writes to and reads from D1, with one Vitest test using `@cloudflare/vitest-plugin` 1.8 and Vitest 5. Bun 1.4.4.

| Step | Result |
|---|---|
| `bun add -d vitest@5 @cloudflare/vitest-plugin wrangler typescript@6` | Installed 72 packages in about 25 s |
| `npx vitest run` (Node runs Vitest) | 1 test passed |
| `bun --bun vitest run` (Bun runs Vitest) | 1 test passed; the Worker and D1 still ran in `workerd` |
| `bun test` (Bun's own test runner) | Failed: `Cannot find package 'cloudflare:workers'` |
| `bunx wrangler dev`, then a request | Returned `{"n":1}` from D1 |

Conclusion: Bun can install packages and launch our tools, but its own test runner cannot run Worker tests, and production always runs in `workerd`.

### Linters on a floating promise

A sample file with an un-awaited async call and an `any`:

| Tool | Result |
|---|---|
| Oxlint 1.93 with `oxlint-tsgolint` 7.0, `--type-aware` | Reported `typescript(no-floating-promises)` and `typescript(no-explicit-any)` |
| Biome 2.6 with `noFloatingPromises` | Reported `lint/nursery/noFloatingPromises` and `lint/suspicious/noExplicitAny`; also warned that the configuration should be migrated, so the rule's config location has changed in this version |

Both catch the most important Workers bug pattern. ADR 0011 assigns type-aware rules to Oxlint.
