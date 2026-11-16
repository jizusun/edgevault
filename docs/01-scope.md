# 01. Scope

## Problem

Self-hosting a Bitwarden-compatible server normally means running and paying for a VPS. Existing Cloudflare-based projects solve that, but they have little automated testing (see [06-prior-art.md](06-prior-art.md)), and none of them is designed to run on both the Free and Paid Workers plans with plan-appropriate behavior.

## Goals

1. Official Bitwarden clients (web vault, browser extension, `bw` CLI, mobile) work against a Worker we deploy to our own Cloudflare account.
2. The server only ever stores encrypted vault data. It never sees master passwords or plaintext vault items.
3. It runs on the **Workers Free** plan for a personal or family vault, and takes advantage of the **Workers Paid** plan when available. One codebase, one configuration switch (see [ADR 0003](adr/0003-plan-profiles-via-adapters.md)).
4. Tests are the specification. Behavior is described as readable data and checked against real dependencies wherever possible (see [04-testing-strategy.md](04-testing-strategy.md)).
5. Deploying is one command (`wrangler deploy`) or one GitHub Actions workflow, with no manual SQL.

## MVP scope

The MVP is a **personal vault with a small number of users**, invite-only or registration-toggled.

| Area | In MVP | Milestone |
|---|---|---|
| Server discovery and health (`config`, `alive`, `now`, `version`) | yes | M0 |
| Account: prelogin, registration, profile, keys, revision date | yes | M1 |
| Login: password grant, refresh-token grant, device records | yes | M1 |
| Security stamp and token revocation on password change | yes | M1 |
| Folders: create, read, update, delete | yes | M2 |
| Vault items (ciphers): create, read, update, delete, soft-delete, restore, purge, move to folder, bulk operations | yes | M2 |
| Full sync (`/api/sync`) | yes | M2 |
| Import (Bitwarden JSON, as encrypted by the client) | yes | M2 |
| Web vault served from the same Worker (pinned upstream build) | yes | M3 |
| Rate limiting on login, registration, prelogin | yes | M3 |
| Free-plan usage budgets enforced by tests | yes | M3 |
| Scheduled cleanup of trashed items and expired tokens | yes | M3 |

## Planned after the MVP (not designed in detail here)

TOTP two-factor login, attachments (KV on Free without a card, R2 where available), Bitwarden Send, live sync over WebSocket (Durable Object), mobile push through the Bitwarden push relay, backup export to S3 or WebDAV, passkey login.

## Non-goals

- Organizations, collections, sharing, groups, policies, SSO, SCIM, emergency access, admin console. Both existing Workers projects we studied also omit these. The data model must not make them impossible to add later (users and ciphers stay separate from any future ownership table).
- Matching Vaultwarden's or Bitwarden's internal code structure.
- Running on anything other than Cloudflare.

## Acceptance criteria for the MVP

The MVP is done when all of these hold on a fresh Cloudflare account on the Free plan:

1. The pinned web vault, the Chrome or Firefox extension, and `bw` CLI can register a user, log in, create, edit, trash, restore, and delete items and folders, and see changes after a sync on a second client.
2. The subset of Bitwarden's SDK live tests that targets our MVP endpoints passes (the same oracle Vaultwarden runs; see [06-prior-art.md](06-prior-art.md)).
3. All scenario files in `specs/` pass in three runners: in-process, against `wrangler dev`, and against a deployed staging Worker.
4. The usage budgets in [04-testing-strategy.md](04-testing-strategy.md#budget-tests) pass.
5. Mutation score on the domain layer meets the threshold set in M0.
6. A client-compatibility matrix lists the exact client versions tested.

## Compatibility policy

Bitwarden ships client changes often. We pin tested client versions in the compatibility matrix, run a nightly contract job against the latest `bw` CLI and SDK, and treat a nightly failure as a bug to triage, not a blocker for unrelated PRs.

## Legal hygiene

- This is a clean-room implementation of the observable API. We read other projects to learn behavior and structure, but we do not copy code from AGPL projects such as Vaultwarden.
- We name the project without "Bitwarden" and state that it is unaffiliated.
- The web vault is an upstream build downloaded at deploy time at a pinned version. Its license must be reviewed before we bundle it anywhere (open question 4).
