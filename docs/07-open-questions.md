# 07. Open questions

## Summary

These are decisions only the project owner can make. Each has a **default in bold**, so work can start without waiting. The most important ones right now are:

- **1. License:** still undecided.
- **12. How strictly to separate from Vaultwarden's code:** decide before M1.
- **14. Turn on private vulnerability reporting** in the GitHub settings.
- **15. How many server-side hashing rounds** to use, now that we know hashing costs about 100 ms.

Questions that have been answered are listed at the end, with the answer.

## Open

| # | Question | Options | Default and reason |
|---|---|---|---|
| 1 | Project license | MIT, Apache-2.0, AGPL-3.0, LGPL-3.0 | **Undecided; no LICENSE file yet.** A public repo without a license is "all rights reserved", the safest state until you choose. We do not copy AGPL code, but Vaultwarden's source was read during design (question 12), so a permissive license carries more risk than usual. Get advice if unsure. |
| 3 | Project name | `edgevault` or another | **`edgevault`.** Avoids "Bitwarden" in the name. Renaming is cheap now. |
| 4 | Web vault source | Pinned upstream build downloaded at deploy, build from source, or none | **Pinned upstream build downloaded at deploy time.** Its license and redistribution terms need a review before we publish any bundled copy. |
| 5 | Registration policy | Open, invite-only, first user only plus a flag | **First user may register; after that, only emails invited to an organization, unless a flag opens registration.** Fits a family. |
| 6 | Mobile push | Bitwarden push relay (needs an installation id), or none | **None in the MVP.** Listed after the MVP. |
| 7 | Mutation-testing threshold | Percent | **Set in M0 from the first measurement; never lowered.** |
| 8 | Oracle strictness | Fail the nightly job on any difference, or triage | **Triage.** Vaultwarden can differ from official Bitwarden. |
| 9 | Contributions | Accept outside PRs now, or after M5 | **After M5.** Keeps the harness stable while it is young. |
| 10 | Minimum supported client versions | Exact versions | **Chosen in M0** from the latest stable web vault, extension, and `bw` CLI. |
| 11 | "V2" accounts in the MVP | V1 only, or V1 and V2 | **Decided by M0 spike 7.** If current clients create V2 accounts by default, V2 moves into M1. |
| 12 | Separation from Vaultwarden's source | Independent implementation (current plan), or strict clean-room with a separate implementer | **Independent implementation** ([01-scope.md](01-scope.md#legal-hygiene)). Decide before M1. |
| 14 | Private vulnerability reporting | Turn on GitHub's feature | **Turn it on.** [SECURITY.md](../SECURITY.md) tells reporters to use it. It is a repository setting, so the owner switches it on. |
| 15 | Server-side hashing rounds | 600,000 (same as client default), lower (for example 100,000), or configurable | **Configurable, default 600,000.** About 100 ms per login, done in the Durable Object on Free. A lower count is cheaper but weakens protection of a leaked database, because the client already hashed with its own rounds before sending. |
| 16 | Attachments in the MVP | After the MVP (current plan), or in M3 | **Right after the MVP.** The storage design is ready ([ADR 0007](adr/0007-attachment-storage.md)), but organizations and TOTP come first. |
| 17 | Email | None (current plan), Cloudflare Email Service to verified addresses on Free, any recipient on Paid | **None in the MVP.** Invites work without email. Email is an option after the MVP. |
| 18 | TypeScript version | 6.0 (current plan) or 7.0 | **6.0** until M0 spike 10 shows the toolchain supports 7. |

## Answered

| # | Question | Answer |
|---|---|---|
| 2 | Language | **TypeScript** ([ADR 0001](adr/0001-typescript-on-workers.md)). |
| 13 | Token signing | **HMAC-SHA-256** ([ADR 0006](adr/0006-token-signing.md)). The POC measured about 0.015 ms per sign and verify. |
| 19 | Organizations and TOTP in the MVP | **Yes**, both, as asked by the owner for family use ([ADR 0008](adr/0008-organizations.md), [ADR 0009](adr/0009-totp.md)). |
| 20 | Default attachment storage | **R2 by default, KV as the no-card option**, as asked by the owner ([ADR 0007](adr/0007-attachment-storage.md)). |
