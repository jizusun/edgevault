# ADR 0011: Toolchain for building, linting, and static analysis

Status: proposed

## Summary

- We write **TypeScript** and run the tools with **Node.js and npm**. Bun can be used as a faster package manager, but not as the test runner.
- **Biome** formats the code and runs general lint rules.
- **Oxlint**, with its type-aware engine, checks the rules that need type information, such as "a promise was not awaited", which matters a lot on Workers.
- **Semgrep** checks our own security rules, such as "never use `Math.random`" and "never compare tokens with `===`".
- Other checks: the TypeScript compiler, **knip** for unused code, **dependency-cruiser** for layer rules, and **gitleaks** for secrets.

## Context

### Bun

Bun is a JavaScript runtime and toolkit, not a separate language. In production the Worker always runs in Cloudflare's `workerd` runtime. A local POC with Bun 1.4.4 ([spike notes](../spikes/2026-11-design-pocs.md#4-bun-and-linters)) showed:

| Use | Result |
|---|---|
| Package manager (`bun add`) | works |
| Launching Vitest with `@cloudflare/vitest-plugin` (`bun --bun vitest run`) | works; the Worker still runs in `workerd` with real D1 |
| `wrangler dev` through `bunx` | works |
| Bun's own test runner (`bun test`) | fails: it cannot load `cloudflare:workers` |
| Bun-only APIs (`Bun.serve`, `bun:sqlite`) | not available inside a Worker |

Cloudflare's build service supports Bun. Cloudflare's docs, Stryker, Playwright, and dependency-cruiser are documented mainly with Node.

### Linters

| Tool | Version (Nov 2026) | Strength | Limit |
|---|---|---|---|
| Biome | 2.6 | Formatter and linter in one fast tool; caught a floating promise in the POC | Fewer type-aware rules |
| Oxlint with `oxlint-tsgolint` | 1.93 / 7.0 | Very fast; type-aware engine covers 59 of typescript-eslint's 61 type-aware rules and tracks TypeScript 7 ([Oxc docs](https://oxc.rs/docs/guide/usage/linter/type-aware)); caught a floating promise in the POC | Younger project |
| ESLint with typescript-eslint | 10.15 / 8.72 | Most rules and plugins; easiest custom rules | Slower; typescript-eslint supports only TypeScript `<6.1` |

## Decision

| Concern | Tool | Notes |
|---|---|---|
| Language | TypeScript 6.0, `strict` | Move to 7 when M0 spike 10 shows Stryker, dependency-cruiser, and knip work with it |
| Runtime for tools | Node.js LTS, pinned in `.node-version` | |
| Package manager | npm, lockfile committed | Bun is allowed locally as a faster installer if it produces the same lockfile result; CI uses npm |
| Formatting | Biome | One formatter, no Prettier |
| General lint | Biome | |
| Type-aware lint | Oxlint with `oxlint-tsgolint` | `no-floating-promises`, `no-misused-promises`, `await-thenable`, `no-unnecessary-condition`, and similar |
| Project security rules | Semgrep | No `Math.random`, no `===` on secrets, no `sql.raw()`, no `passThroughOnException`, no reading `PLAN_PROFILE` outside `src/composition/` |
| Type check | `tsc --noEmit` (and `tsgo` once on TypeScript 7) | |
| Unused code and dependencies | knip | Replaces `ts-prune`, unmaintained since 2022 |
| Layer rules | dependency-cruiser | [02-architecture.md](../02-architecture.md#layer-rules) |
| `any` creep | type-coverage | Threshold set in M0, never lowered |
| Secrets | gitleaks | |
| Dependency vulnerabilities | `npm audit` and osv-scanner | |
| Code scanning | CodeQL (GitHub) | |
| Workflows | zizmor, actionlint | |
| Spelling | typos | |
| Workers config | `wrangler types` diff, `wrangler deploy --dry-run` | [08-cloudflare-practices.md](../08-cloudflare-practices.md) |

## Consequences

- Two linters instead of one. Biome and Oxlint must not both enforce the same rule; the config assigns each rule to one tool.
- Using Oxlint instead of typescript-eslint removes the main blocker to TypeScript 7.
- Custom rules live in Semgrep, which reads like the code it matches and is easy to review.
- If a rule we need exists only as an ESLint plugin, we add ESLint for that rule alone rather than switching linters.

## Alternatives considered

- **Bun for everything:** `bun test` cannot run Worker tests, so the test pyramid would lose its in-runtime layer.
- **ESLint with typescript-eslint only:** the most mature, but slower, and it holds us on TypeScript 6.
- **Biome only:** simplest, but fewer type-aware rules.
