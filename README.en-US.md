**English** · [Português](README.pt-BR.md)

# FIAP Games

Two things live here: a **finished reference implementation**, and the **distributed system that replaces it** — now built and running.

## What's built

**`base-project/`** — the original .NET 10 modular monolith (Users + Games, JWT, MongoDB, Docker Compose, CI/CD, Terraform), also mirrored publicly at [`github.com/KainanGuerra/fiap-games`](https://github.com/KainanGuerra/fiap-games). Read-only reference material; not modified. The commands below run the local copy already in this workspace:

```bash
cd base-project
cp .env.example .env
docker compose up --build      # API on :8080, Swagger on :8080/swagger
```

**`repos/`** — the distributed system it was rearchitected into: five .NET backend services, a React + Vite frontend, and an `orchestration` repo holding the Helm umbrella chart. Runs on a local `kind` Kubernetes cluster. Every repo below has its own bilingual `README.md`.

| Repo | Owns |
|---|---|
| [`users-api`](https://github.com/tc2-fiap/users-api) | Registration, login, Google sign-in, JWT issuance, roles (Player/Admin) |
| [`catalog-api`](https://github.com/tc2-fiap/catalog-api) | The game catalog — product reference data only, outside the purchase event flow |
| [`orders-api`](https://github.com/tc2-fiap/orders-api) | The `Order` aggregate, the purchase lifecycle, the library, the per-order audit log |
| [`payments-api`](https://github.com/tc2-fiap/payments-api) | The payment gateway abstraction (deterministic simulated gateway by default), persisted payment records |
| [`notifications-api`](https://github.com/tc2-fiap/notifications-api) | Welcome emails and purchase confirmations — console by default, real delivery via Resend optionally |
| [`frontend`](https://github.com/tc2-fiap/frontend) | The React app — catalog with cover images, a real checkout step (PIX QR when a real gateway is active), library, an admin section for the cross-service audit trail, and an English/Portuguese language toggle that shows R$ or a live-converted USD price |
| [`orchestration`](https://github.com/tc2-fiap/orchestration) | The Helm umbrella chart: Postgres, RabbitMQ, Ingress, and every service subchart |

```bash
cd repos/orchestration
helm install fiap-games .
```

Brings up all eight pods (five backend services + Postgres + RabbitMQ + frontend) from a clean cluster with zero restarts. The frontend and every backend are reached through one Ingress base URL. See [`GETTING_STARTED.en-US.md`](narrative/GETTING_STARTED.en-US.md) ([pt-BR](narrative/GETTING_STARTED.pt-BR.md)) for prerequisites, the full bring-up sequence, and a demo walkthrough.

Each service repo also runs standalone via its own `docker-compose.yml` — see that repo's own `README.md`.

## The design documents

This repo — `repos/documentation/` — holds every spec, decision record, and narrative document for the project (`notes.md` 34, 44):

| Document | Contents | Languages |
|---|---|---|
| [`instructions.md`](spec/instructions.md) | The spec — architecture, service responsibilities, event contracts, Kubernetes requirements, acceptance criteria (checked off as they were verified) | English only |
| [`notes.md`](spec/notes.md) | Decision record — why each choice was made, what was rejected, and what would reopen it | English only |
| [`bdd.md`](spec/bdd.md) | Gherkin acceptance scenarios covering the event flows, order lifecycle, and deployment shape | English only |
| [`DOCUMENTATION.en-US.md`](narrative/DOCUMENTATION.en-US.md) / [`.pt-BR`](narrative/DOCUMENTATION.pt-BR.md) | What was actually built — architecture, event-flow diagrams, per-service breakdown | English, Português |
| [`GETTING_STARTED.en-US.md`](narrative/GETTING_STARTED.en-US.md) / [`.pt-BR`](narrative/GETTING_STARTED.pt-BR.md) | Prerequisites, cluster bring-up, verification, demo walkthrough | English, Português |
| [`../../CLAUDE.md`](../../CLAUDE.md) | Working context for AI agent sessions | English only |
| [`../frontend/design/`](../frontend/design/) | Brand identity — color tokens, wordmark, logo mark, favicon — applied verbatim in the frontend, kept in the one repo that uses it (`notes.md` 44) | English only |

`instructions.md` is the *what*. `notes.md` is the *why*. `bdd.md` is *how you'd know it works*. `DOCUMENTATION.md` is *what actually got built*. `instructions.md`/`notes.md`/`bdd.md` stay single-language — they're a spec and an append-only decision log written during the build, not reader-facing narrative documentation (`notes.md` 35).

### Future features

[`features/`](features/) holds specs for work that hasn't been built yet — each file states its own "not implemented" status up front. English only; these aren't documentation of the delivered system (`notes.md` 37).

| Document | What it specs |
|---|---|
| [`features/payment-gateway-simulate.md`](features/payment-gateway-simulate.md) | A real AbacatePay/Mercado Pago sandbox payment gateway, behind the existing `IPaymentGateway` interface |
| [`features/quotation-feature.md`](features/quotation-feature.md) | An optional USD-equivalent price display alongside R$, informational only |

## Notable design choices

A few that differ from the obvious reading of the requirements, each argued in `notes.md`:

- **A dedicated OrdersAPI.** The original brief put purchase initiation in the catalog; a catalog is product reference data, an order is a transactional aggregate. They're separated here, deliberately.
- **PostgreSQL, not MongoDB.** A change from the reference project, driven by needing real transactions for a transactional outbox.
- **One database, one schema and role per service.** Boundaries enforced by Postgres grants rather than developer discipline — verified live, not just asserted.
- **A swappable payment gateway.** Deterministic simulation by default; an optional real AbacatePay sandbox integration behind a ConfigMap flag remains unbuilt (`notes.md` 4).
- **An admin audit trail composed at the view layer.** An admin can see every order, its events, its payment record, and its notifications — each service exposing only its own data over its own admin endpoint, never a cross-schema query.
- **Google sign-in without an OAuth redirect flow.** ID-token verification needs no public callback URL, sidestepping the tunnel problem that keeps the real payment gateway optional.
- **Documentation lives in its own repo (`repos/documentation/`), not inside any one service repo**, and its narrative layer (this file, `DOCUMENTATION.md`, `GETTING_STARTED.md`, every repo's `README.md`) is bilingual English/Portuguese — the spec and decision record stay English-only (`notes.md` 34, 35, 44).

## Context

Academic project (FIAP). The original requirements brief is in Portuguese; this documentation is bilingual (English/Portuguese) for its narrative layer, English-only for the technical spec and decision record.
