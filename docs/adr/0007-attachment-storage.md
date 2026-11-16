# ADR 0007: Attachment storage, R2 by default with KV as the no-card option

Status: proposed

## Summary

Attachments and Send files are stored in **R2** by default. People who do not want to add a payment card to their Cloudflare account can choose **KV** instead, with a smaller size limit. This choice is separate from the Workers plan: both options work on the Free plan.

## Context

- R2 has its own monthly free allowance: 10 GB of Standard storage, 1 million write operations, and 10 million read operations, regardless of Workers plan. Turning R2 on needs a payment method, because usage past the allowance is billed rather than capped ([R2 pricing](https://developers.cloudflare.com/r2/pricing/), [community answer](https://community.cloudflare.com/t/why-using-r2-free-tier-involves-giving-card-info/945179)).
- KV needs no card. On Free it allows 1 GB of storage, 25 MiB per value, 100,000 reads and 1,000 writes per day ([KV limits](https://developers.cloudflare.com/kv/platform/limits/)).
- A Worker's request body is limited to 100 MB on Cloudflare's Free and Pro zone plans ([Workers limits](https://developers.cloudflare.com/workers/platform/limits/)).
- NodeWarden uses R2 by default with an optional KV mode. warden-worker uses KV by default and R2 if configured.

## Decision

- A `BlobStore` port with two adapters, `R2BlobStore` (default) and `KvBlobStore`.
- The backend is chosen by configuration (`ATTACHMENTS_BACKEND=r2|kv`), independent of `PLAN_PROFILE`.
- Size limits come from the backend: R2 up to 100 MB per file (bounded by the request body limit), KV up to 25 MiB per file.
- Per-user and per-organization storage quotas are configurable, with defaults that stay well inside the free allowances (for example 1 GB per user on R2, 200 MB per user on KV).
- Uploads and downloads stream; the Worker never buffers a whole file in memory (128 MB limit).
- Both adapters run the same contract test file.
- Attachment metadata (size, file name ciphertext, key ciphertext) is in D1; only the encrypted bytes are in R2 or KV.

## Consequences

- The default setup asks the user to add a card. The deploy guide explains that the card is only charged past the free allowance, and how to set a billing alert.
- With KV, the 1,000 writes per day limit is not a practical problem for a family, but the 1 GB total is. The quota check returns a clear error.
- Switching backend later needs a copy job. That is documented, not automated, in the MVP.

## Open points

- Whether Cloudflare's spending limits cover R2. The Workers spending-limit docs mention Workers requests and CPU time; we have not confirmed R2. Until then, the deploy guide recommends a billing alert.
- Some users report a small charge when they first activate R2. We could not confirm whether it is a card check or a fee.

## Alternatives considered

- **KV only:** no card, but 25 MiB files and 1 GB total are tight for a family.
- **R2 only:** simplest, but excludes people who will not add a card.
- **Tie the backend to the Workers plan:** wrong, because R2 availability depends on having a card, not on the Workers plan.
