# ADR 0003: Free and Paid plans through adapter profiles

Status: proposed, pending M0 spike 1 and spike 2

## Summary

The same code runs on the **Free** and **Paid** plans. Where the plans differ (mainly how much CPU a request may use), we plug in a different implementation behind the same interface, chosen by one setting. The business rules never ask which plan they are on, and both implementations pass the same tests.

## Context

Free and Paid Workers plans differ in CPU time per request (10 ms vs 30 s default), in daily D1 caps, and in access to R2. We want one codebase that is correct on both.

## Decision

Define ports for everything whose implementation depends on the plan (`PasswordHasher`, later `BlobStore`). Per-account login throttling is the same on both plans and is described in [ADR 0005](0005-per-account-durable-object.md). The attachment backend is a separate setting that depends on whether the account has a card, not on the Workers plan ([ADR 0007](0007-attachment-storage.md)). Provide one adapter per plan profile. A composition root reads `PLAN_PROFILE` (`free` or `paid`) and wires adapters. Domain and application code never reads the profile.

Both adapters for a port run the same contract test file.

## Consequences

- No `if (plan === "free")` in business logic. A lint rule forbids reading `PLAN_PROFILE` outside `src/composition/`.
- The `free` profile pays the cost of a Durable Object call for password hashing. The `paid` profile hashes inline.
- A user can start on `free` and switch to `paid` by changing one variable and redeploying.
- Plan limits are tested as budgets, not trusted from documentation.

## Open points for the M0 spikes

- Local POC: PBKDF2 at 600,000 iterations takes about 100 ms, ten times the Free limit, so the `free` profile hashes in the Durable Object. A hosted measurement in M0 confirms it.
- If the Rate Limiting binding turns out to be unavailable on Free, the coarse first layer falls back to a counter inside a Durable Object, and we document the extra Durable Object requests. The strict per-account throttle does not depend on the binding.

## Alternatives considered

- **Free-only design, with Paid as "remove limits":** simpler, but leaves Paid capabilities (inline hashing, R2, larger imports) unused and untested.
- **Runtime feature detection:** harder to test and to reason about than explicit profiles.
