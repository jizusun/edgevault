# edgevault design documents

Status: **proposed**, for review in the first pull request. Nothing in this folder is implemented yet.

Facts about Cloudflare limits were checked against the Cloudflare docs in November 2026. Re-check them before relying on any number.

| Document | Purpose |
|---|---|
| [01-scope.md](01-scope.md) | Goals, MVP scope, non-goals, acceptance criteria |
| [02-architecture.md](02-architecture.md) | Runtime layout, layers, data model, auth, Free vs Paid profiles |
| [03-api-surface.md](03-api-surface.md) | Bitwarden endpoints per milestone |
| [04-testing-strategy.md](04-testing-strategy.md) | Test pyramid, tests-as-data scenarios, TDD workflow, harness protection |
| [05-delivery-plan.md](05-delivery-plan.md) | Milestones, PR slicing, CI stages, definition of done |
| [06-prior-art.md](06-prior-art.md) | What we learned from existing projects, and what we copy or avoid |
| [07-open-questions.md](07-open-questions.md) | Decisions the reviewer needs to make |
| [08-cloudflare-practices.md](08-cloudflare-practices.md) | Cloudflare best practices and how CI enforces them |
| [../THREAT_MODEL.md](../THREAT_MODEL.md), [../SECURITY.md](../SECURITY.md) | Threat model (draft) and vulnerability reporting policy |
| [adr/](adr/) | Architecture decision records |

## Reading order

Read 01, then 04 (it drives how everything else is built), then 02. The rest are reference.
