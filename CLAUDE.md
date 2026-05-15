# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

It is written to be useful to any contributor or coding agent, not a specific person.

## What this repository is

AI-meter is a self-hosted, single-user service that tracks usage, quota, and
subscription cost-efficiency across multiple AI/LLM provider accounts. It is the
always-on, server-side counterpart to menu-bar tools such as CodexBar.

`docs/DESIGN.md` is the authoritative architecture and scope document. Read it
before making non-trivial changes. If an implementation decision contradicts
`docs/DESIGN.md`, update the design doc in the same change rather than letting
the two diverge.

## Project status

Pre-implementation. The repository currently contains the design document and
repository scaffolding only. There is no application code, build system, test
suite, or lint configuration yet. Do not invent build/test/lint commands or
claim they exist. When the application is scaffolded, replace this section with
the real commands (install, run, test a single test, lint, build the
container).

The planned stack is Python with FastAPI, packaged as a single Docker image
deployed via `docker compose` with SQLite on a mounted volume. Treat that as
the intended direction, not as existing fact.

## Architectural invariants

These constraints define the product. Do not weaken them without an explicit
decision recorded in `docs/DESIGN.md`:

- **Single principal.** There is no multi-user or multi-tenant capability
  anywhere. Do not add user tables, roles, tenancy, or per-user data
  partitioning.
- **Read-only provider access.** AI-meter polls providers outbound only. It is
  never an LLM proxy/gateway and never sits in the request path.
- **Provider adapter model.** Each provider is a self-contained adapter:
  a descriptor plus an ordered list of fetch strategies, each normalizing to a
  common `UsageSnapshot`. Adding a provider must not require editing a central
  wiring list. Provider-reported identity is siloed per account and never
  cross-displayed.
- **Credential handling.** Provider credentials are encrypted at rest using a
  key derived from the required `AIMETER_SECRET` environment variable. They are
  never returned by any API endpoint and never rendered back into the UI.
- **Passwords are hashed, not encrypted.** Local login passwords use a strong
  KDF (argon2id). Never store passwords with reversible encryption.
- **History is retained indefinitely.** All collected snapshots are kept;
  retention/auto-deletion is out of scope. Derived rollups may be materialized
  but never replace raw snapshots.
- **Account / Connection split.** A provider has many accounts; an account has
  one or more ordered connections. All connections are polled and stored; a
  user-selected "counted" connection (with priority fallback) determines what
  feeds headline numbers, the cost engine, and Prometheus metrics.

## Repository conventions

- Local brainstorming, planning, and superpowers session artifacts must not be
  committed. `.gitignore` enforces this (`.superpowers/`, brainstorm/plan
  directories and file patterns). Finalized design lives in `docs/DESIGN.md`,
  which is the only design artifact that belongs in version control.
- Keep `docs/DESIGN.md` and the implementation in sync within the same change.
