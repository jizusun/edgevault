# 01. Scope

## Summary

edgevault is a password-manager server for a **family**. It works with the official Bitwarden apps and runs entirely on your own Cloudflare account, on the Free plan or the Paid plan.

The first release (the MVP) lets a family:

- create accounts and log in, with an optional **authenticator-app code (TOTP)** as a second step,
- keep a **personal vault** of passwords, notes, cards, and identities,
- **share** passwords with each other through a family **organization** and its **collections**,
- see **TOTP codes** for websites inside the Bitwarden apps,
- use the Bitwarden web vault, browser extension, mobile apps, and CLI.

The server never sees anyone's master password or any password in the vault. Everything is encrypted by the apps before it reaches the server.

Attachments, Send, and live sync come right after the MVP. Business features (SSO, groups, policies, admin console) are not planned.

## Problem

Self-hosting a Bitwarden-compatible server normally means running and paying for a VPS. Existing Cloudflare-based projects avoid that, but none supports organizations, and their automated testing is thin (see [06-prior-art.md](06-prior-art.md)). None is designed to run on both the Free and Paid Workers plans with behavior suited to each.

## Goals

1. Official Bitwarden clients (web vault, browser extension, `bw` CLI, mobile) work against a Worker we deploy to our own Cloudflare account.
2. The server only ever stores encrypted vault data. It never sees master passwords, organization keys, or plaintext vault items.
3. It runs on the **Workers Free** plan for a family, and takes advantage of the **Workers Paid** plan when available. One codebase, one configuration switch (see [ADR 0003](adr/0003-plan-profiles-via-adapters.md)).
4. Tests are the specification. Behavior is described as readable data and checked against real dependencies wherever possible (see [04-testing-strategy.md](04-testing-strategy.md)).
5. Deploying is one command (`wrangler deploy`) or one GitHub Actions workflow, with no manual SQL.

## MVP scope

The MVP is a **family vault**: a handful of users, one or a few organizations, registration closed after the first user unless opened by config.

| Area | Milestone |
|---|---|
| Server discovery and health (`config`, `alive`, `now`, `version`) | M0 |
| Account: prelogin, registration (current and older payload shapes), profile, keys, revision date | M1 |
| Login: password and refresh-token grants, devices, security stamp | M1 |
| Strict per-account login throttle | M1 |
| Folders and vault items: create, read, update, delete, trash, restore, purge, move, bulk operations | M2 |
| Full sync, import of client-encrypted data | M2 |
| `premium: true` for every user, so the apps show TOTP codes stored in items | M2 |
| **Organizations:** create, invite (no email needed), accept, confirm, revoke, remove, leave, delete ([ADR 0008](adr/0008-organizations.md)) | M3 |
| **Collections:** create, rename, delete, member access (read-only, hide passwords, manage, access all) | M3 |
| **Shared items:** share into an organization, edit and delete per permissions, personal folders and favorites on shared items | M3 |
| **Login two-factor with TOTP:** setup, login, recovery code, remember device ([ADR 0009](adr/0009-totp.md)) | M4 |
| Web vault served from the same Worker (pinned upstream build) | M5 |
| Rate limits, scheduled cleanup, Free-plan budgets, compatibility matrix | M5 |

## Planned right after the MVP

| Feature | Notes |
|---|---|
| Attachments | R2 by default, KV without a card ([ADR 0007](adr/0007-attachment-storage.md)); designed already |
| Bitwarden Send | Text first, then files on the attachment store |
| Live sync over WebSocket | Hibernatable WebSockets in a Durable Object |
| Email | Invites and 2FA notices through Cloudflare Email Service (optional) |
| Emergency access | Useful for families; needs email or an in-app flow |
| Backups | Export to S3 or WebDAV, in addition to D1 Time Travel |
| Mobile push | Through the Bitwarden push relay |
| Passkey login and other 2FA providers | |

## Non-goals

- Groups, policies, SSO, SCIM, directory sync, account recovery by admins, organization API keys, event logs, an admin console, billing.
- Matching Vaultwarden's or Bitwarden's internal code structure.
- Running on anything other than Cloudflare.

## Security documents

[THREAT_MODEL.md](../THREAT_MODEL.md) and [SECURITY.md](../SECURITY.md) exist from the first commit and are kept current. The README states that the project is unaudited.

## Acceptance criteria for the MVP

The MVP is done when all of these hold on a fresh Cloudflare account on the Free plan:

1. With the pinned web vault, a Chrome or Firefox extension, and the `bw` CLI:
   - Alice registers, turns on TOTP login, logs out, and logs in with password and code.
   - Alice creates an organization, invites Bob, Bob registers, Alice confirms him.
   - Alice creates a "Family" collection, gives Bob read-only access, and shares an item into it.
   - Bob sees the item after a sync but cannot edit it. A third user, Carol, who is not a member, cannot see it.
   - Alice revokes Bob; after a sync Bob no longer receives the organization's items.
   - A login item with a TOTP secret shows a code in the client.
2. The subset of Bitwarden's SDK live tests that targets our MVP endpoints passes (the same oracle Vaultwarden runs; see [06-prior-art.md](06-prior-art.md)).
3. All scenario files in `specs/` pass in all runners: in-process, against `wrangler dev` over HTTPS, and against a deployed staging Worker.
4. The usage budgets in [04-testing-strategy.md](04-testing-strategy.md#budget-tests) pass.
5. Mutation score on the domain layer meets the threshold set in M0.
6. A client-compatibility matrix lists the exact client versions tested.

## Compatibility policy

Bitwarden ships client changes often. We pin tested client versions in the compatibility matrix, run a nightly contract job against the latest `bw` CLI and SDK, and treat a nightly failure as a bug to triage, not a blocker for unrelated PRs.

## Legal hygiene

- This is an **independent implementation**, not a strict clean-room one. While preparing this design I read parts of Vaultwarden's source (AGPL-3.0), including route tables, tests, and its threat model, and I ran its container to record the traffic of a real web vault. Implementation will be written from observed client behavior, the Bitwarden SDK test vectors, the oracle's responses, and public documentation. We do not copy or transliterate Vaultwarden code, including its constants and structure. Where we need a value that Vaultwarden also uses (for example a list of common KDF settings), we derive it from Bitwarden's documented or observed behavior and record the source in the scenario. If a stricter separation is wanted, it must be decided before M1 (open question 12).
- We name the project without "Bitwarden" and state that it is unaffiliated.
- The web vault is an upstream build downloaded at deploy time at a pinned version. Its license must be reviewed before we bundle it anywhere (open question 4).
