# ADR 0003: Free and Paid plans through adapter profiles

Status: proposed, pending M0 spike 1 and spike 2

## Context

Free and Paid Workers plans differ in CPU time per request (10 ms vs 30 s default), in daily D1 caps, and in access to R2. We want one codebase that is correct on both.

## Decision

Define ports for everything whose implementation depends on the plan (`PasswordHasher`, `RateLimiter`, later `BlobStore`). Provide one adapter per plan profile. A composition root reads `PLAN_PROFILE` (`free` or `paid`) and wires adapters. Domain and application code never reads the profile.

Both adapters for a port run the same contract test file.

## Consequences

- No `if (plan === "free")` in business logic. A lint rule forbids reading `PLAN_PROFILE` outside `src/composition/`.
- The `free` profile pays the cost of a Durable Object call for password hashing. The `paid` profile hashes inline.
- A user can start on `free` and switch to `paid` by changing one variable and redeploying.
- Plan limits are tested as budgets, not trusted from documentation.

## Open points for the M0 spikes

- If WebCrypto PBKDF2 turns out not to count against Free CPU time, the `free` profile could hash inline and the Durable Object path becomes optional.
- If the Rate Limiting binding is unavailable on Free, the `free` rate limiter uses a D1 counter with a coarse window, and we document the extra writes.

## Alternatives considered

- **Free-only design, with Paid as "remove limits":** simpler, but leaves Paid capabilities (inline hashing, R2, larger imports) unused and untested.
- **Runtime feature detection:** harder to test and to reason about than explicit profiles.
