**English** · [Português](OVERVIEW.pt-BR.md)

# FIAP Games — Overview

## 1. Introduction

This document describes what was actually built when [`base-project/`](https://github.com/KainanGuerra/fiap-games) — a .NET modular monolith managing Users and Games — was rearchitected into a distributed system: five independently deployable services, an event-driven purchase flow, per-service data isolation, and a React frontend, all running on a local Kubernetes cluster.

The focus here is the same as [`base-project/docs/DOCUMENTATION.md`](https://github.com/KainanGuerra/fiap-games/blob/main/docs/DOCUMENTATION.md)'s: not just *what* was built, but *how*, and why it took the shape it did. The rest of that story — the technical deep-dive — lives one door over, in `architecture/`.

### Navigation

| File | Content |
|---|---|
| [`ARCHITECTURE.en-US.md`](../architecture/ARCHITECTURE.en-US.md) | How it's built — solution architecture, domain model, event flows, RBAC, deployment |
| [`GETTING_STARTED.en-US.md`](../getting-started/GETTING_STARTED.en-US.md) | Prerequisites, cluster bring-up, verification, a demo walkthrough |
| [`instructions.md`](../spec/instructions.md) | The spec — architecture, service responsibilities, event contracts, acceptance criteria |
| [`notes.md`](../spec/notes.md) | Decision record — 57 entries, each with the rejected alternative and what would reopen it |
| [`bdd.md`](../spec/bdd.md) | Gherkin acceptance scenarios — the project's acceptance layer |
| [`TEST_COVERAGE.en-US.md`](../test-coverage/TEST_COVERAGE.en-US.md) | Measured per-service line coverage |
| [`frontend/design/`](https://github.com/tc2-fiap/frontend/tree/main/design) | Brand identity — color tokens, wordmark, logo mark, favicon (applied verbatim in the frontend) |
| [`base-project/docs/DOCUMENTATION.md`](https://github.com/KainanGuerra/fiap-games/blob/main/docs/DOCUMENTATION.md) | How the reference monolith this system replaces is built |

## 2. Why a distributed system, and how it was built

`base-project` already applied DDD (bounded contexts), Clean Architecture, and BDD within a single deployable. This project keeps all three practices but changes the *boundary*: each bounded context — Users, Catalog, Orders, Payments, Notifications — becomes its own repository, schema, and deployable, communicating with the others only through events (or, for the one case where synchronous consistency actually matters, a plain HTTP call).

That's a materially larger failure surface than a monolith: five services, a message broker, per-service databases, and Kubernetes manifests, instead of one process and one database. The build order was chosen specifically to manage that risk — a **walking skeleton** first (the narrowest possible path through every piece of infrastructure — cluster, Helm, Ingress, Postgres, RabbitMQ, structured logging — proven with `users-api` and `notifications-api` alone), then breadth (completing the simple services, proving cross-service JWT trust), then the actual core of the assignment (the purchase flow), then hardening against the acceptance criteria, then late-arriving features (admin RBAC and the cross-service audit trail), then the frontend, in that order. Integrating everything only after every service was "complete" was treated as the primary risk to avoid — see `notes.md`'s risk table.

## 3. Deliverables

- Five independently deployable, independently tested backend services and a React frontend, each with its own Dockerfile, Helm subchart, and standalone `docker-compose.yml`.
- An event-driven purchase flow spanning three services with a transactional outbox, verified for both outcomes (`Approved`/`Rejected`) and for idempotent redelivery.
- Per-service Postgres schema isolation, enforced by role grants and verified live by a refused cross-schema query.
- An admin role with a cross-service audit trail composed from four independent endpoints, and optional Google sign-in and real email delivery, both gracefully degrading to "off" when unconfigured.
- A single-command cluster bring-up (`helm install`) verified with zero restarts from a genuinely clean state.
- A written specification (`instructions.md`), a 57-entry decision record (`notes.md`), and Gherkin acceptance scenarios (`bdd.md`).
- Bilingual (English/Portuguese) narrative documentation — this overview, `ARCHITECTURE.md`, `GETTING_STARTED.md`, `TEST_COVERAGE.md`, and every repo's `README.md` — and an English/pt-BR language toggle in the frontend itself; every price is still a BRL `decimal` end-to-end, shown as R$ or converted to a display-only USD figure depending on the toggle (`notes.md` 35, 36, 39).
- A real checkout step with a product summary, a dual-currency price, and a PIX QR code/copy-paste code when a real gateway is active — plus a duplicate-purchase guard so a game already owned or in flight can't be bought twice (`notes.md` 40, 42).
- A second admin view, `/admin/events`, listing and filtering every event/message across all five services — not scoped to one order — composed from four admin "list all" endpoints the same way the per-order audit trail is (`notes.md` 43).
- A `localStorage` cart and a `/checkout` confirmation page that both cart checkout and a catalog `Buy Now` shortcut funnel through — no purchase path skips the review step — backed by `Order` becoming a true multi-item aggregate, with the duplicate-in-order and duplicate-ownership rules now enforced as real Postgres unique indexes on `order_items`, not just application checks (`notes.md` 51, 52).
- Live order/payment status via Server-Sent Events (`GET /api/orders/{id}/stream`) instead of the client polling every two seconds (`notes.md` 53).
- The ability to remove a game from the library (freeing it up for repurchase) with a confirmation modal, without ever reversing the underlying `Paid` order (`notes.md` 54).

## 4. Conclusion

The same three practices that shaped `base-project` — DDD, Clean Architecture, BDD — carried over unchanged; what changed was the unit they were applied to, from a module inside one process to five independently deployable services trusting each other only through events and a shared JWT secret. The walking-skeleton build order, the transactional outbox, and the per-schema Postgres isolation were the three decisions that made that split survivable rather than merely aspirational — each verified live against the running system, not just asserted in a document.
