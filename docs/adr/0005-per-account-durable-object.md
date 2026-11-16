# ADR 0005: One Durable Object per account for credential checks

Status: proposed, pending M0 spikes 1 and 2

## Context

Two needs push toward a Durable Object:

1. **CPU.** On the Free plan a Worker request has 10 ms of CPU. Server-side password hashing (PBKDF2 over the client's master password hash) takes far longer. A Durable Object request has a 30 s CPU default, and Durable Objects are available on Free with SQLite storage.
2. **Strict throttling.** The Workers Rate Limiting binding is approximate. Counts are local to one Cloudflare location, they sync eventually, and a burst can exceed the limit. Cloudflare recommends a Durable Object when you need a strict limit, and advises against keying limits on IP addresses, which many users can share.

The first draft of this design used one shared `Hasher` Durable Object. Cloudflare's "Rules of Durable Objects" says not to use a single global object. Each object is single-threaded, so one shared hasher would queue every login behind a long CPU task.

## Decision

Create one Durable Object (`AccountGate`, SQLite-backed) per account.

- **Name:** `getByName(HMAC(server_secret, normalized_email))`. The id does not expose the email address in the Cloudflare dashboard or logs.
- **Methods (RPC, not `fetch`):**
  - `verifyPassword(clientHash, storedHash, salt, iterations)`: hashes, compares in constant time, records the attempt, enforces lockout.
  - `hashPassword(clientHash, iterations)`: used at registration and password change.
  - `status()`: failed-attempt counters for tests and operators.
- **State in the object:** only counters and lockout timestamps. **Credentials stay in D1.** The Worker reads the stored hash from D1 and passes it in, so there is no second copy of credentials to keep consistent.
- **Strict throttle:** failed attempts per account within a window, with exponential backoff. Counters expire through an idempotent alarm.
- **Unknown emails:** route to one of a fixed number (default 16) of shared "unknown" objects, chosen by a keyed hash of the email. They do the same amount of CPU work against a dummy hash, so response time does not reveal whether an account exists, and an attacker cannot create unlimited objects.
- **Profile behavior (see [ADR 0003](0003-plan-profiles-via-adapters.md)):**
  - `free`: hashing runs inside `AccountGate`.
  - `paid`: hashing runs inline in the Worker. `AccountGate` still does the strict throttle.
- **Coarse layer:** the Workers Rate Limiting binding stays as a cheap first filter for registration and prelogin, keyed on something other than IP when a stable key exists, and on IP with a generous limit otherwise.

## Consequences

- Each login makes one or two Durable Object RPC calls. The Free plan allows 100,000 Durable Object requests and 13,000 GB-s of compute per day. Past either limit, Durable Object operations fail, so login would fail. We document this, add a budget test, and keep the quota in the monitoring checklist.
- Two logins for the same account are serialized, which is what we want for counting failures.
- Contract tests for `PasswordHasher` run against both the Durable Object adapter and the inline adapter.
- The design needs the M0 spikes: if PBKDF2 turns out not to count against the Free CPU limit, `free` can hash inline, and `AccountGate` only does throttling.

## Alternatives considered

- **One global hasher object:** violates Cloudflare's guidance and serializes all logins.
- **A fixed pool of hasher objects:** avoids one global queue, but still gives no per-account strict throttle.
- **D1 counters for throttling:** every failed attempt would spend a write from the 100,000 daily Free allowance, and D1 has no atomic read-modify-write across requests without extra care.
- **Rate Limiting binding only:** approximate, per location, and not suitable as the only brute-force defense.
