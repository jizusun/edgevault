# ADR 0001: TypeScript on Workers

Status: proposed

## Context

The server must run on Cloudflare Workers, on the Free and Paid plans. Two realistic languages exist: TypeScript (native runtime) and Rust compiled to WebAssembly (used by warden-worker). NodeWarden uses TypeScript.

## Decision

Use TypeScript in strict mode with Hono for HTTP, zod for validation at the boundary, and Vitest with `@cloudflare/vitest-pool-workers` for tests.

## Reasons

- The in-process test runner uses the real Workers runtime, D1, and Durable Objects through Miniflare. This is the foundation of our test pyramid (see [ADR 0004](0004-scenario-tests-as-data.md)).
- No WASM build step, and no compile time in the test loop.
- Web Crypto, `fetch`, and Durable Object APIs are first-class in JavaScript.
- Test tooling we want (fast-check, Stryker, Playwright) is mature in TypeScript.

## Consequences

- Weaker compile-time guarantees than Rust. We offset this with strict TypeScript, zod parsing at boundaries, branded types for ids and EncStrings, and property-based tests.
- If the M0 spikes show the CPU cost on Free cannot be met in JavaScript for a hot path, a Rust or WASM module can sit behind the same port. The port design keeps this possible.

## Alternatives considered

- **Rust (WASM):** strong types and a fit for crypto work, but slower feedback and a heavier test setup. warden-worker shows it is viable.
- **Containers running an existing server:** not suited to the Free plan and not "fully on Cloudflare".
