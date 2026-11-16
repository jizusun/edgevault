# edgevault

## Summary

edgevault is a password-manager server for a family. It works with the official Bitwarden apps and runs entirely on your own Cloudflare account (Workers, D1, Durable Objects, R2 or KV), on the Free or Paid plan.

The MVP supports personal vaults, family sharing through organizations and collections, and TOTP: codes stored in vault items, and an authenticator-app code as a second login step.

**Status: design phase.** No code yet. The design is under review in the first pull request, and implementation starts after it is approved.

## Documentation

Start at [docs/README.md](docs/README.md). It covers scope, architecture, API surface, the testing strategy (test pyramid and tests as data), the delivery plan, prior art, open questions, Cloudflare practices, and the technology stack.

## Security

The project is unaudited. See [SECURITY.md](SECURITY.md) and [THREAT_MODEL.md](THREAT_MODEL.md).

This project is not affiliated with Bitwarden, Inc. or the Vaultwarden project.
