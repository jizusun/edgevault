# ADR 0006: Token signing and token types

Status: proposed, pending M0 spike 1

## Context

Vaultwarden signs its tokens with RS256 and uses a separate claims structure and issuer for each purpose: login, refresh, invitations, Send access, file downloads, and others. We need a smaller set for the MVP, and we have a 10 ms CPU budget per request on the Free plan.

Workers supports HMAC, RSASSA-PKCS1-v1_5, ECDSA, and Ed25519 through Web Crypto.

## Decision

- **Access tokens:** JWTs signed with **HMAC-SHA-256** using a key stored as a Workers Secret. Only this server issues and verifies them, so an asymmetric key adds CPU cost and no benefit. If a later feature needs third parties to verify tokens, we add an asymmetric key then.
- **Refresh tokens:** random 256-bit values from `crypto.getRandomValues`, stored only as SHA-256 hashes in D1, bound to a device, rotated on use. They are not JWTs.
- **Purpose separation:** every token carries a purpose claim (`login`, and later `send`, `download`, `invite`), and the verifier for each purpose rejects the others. A test table checks that a token for one purpose is rejected by every other verifier.
- **Claims for login tokens:** user id, device id, security stamp, issued-at, expiry. A changed security stamp invalidates earlier tokens.
- **Comparisons:** all secret comparisons use `crypto.subtle.timingSafeEqual`.
- **Rotation:** the secret carries a key id (`kid`). The verifier accepts the current and the previous key, so rotation does not log everyone out at once.

## Consequences

- Cheap verification suits the Free CPU limit.
- If the key leaks, an attacker can mint tokens, so the secret is only in Workers Secrets, CI secret stores, or `.dev.vars` (git-ignored), and never in config files or logs.
- The M0 CPU spike measures verification cost on the `free` profile.

## Alternatives considered

- **RS256 like Vaultwarden:** larger CPU cost per request, no benefit while only we verify tokens.
- **EdDSA or ECDSA:** cheaper than RSA, but still more work than HMAC, with the same lack of benefit.
- **Opaque tokens stored in D1:** simple revocation, but a D1 read on every request spends the Free rows-read allowance.
