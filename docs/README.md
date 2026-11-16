# edgevault design documents

Status: **proposed**, for review in the first pull request. Nothing in this folder is implemented yet.

## Summary

This folder describes **what edgevault is, how it will be built, and how it will be tested**, before any code is written.

edgevault is a password-manager server for a family. It works with the official Bitwarden apps and runs on your own Cloudflare account, on the Free or Paid plan. It supports personal vaults, family sharing through organizations, and TOTP codes, both inside vault items and as a second login step.

Every document starts with a plain-English summary like this one, then goes into detail. Diagrams use Mermaid, which GitHub renders.

Facts about Cloudflare limits were checked against the Cloudflare docs in November 2026. Re-check them before relying on any number.

| Document | Purpose |
|---|---|
| [01-scope.md](01-scope.md) | What the MVP does, what comes later, what is out, acceptance criteria |
| [02-architecture.md](02-architecture.md) | How the system is put together: Worker, D1, Durable Objects, data model, login, sync, Free vs Paid |
| [03-api-surface.md](03-api-surface.md) | The endpoints the Bitwarden apps call, per milestone, and what was observed in real traffic |
| [04-testing-strategy.md](04-testing-strategy.md) | Test pyramid, scenario files (tests as data), TDD workflow, harness protection, test validation |
| [05-delivery-plan.md](05-delivery-plan.md) | Milestones M0 to M5, spikes, PR rules, definition of done, risks |
| [06-prior-art.md](06-prior-art.md) | What we learned from Vaultwarden, NodeWarden, warden-worker, and others |
| [07-open-questions.md](07-open-questions.md) | Decisions for the project owner, with defaults |
| [08-cloudflare-practices.md](08-cloudflare-practices.md) | Cloudflare best practices and how CI enforces them |
| [09-tech-stack.md](09-tech-stack.md) | What we use, with versions, and what we deliberately do not use |
| [adr/](adr/) | Architecture decision records 0001 to 0011 |
| [spikes/](spikes/) | Results of experiments (POCs) that replaced guesses with measurements |
| [../THREAT_MODEL.md](../THREAT_MODEL.md), [../SECURITY.md](../SECURITY.md) | Threat model (draft) and vulnerability reporting |

## Decision records

| ADR | Decision |
|---|---|
| [0001](adr/0001-typescript-on-workers.md) | TypeScript on Workers |
| [0002](adr/0002-d1-as-primary-store.md) | D1 as the main store |
| [0003](adr/0003-plan-profiles-via-adapters.md) | Free and Paid plans through adapter profiles |
| [0004](adr/0004-scenario-tests-as-data.md) | Scenario files as the specification |
| [0005](adr/0005-per-account-durable-object.md) | One Durable Object per account for password and 2FA checks |
| [0006](adr/0006-token-signing.md) | HMAC-signed, purpose-separated tokens |
| [0007](adr/0007-attachment-storage.md) | Attachments in R2 by default, KV without a card |
| [0008](adr/0008-organizations.md) | Organizations and collections for families |
| [0009](adr/0009-totp.md) | TOTP in vault items and as a login second factor |
| [0010](adr/0010-drizzle-for-d1.md) | Drizzle as a typed query builder for D1 |
| [0011](adr/0011-toolchain.md) | Toolchain: Node and npm, Biome, Oxlint, Semgrep, and other static analysis |

## Reading order

Read 01, then 04 (it drives how everything else is built), then 02 and 09. The rest are reference.

Facts about Cloudflare limits were checked against the Cloudflare docs in November 2026. Re-check them before relying on any number.
