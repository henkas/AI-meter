# AI-meter

Self-hosted, single-user tracking of usage, quota, and subscription
cost-efficiency across multiple AI/LLM provider accounts.

AI-meter is the always-on, server-side counterpart to menu-bar tools such as
[CodexBar](https://github.com/steipete/CodexBar). Instead of running on a
personal machine and reading local sessions, it runs as a container in a
homelab and reaches providers over the network using credentials you supply.

## Why

If you pay flat monthly subscriptions across several AI providers, it is hard
to see what that money actually buys. AI-meter answers that: it tracks live
quota windows and reset countdowns, records the history, and derives effective
cost (per token / message / session / request) from your subscription cost
versus real usage. The goal is **subscription awareness**.

## Status

Early. This repository currently contains the architecture and scope design
only; application code is not yet implemented. The authoritative design is in
[`docs/DESIGN.md`](docs/DESIGN.md). Expect the design to evolve before a first
release.

## Planned capabilities (v1)

- Track quota/usage windows per provider account where the provider exposes
  them, with reset countdowns.
- Track each account's subscription plan (catalog-seeded or custom) and derive
  cost-efficiency metrics over time.
- Multiple accounts per provider and multiple redundant credential connections
  per account.
- Web dashboard, authenticated REST JSON API, and a Prometheus `/metrics`
  endpoint.
- Single-container deployment via `docker compose`, single-user (local login or
  optional OIDC).
- Initial provider targets: Claude, ChatGPT/Codex, Cursor, GitHub Copilot,
  Gemini, Antigravity, Perplexity, with more added as they map onto the
  adapter model.

Read-only by design: AI-meter polls providers and is never in your LLM request
path. It is single-user with no multi-tenant capability.

## Architecture

A Python/FastAPI application in one Docker container: a provider adapter
registry, an in-process polling scheduler, a cost engine, an encrypted
credential store, and SQLite persistence, serving the dashboard, REST API, and
metrics endpoint.

See [`docs/DESIGN.md`](docs/DESIGN.md) for the full architecture, data model,
security model, and scope.

## Contributing

The design document is the source of truth. Implementation changes and the
design document are expected to stay in sync within the same change. See
[`CLAUDE.md`](CLAUDE.md) for architectural invariants that must be preserved.

## License

[MIT](LICENSE).
