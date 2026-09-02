**English** · [Português](README.pt-BR.md)

# FIAP Games

A .NET modular monolith, rearchitected into a distributed system: five backend services, a React frontend, and a Helm orchestration chart, running on local Kubernetes with event-driven messaging.

This repo — `documentation` — is the project's specs, decision record, and narrative documentation. It's read on GitHub, not cloned: unlike the seven repos below, it's never checked out as a sibling to run anything, since nothing about running the system needs it on disk (`notes.md` 50).

## The distributed system

Seven independent repos under [`github.com/tc2-fiap`](https://github.com/tc2-fiap), each cloned as a flat sibling of the *other six* — never of this repo. Every one has its own bilingual `README.md`.

| Repo | Owns |
|---|---|
| [`users-api`](https://github.com/tc2-fiap/users-api) | Registration, login, Google sign-in, JWT issuance, roles (Player/Admin) |
| [`catalog-api`](https://github.com/tc2-fiap/catalog-api) | The game catalog — product reference data only, outside the purchase event flow |
| [`orders-api`](https://github.com/tc2-fiap/orders-api) | The `Order` aggregate, the purchase lifecycle, the library, the per-order audit log |
| [`payments-api`](https://github.com/tc2-fiap/payments-api) | The payment gateway abstraction (deterministic simulated gateway by default), persisted payment records |
| [`notifications-api`](https://github.com/tc2-fiap/notifications-api) | Welcome emails and purchase confirmations — console by default, real delivery via Resend optionally |
| [`frontend`](https://github.com/tc2-fiap/frontend) | The React app — catalog with cover images, a real checkout step (PIX QR when a real gateway is active), library, an admin section for the cross-service audit trail, and an English/Portuguese language toggle that shows R$ or a live-converted USD price |
| [`orchestration`](https://github.com/tc2-fiap/orchestration) | The Helm umbrella chart: Postgres, RabbitMQ, Ingress, and every service subchart |

To bring the whole system up on a local Kubernetes cluster, clone the seven repos above into one empty parent directory (keeping their default folder names — `orchestration`'s Helm chart expects the other six as literal sibling paths), then from `orchestration/`:

```bash
helm dependency update
helm install fiap-games .
```

This brings up all eight pods (five backend services + Postgres + RabbitMQ + frontend) from a clean cluster with zero restarts, reachable through one Ingress base URL. See [`GETTING_STARTED.en-US.md`](getting-started/GETTING_STARTED.en-US.md) ([pt-BR](getting-started/GETTING_STARTED.pt-BR.md)) for the full clone step, prerequisites, verification, and a demo walkthrough.

Each service repo also runs standalone via its own `docker-compose.yml` — see that repo's own `README.md`.

## The reference monolith

`base-project` — the original .NET 10 modular monolith (Users + Games, JWT, MongoDB, Docker Compose, CI/CD, Terraform) this system replaces — lives at [`github.com/KainanGuerra/fiap-games`](https://github.com/KainanGuerra/fiap-games), read-only reference material, not modified during this project. See that repo's own [`docs/DOCUMENTATION.md`](https://github.com/KainanGuerra/fiap-games/blob/main/docs/DOCUMENTATION.md) for how it's built, and its `README.md` for how to run it.

## The design documents

This repo holds every spec, decision record, and narrative document for the project (`notes.md` 34, 44, 46, 50, 55):

| Document | Contents | Languages |
|---|---|---|
| [`DISCOVERIES.en-US.md`](discovers/DISCOVERIES.en-US.md) / [`.pt-BR`](discovers/DISCOVERIES.pt-BR.md) | Concepts that came up while building this, traced to where they actually originated | English, Português |
| [`instructions.md`](spec/instructions.md) | The spec — architecture, service responsibilities, event contracts, Kubernetes requirements, acceptance criteria (checked off as they were verified) | English only |
| [`notes.md`](spec/notes.md) | Decision record — why each choice was made, what was rejected, and what would reopen it | English only |
| [`bdd.md`](spec/bdd.md) | Gherkin acceptance scenarios covering the event flows, order lifecycle, and deployment shape | English only |
| [`OVERVIEW.en-US.md`](context/OVERVIEW.en-US.md) / [`.pt-BR`](context/OVERVIEW.pt-BR.md) | Why this system exists and what it delivers | English, Português |
| [`ARCHITECTURE.en-US.md`](architecture/ARCHITECTURE.en-US.md) / [`.pt-BR`](architecture/ARCHITECTURE.pt-BR.md) | How it was actually built — solution architecture, event-flow diagrams, per-service breakdown, deployment | English, Português |
| [`GETTING_STARTED.en-US.md`](getting-started/GETTING_STARTED.en-US.md) / [`.pt-BR`](getting-started/GETTING_STARTED.pt-BR.md) | Prerequisites, cluster bring-up, verification, demo walkthrough | English, Português |
| [`TEST_COVERAGE.en-US.md`](test-coverage/TEST_COVERAGE.en-US.md) / [`.pt-BR`](test-coverage/TEST_COVERAGE.pt-BR.md) | Measured per-service line coverage | English, Português |
| [`frontend/design/`](https://github.com/tc2-fiap/frontend/tree/main/design) | Brand identity — color tokens, wordmark, logo mark, favicon — applied verbatim in the frontend, kept in the one repo that uses it (`notes.md` 44) | English only |

`instructions.md` is the *what*. `notes.md` is the *why*. `bdd.md` is *how you'd know it works*. `OVERVIEW.md` is *why it exists*, `ARCHITECTURE.md` is *what actually got built*. `instructions.md`/`notes.md`/`bdd.md` stay single-language — they're a spec and an append-only decision log written during the build, not reader-facing narrative documentation (`notes.md` 35).

### Feature build notes

[`features/`](features/) started as specs for work that hadn't been built yet (`notes.md` 37) — both have since been implemented, with their status banners updated in place rather than left stale. They're kept as build notes and provider-API reference (routes, status-vocabulary mapping, what's verified live vs. still pending) rather than folded into `ARCHITECTURE.md`, since its reader doesn't need raw third-party request/response shapes. English only.

| Document | Status |
|---|---|
| [`features/quotation-feature.md`](features/quotation-feature.md) | **Implemented** (`notes.md` 39) — `GET /api/quotations/usd-brl` in `catalog-api`, Frankfurter with an ExchangeRate-API fallback, display-only |
| [`features/payment-gateway-simulate.md`](features/payment-gateway-simulate.md) | **Implemented, live sandbox verification pending** (`notes.md` 38) — `AbacatePayGateway`/`MercadoPagoGateway` behind an ordered `PaymentGatewayChain`. Two confirmation paths exist: an exponential-backoff polling `BackgroundService`, the active one in this local topology (no public Ingress for a provider to call back to); and a webhook receiver (`POST /api/payments/webhooks/{provider}`, HMAC-verified) that's fully built and wired for a real deployment, just dormant here. All gateway HTTP calls are mocked in tests — nothing has called a real provider yet. |

## Notable design choices

A few that differ from the obvious reading of the requirements, each argued in `notes.md`:

- **A dedicated OrdersAPI.** The original brief put purchase initiation in the catalog; a catalog is product reference data, an order is a transactional aggregate. They're separated here, deliberately.
- **PostgreSQL, not MongoDB.** A change from the reference project, driven by needing real transactions for a transactional outbox.
- **One database, one schema and role per service.** Boundaries enforced by Postgres grants rather than developer discipline — verified live, not just asserted.
- **A swappable payment gateway.** Deterministic simulation by default (`notes.md` 4); `AbacatePayGateway` and `MercadoPagoGateway` are now built too, composed into an ordered fallback chain behind the `PaymentGateway:Providers` ConfigMap value — live sandbox verification against a real provider account is the one thing still pending (`notes.md` 38).
- **An admin audit trail composed at the view layer.** An admin can see every order, its events, its payment record, and its notifications — each service exposing only its own data over its own admin endpoint, never a cross-schema query.
- **Google sign-in without an OAuth redirect flow.** ID-token verification needs no public callback URL, sidestepping the tunnel problem that keeps the real payment gateway optional.
- **Documentation lives in its own repo, published separately and never cloned alongside the seven runtime repos** — its narrative layer (this file, `OVERVIEW.md`, `ARCHITECTURE.md`, `GETTING_STARTED.md`, `TEST_COVERAGE.md`, every repo's `README.md`) is bilingual English/Portuguese; the spec and decision record stay English-only (`notes.md` 34, 35, 44, 50).

## Context

Academic project (FIAP). The original requirements brief is in Portuguese; this documentation is bilingual (English/Portuguese) for its narrative layer, English-only for the technical spec and decision record.
