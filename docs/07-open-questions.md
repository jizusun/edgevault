# 07. Open questions

Decisions the reviewer needs to make before or during M0. Each has a default so work can start; the default is in bold.

| # | Question | Options | Default and reason |
|---|---|---|---|
| 1 | Project license | MIT, Apache-2.0, AGPL-3.0, LGPL-3.0 | **Undecided; no LICENSE file yet.** A public repo without a license is "all rights reserved", which is the safest state until you choose. We do not copy AGPL code, so any license is possible. |
| 2 | Language | TypeScript, Rust (WASM) | **TypeScript.** See [ADR 0001](adr/0001-typescript-on-workers.md). Rust has stronger compile-time checks, and warden-worker shows it works. TypeScript gives faster test feedback with Miniflare, and a runtime that is native to Workers. |
| 3 | Project name | `edgevault` or another | **`edgevault`.** Avoids "Bitwarden" in the name. Rename is cheap now. |
| 4 | Web vault source | Pinned upstream build downloaded at deploy, or build from source, or none | **Pinned upstream build downloaded at deploy time** (as warden-worker does). Its license and redistribution terms need a review before we publish any bundled copy. |
| 5 | Registration policy | Open, invite-only, first-user-only with a flag | **First user is allowed; after that closed unless a flag opens it.** Safer default for a public URL. |
| 6 | Mobile push | Use the Bitwarden push relay (needs an installation id), or none in MVP | **None in MVP.** Listed after MVP. |
| 7 | Mutation-testing threshold | Percent | **Set in M0 from the first measurement; never lower it.** |
| 8 | Oracle strictness | Fail the nightly job on any oracle difference, or triage | **Triage.** The oracle is Vaultwarden, which can differ from official Bitwarden. |
| 9 | Contributions | Accept outside PRs now, or after M3 | **After M3.** Keeps the harness stable while it is young. |
| 10 | Minimum supported client versions | Exact versions | **Chosen in M0** from the latest stable web vault, extension, and `bw` CLI at that time. |
