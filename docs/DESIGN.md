# AI-meter - Architecture & Design

## 1. Overview

AI-meter is a self-hosted, single-user service that tracks usage, quota, and
subscription cost-efficiency across multiple AI/LLM provider accounts. It is the
server-side, always-on counterpart to menu-bar tools such as [CodexBar](https://github.com/steipete/CodexBar):
instead of running on a personal machine and reading local sessions, it runs as
a container in a homelab and reaches providers over the network using
user-supplied credentials.

The product goal is **subscription awareness**: given a flat monthly
subscription, show what that money actually buys in real usage terms (effective
cost per token / message / session / request), alongside live quota windows and
reset countdowns.

## 2. Goals and non-goals

### Goals

- Track quota/usage windows per provider account (percent used, window length,
  reset time) where the provider exposes them.
- Track the subscription plan per account (monthly cost, billing cycle,
  included limits) and derive effective cost-efficiency metrics over time.
- Retain all collected history indefinitely.
- Provide a web dashboard, an authenticated REST JSON API, and a Prometheus
  metrics endpoint.
- Support multiple accounts per provider and multiple credential connections
  per account (redundancy and provenance).
- Be a single, simple Docker deployment suitable for a homelab.

### Non-goals

- Multi-user or multi-tenant operation. The system has exactly one principal.
- Acting as an LLM proxy/gateway or sitting in the request path. All provider
  access is read-only polling.
- Maintaining exhaustive per-model API pricing tables in v1. Cost is derived
  primarily from the subscription model (see Section 8).

## 3. Deployment model

A single container image running a Python/FastAPI application. The application
serves the dashboard, the REST API, and the metrics endpoint, runs the polling
scheduler in-process, and persists to a SQLite database on a mounted volume.

Distributed via `docker compose` with one service, one named volume for the
database, and a small set of environment variables. The process runs as a
non-root user and exposes a healthcheck endpoint. By default the service binds
to `127.0.0.1:<PORT>`; the bind address and port are configurable through the
compose file. A configurable external base URL supports reverse-proxy
deployments.

## 4. High-level architecture

Three layers:

1. **External providers** - reached outbound only, through per-provider fetch
   strategies. AI-meter is never in the LLM request path.
2. **AI-meter core** - the provider adapter registry, the polling scheduler,
   the cost engine, the encrypted credential store, the SQLite persistence
   layer, and the HTTP layer (dashboard + REST API + metrics).
3. **Consumers** - the web dashboard, external API clients/widgets, and
   Prometheus/Grafana.

## 5. Provider adapter model

The adapter model is ported in concept from CodexBar's descriptor-driven
architecture. Provider-specific *knowledge* (endpoints, auth flows, response
parsing) is reimplemented in Python; no original source is copied.

### Descriptor

Each provider is a code-defined descriptor: stable id, display metadata and
branding, declared capabilities (quota windows, cost reporting, login flows),
the credential/strategy kinds it supports, and seeded plan-catalog entries.

### Fetch strategies

A provider declares an ordered list of fetch strategies. Each strategy:

- advertises its `kind`: `api_token`, `oauth_device`, `pasted_token`, or
  `pasted_cookie`;
- implements an availability precondition check;
- implements a fetch that returns a normalized `UsageSnapshot`;
- may declare fallback behavior.

### UsageSnapshot

The normalized result of any successful fetch:

- `primary_window`: used percent, window length, reset time, reset description
- `secondary_window`: optional second window (some providers expose several
  distinct limit types, for example session vs weekly vs model-gated message
  limits)
- `captured_at`
- `identity`: provider-reported identity (email, org, plan). Identity is
  **siloed per account** and never cross-displayed between accounts or
  providers.

### Adding a provider

Adding a provider is: add one adapter module (descriptor + strategies +
parser), add fixtures and tests, and the registry auto-discovers it. No central
wiring list.

## 6. Data model

| Entity | Origin | Cardinality | Purpose |
|---|---|---|---|
| Provider | Code catalog | static | Provider definition, capabilities, seeded plans |
| Account | User-created | many per Provider | One subscription/identity the operator holds |
| Connection | User-created | one or more per Account | One credential binding; ordered; provenance + redundancy |
| Plan | Catalog or custom | one per Account | Subscription terms used for cost math |
| Snapshot | Collected | time series per Connection | A single fetch result, retained indefinitely |
| Effective-cost rollup | Computed | per Account per billing cycle | Awareness metrics |

- **Account** carries a user label (for example `Claude Max - personal`,
  `Claude API - work`), an attached Plan, and the siloed reported identity.
- **Connection** binds one encrypted credential of a given strategy kind. It
  has a user-set priority and tracks its own health (ok / failing / expired).
- **Plan** has a type: `flat_subscription`, `usage_based`, or `hybrid`. The
  catalog is seeded with known public plans; the operator can add or edit
  custom plans for enterprise/business/custom terms.

## 7. Connection behavior

All connections on an account are polled and **all readings are stored** and
exposed via the API and history. This preserves provenance and cross-source
comparison.

Each account has a user-selected **counted connection** plus a user-editable
priority order. The dashboard headline numbers, the cost engine, and the
Prometheus metrics use the counted connection. For any cycle where the counted
connection fails, the value falls back down the priority order so that
graphs and metrics do not gap. Nothing collected is discarded; the selection
only determines what counts for awareness metrics.

Trade-off recorded: polling every connection each cycle multiplies provider
requests. This is managed through the adaptive, per-provider, configurable
polling interval (Section 9).

## 8. Cost engine

The cost engine joins snapshots with the account's plan over a billing cycle to
produce awareness metrics:

- effective cost per unit (per 1k tokens / per message / per session / per
  request, depending on what the provider exposes)
- month-to-date and end-of-cycle cost-efficiency

For `flat_subscription` plans, effective unit cost is the subscription cost
divided by measured usage volume over the cycle. For `usage_based` and `hybrid`
plans the metered/overage component is included where the provider reports it.
Maintaining broad per-model API price tables is out of scope for v1; usage-based
accounts rely on provider-reported cost where available.

## 9. Scheduler and polling

APScheduler runs in-process with one job per connection. Each provider has a
sensible default interval; intervals are overridable per account/connection
from the dashboard. Jitter avoids synchronized bursts against a provider. A
failed fetch marks connection health and retries with backoff without crashing
the scheduler. Every successful fetch is written as a Snapshot tagged with its
connection.

## 10. External API and metrics

All HTTP API endpoints, read included, require a bearer token. There is no
anonymous access; usage data is never publicly visible. Provider credentials
are never returned by any endpoint and are never rendered back into the UI
(write-only fields; only a set/last-4 indicator is shown).

Indicative surface:

- `GET /api/accounts` - accounts, plans, connection health, latest counted
  snapshot
- `GET /api/accounts/{id}/snapshots?from&to&connection=` - history; defaults to
  the counted connection, can request a specific connection or all
- `GET /api/accounts/{id}/cost` - effective-cost rollup for a billing cycle
- `GET /metrics` - Prometheus exposition: usage percent, seconds-to-reset,
  effective cost metrics, and connection-up gauges, labeled by provider,
  account, and connection

The dashboard session and the API bearer token are separate mechanisms so that
headless consumers (Grafana, widgets) authenticate without a browser session.

## 11. Authentication and security

### First-run setup

On first start, when no login exists in the database, the operator is prompted
to create the single username and password. Credentials are not provided
through environment variables.

### Local login

Single username plus password. Passwords are hashed with a strong KDF
(argon2id). Passwords are never reversibly encrypted.

### Optional OIDC

OIDC can be configured through environment variables (issuer, client id,
client secret, redirect). When configured, OIDC replaces local login entirely.
Any successfully authenticated OIDC user is granted access (single principal;
no user table, no roles). Passkey/WebAuthn is a documented future addition and
is out of scope for v1.

### API token

Auto-generated on first run, viewable and regenerable in dashboard settings,
with an optional environment override. Required on all API endpoints.

### Credential encryption

Connection secrets are encrypted at rest with authenticated symmetric
encryption using a key derived from a required environment secret
(`AIMETER_SECRET`). The application refuses to start without it. This secret is
dedicated to provider-credential encryption and is distinct in purpose from
password hashing.

### Networking

Binds `127.0.0.1:<PORT>` by default; bind and port are set through compose. A
configurable external base URL supports reverse-proxy deployments (correct OIDC
redirect URIs, cookie domain, absolute links). No multi-user capability exists
anywhere in the system.

### Backups

The two artifacts an operator must protect are the environment configuration
(specifically `AIMETER_SECRET`) and the SQLite volume.

## 12. Dashboard

The overview is an account-cards grid: each account is a tile showing its
quota windows, reset countdowns, effective-cost figures, plan, and connection
health, suitable for at-a-glance homelab use.

Shared views, identical regardless of overview styling:

- **Account detail**: history graphs, connection priority and counted-connection
  selector, plan editor.
- **Add-account wizard**: choose provider, add one or more connections
  (paste token / paste cookie / API key / OAuth device flow), assign a plan
  from the catalog or define a custom plan.
- **Settings**: per-provider polling intervals, API token management, auth
  configuration.

Card content customization (which windows/metrics a tile shows, since providers
expose multiple limit types) is an intended capability whose detailed frontend
design is deferred to a post-architecture phase.

## 13. Persistence and retention

SQLite on a mounted volume. All snapshots are retained indefinitely; history
availability is a core product property. Effective-cost rollups are computed
from snapshots and the plan; cycle rollups may be materialized for fast graphs
and metrics.

## 14. Configuration

Environment variables (compose-level), with dashboard-managed settings
persisted to the database:

- `AIMETER_SECRET` (required) - credential encryption key
- bind address / port / external base URL
- optional OIDC issuer / client id / client secret / redirect
- optional API token override

Operational settings (polling intervals, counted-connection selection, plans)
are managed in the dashboard and stored in the database.

## 15. Testing strategy

- Per adapter: recorded-fixture tests for parsing and snapshot mapping, and for
  strategy availability logic. No live network in CI.
- Core: cost-engine math, counted-connection selection and fallback,
  credential encryption round-trip, scheduler failure handling.

The fixture-based adapter tests are the safety net that allows the provider
list to grow without regressions.

## 16. v1 scope

### In scope

- Providers: Claude, ChatGPT/Codex, Cursor, GitHub Copilot, Gemini,
  Antigravity, Perplexity, plus opportunistic additions of providers that map
  cleanly onto the supported strategy kinds.
- Server-friendly credential acquisition (API key, OAuth device flow),
  user-pasted tokens, and user-pasted browser cookies, configurable via the
  dashboard.
- Single-user dashboard (local login or optional OIDC), authenticated REST API,
  Prometheus endpoint.
- Multiple accounts per provider and multiple connections per account with the
  poll-all/store-all/counted-connection behavior.

### Deferred

- Companion agent that harvests local credentials on the operator's machine
  (for providers reachable no other way).
- Passkey/WebAuthn login.
- Dashboard card-content customization (detailed frontend phase).
- Broad per-model usage-based pricing tables.
