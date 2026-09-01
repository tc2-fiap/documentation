# FIAP Games — Microservices Migration Spec

## 1. Overview

[`base-project/`](https://github.com/KainanGuerra/fiap-games) is a working reference: a .NET modular monolith (Users + Games) built with Clean Architecture, DDD-flavored bounded contexts, JWT auth, the Result pattern, global error handling, structured logging, and containerization — see [`base-project/docs/DOCUMENTATION.md`](https://github.com/KainanGuerra/fiap-games/blob/main/docs/DOCUMENTATION.md).

This document specifies the next step: **splitting that same domain into a real distributed system** — five independently deployable backend services, one frontend, and one orchestration repository — communicating asynchronously over a message broker, running **locally on Kubernetes** (no cloud deployment at this stage).

`base-project` is not being discarded; it is the architectural template each new service's internals should follow (layering, validation, error handling, logging, testing approach). What changes is the boundary: instead of modules inside one process, each bounded context becomes its own repository, its own schema, its own deployable, talking to the others only through events.

Two companion documents:

| File | Content |
|---|---|
| [`bdd.md`](bdd.md) | Acceptance-level behavior in Gherkin — the same role [`base-project/docs/behavior/behavior.md`](https://github.com/KainanGuerra/fiap-games/blob/main/docs/behavior/behavior.md) plays for the monolith. |
| [`notes.md`](notes.md) | Decision record: *why* the design is this and not the alternative, including the rejected options and what would reopen each choice. |

This document is the *what*; `notes.md` is the *why*. For what was actually built against this spec, see [`DOCUMENTATION.en-US.md`](../narrative/DOCUMENTATION.en-US.md) ([pt-BR](../narrative/DOCUMENTATION.pt-BR.md)); to run it, see [`GETTING_STARTED.en-US.md`](../narrative/GETTING_STARTED.en-US.md) ([pt-BR](../narrative/GETTING_STARTED.pt-BR.md)).

## 2. What carries over from `base-project`

Each new service should reuse the patterns already proven in the reference project, adapted to its own bounded context:

- **Clean Architecture layering**: `Domain` → `Application` → `Infrastructure`, `Endpoints` as the outermost layer; dependencies point inward.
- **Result pattern** for expected failures (not found, validation, conflict) instead of exceptions for control flow.
- **Global exception handling** — unhandled errors return a generic `ProblemDetails` with a trace id, never a stack trace, and are fully logged server-side.
- **FluentValidation** at the request boundary; invalid input returns `400` with field-level errors.
- **Structured logging** (Serilog or equivalent) — one JSON summary line per request, plus named-property business-event logs.
- **JWT authentication** — `UsersAPI` is the only issuer; the other services validate the same token (shared symmetric secret) rather than re-implementing login.
- **Automated tests** for application services and validators, with mocked dependencies — no test depends on a live database, broker, or payment provider.
- **One Dockerfile per service**, multi-stage (SDK build → runtime), non-root user — same shape as [`base-project/Dockerfile`](https://github.com/KainanGuerra/fiap-games/blob/main/src/Api/FiapGames.Api/Dockerfile).

What does **not** carry over: the modular-monolith host, the shared in-process `IModule` contract, and (for now) Terraform/Azure — see [§11](#11-local-only-scope).

**Persistence changes provider, not approach.** `base-project` used MongoDB through EF Core; this system uses **PostgreSQL** through EF Core ([§13](#13-open-decisions)). The `DbContext`, repository abstractions, and Result-based service layer all carry over unchanged — only the provider swaps (`MongoDB.EntityFrameworkCore` → `Npgsql.EntityFrameworkCore.PostgreSQL`). Two things get *simpler*: schema becomes ordinary EF Core migrations instead of hand-written index migrations, and real transactions become available — which is what makes the outbox ([§10](#10-non-functional-requirements)) tractable.

## 3. Repository layout

Seven repositories, each independently buildable and deployable:

All seven live under `repos/` in this workspace.

| Repo | Contents |
|---|---|
| [`users-api`](https://github.com/tc2-fiap/users-api) | UsersAPI service + `Dockerfile` + `docker-compose.yml` + `/k8s` Helm subchart |
| [`catalog-api`](https://github.com/tc2-fiap/catalog-api) | CatalogAPI service + `Dockerfile` + `docker-compose.yml` + `/k8s` Helm subchart |
| [`orders-api`](https://github.com/tc2-fiap/orders-api) | OrdersAPI service + `Dockerfile` + `docker-compose.yml` + `/k8s` Helm subchart |
| [`payments-api`](https://github.com/tc2-fiap/payments-api) | PaymentsAPI service + `Dockerfile` + `docker-compose.yml` + `/k8s` Helm subchart |
| [`notifications-api`](https://github.com/tc2-fiap/notifications-api) | NotificationsAPI service + `Dockerfile` + `docker-compose.yml` + `/k8s` Helm subchart |
| [`frontend`](https://github.com/tc2-fiap/frontend) | React + Vite client + `Dockerfile` + `docker-compose.yml` + `/k8s` Helm subchart |
| [`orchestration`](https://github.com/tc2-fiap/orchestration) | Helm umbrella chart, RabbitMQ, PostgreSQL, namespace, Ingress, bring-up tooling |

Each repo owns the manifests that describe *itself*. The `orchestration` repo owns everything that isn't specific to one service, and composes the rest into one runnable local environment ([§9](#9-kubernetes-requirements)).

### 3.1 Two ways to run

Every service repo is runnable **standalone**, without a Kubernetes cluster — the same pattern `base-project` already uses.

| | `docker compose up` in a service repo | `helm install` in `orchestration` |
|---|---|---|
| **Scope** | One service, alone | The whole system |
| **Postgres** | A container in that repo's compose, holding only that service's schema | One shared instance, five schemas, five roles |
| **RabbitMQ** | A container in that repo's compose, where the service publishes or consumes | One shared broker |
| **For** | Developing, debugging, and testing a single service | Integration, the event cascade, the demo |

The two environments never run against the same database. A service reads identical configuration keys in both — only the connection strings differ, and both come from environment/Secret, never from code. Keeping each repo independently runnable is also what makes it reviewable on its own.

## 4. Microservices — responsibilities

### 4.1 UsersAPI
Registration, authentication, and authorization. Owns the `User` aggregate. Issues JWTs on login; is the sole source of truth for identity. Publishes `UserCreatedEvent` on successful registration.

Also supports:
- **Google sign-in** (`POST /api/users/login/google`) — verifies a client-obtained Google ID token server-side and issues the same JWT any other login path issues. Auto-links to an existing password account by email. See [`notes.md`](notes.md) 28.
- **Roles.** `User.Role` is `Player` or `Admin`. One admin is seeded at startup from Secret-provided config; every other admin is promoted by an existing one (`PUT /api/users/{id}/role`, admin-only). See [`notes.md`](notes.md) 26.
- **Admin visibility.** An admin can see every user's orders and, per order, the full cross-service trail — see [§4.3](#43-ordersapi), [§4.4](#44-paymentsapi), and [§4.5](#45-notificationsapi).

### 4.2 CatalogAPI
CRUD for the game catalog — **product reference data only**. Owns the `Game` aggregate: title, genre, platform, description, price, release date. Read-heavy, not user-scoped, and deliberately **outside the purchase flow**: it publishes no events and consumes none. Its only role in a purchase is answering "does this game exist and what does it cost" when OrdersAPI asks ([§6](#6-how-orders-learns-the-price)).

### 4.3 OrdersAPI
Owns the purchase lifecycle and, by extension, each user's library. Owns the `Order` aggregate: `OrderId`, `UserId`, `Status` (`Pending` → `Paid` | `Failed`), timestamps, and a collection of `OrderItem { GameId, Price }` (each price a **snapshot** taken at order time) — a cart checkout places one order for several games, so an order is multi-item, not a single game/price pair. `TotalPrice` is the sum of its items' snapshotted prices. Two DB-level unique constraints on `order_items` keep two invariants that used to be application-only checks: a game can't appear twice in the same order, and (excluding `Failed` items) a user can't have two active order items for the same game across any of their orders. See [`notes.md`](notes.md) 51–52.

- Exposes the purchase entry point (`POST /api/orders`, taking `GameIds: Guid[]`), which validates and prices each requested game against CatalogAPI, persists the order as `Pending`, and publishes `OrderPlacedEvent`.
- Consumes `PaymentProcessedEvent` and transitions the order (and each of its items) to `Paid` or `Failed`.
- Exposes the user's library (`GET /api/library`) as a flattened, per-game projection over the items of that user's `Paid` orders that haven't been removed from the library — one row per purchased, still-in-library game, not one row per order.
- Exposes live order status (`GET /api/orders/{id}/stream`, Server-Sent Events) instead of requiring the client to poll — pushes the current status immediately and one more update when the order leaves `Pending`. See [`notes.md`](notes.md) 53.
- Lets a user remove a game from their library (`DELETE /api/library/{gameId}`, confirmed client-side via a modal): sets `OrderItem.RemovedFromLibraryAtUtc`, which excludes the item from the library projection **and** frees the game up for repurchase — without ever touching `Order.Status` or reversing the original charge. See [`notes.md`](notes.md) 54.
- Appends an `OrderEvent` audit row (the actual event payload, not a summary) whenever it publishes `OrderPlacedEvent` or receives `PaymentProcessedEvent`. Admin-only: `GET /api/orders/admin` (every user's orders) and `GET /api/orders/{id}/events` (that order's audit trail). See [`notes.md`](notes.md) 30.

**Why the library lives here**: with refunds, gifting, and key redemption out of scope ([§12](#12-out-of-scope)), "your library" is exactly "the games in your paid, not-removed orders" — a projection, not an independent aggregate. Splitting a separate Library/Entitlements service would add a repo, a schema, and a chart for a distinction this project never exercises. Removing a game from the library is not a refund — no `Payment` changes, the charge stays captured — so it doesn't reopen this. If real refunds ever come into scope, the projection equivalence breaks and Library should split out.

**Why the price is a snapshot**: the order stores the price it was placed at, never a live reference to CatalogAPI's current price. A later sale or price change must not retroactively alter an existing order. This is the aggregate boundary doing real work.

### 4.4 PaymentsAPI
Owns the payment lifecycle for an order. Consumes `OrderPlacedEvent`, obtains an Approved/Rejected outcome through the gateway abstraction described in [§5](#5-payment-gateway-integration), and publishes `PaymentProcessedEvent`.

PaymentsAPI **does not decide payments itself** — it delegates to an `IPaymentGateway` and translates whatever comes back into the system's own domain event. That anti-corruption boundary is deliberate: no other service ever learns whether the outcome came from a simulator or a real provider.

Persists one `Payment` row per order — the actual request/response payload exchanged with the gateway (even the simulated one documents its own deterministic decision this way), not just the resulting status. `OrderPlacedEvent` is idempotent, keyed on `OrderId`: a redelivery is detected and skipped before charging again. Admin-only: `GET /api/payments/{orderId}`. See [`notes.md`](notes.md) 27.

### 4.5 NotificationsAPI
Consumes `UserCreatedEvent` (welcome email) and `PaymentProcessedEvent` (purchase confirmation on `Approved`, payment-failed notice on `Rejected`). Idempotent on both, keyed on `UserId`/`OrderId` respectively via a persisted `Notification` record.

Delivery channel is a runtime switch (`Email:Provider` = `console` default, `resend` optional) — console remains the reliable demo signal; Resend sends a real email and records the actual provider request/response. See [`notes.md`](notes.md) 29.

Keeps a local `UserProjection` (UserId → Name/Email), populated from `UserCreatedEvent`, since `PaymentProcessedEvent` carries only a `UserId` and this service needs an address to send to — a projection, not a cross-service read. Admin-only: `GET /api/notifications?orderId=`. See [`notes.md`](notes.md) 30.

### 4.6 Deviation from the original brief

The original requirement assigned purchase initiation to CatalogAPI ("responsável pelo CRUD de jogos **e por iniciar o fluxo de compra**"). This spec moves that to a dedicated OrdersAPI instead.

The reasoning: a catalog is product reference data, an order is a user-scoped transactional aggregate with its own lifecycle, and putting both in one service means one deployable owning two unrelated consistency models. The original `OrderPlacedEvent` was already being published by a service that didn't own orders — the name gave the seam away.

All four originally-required services still exist and still do their required work; CatalogAPI simply stops doing something that was never a catalog's job. **This is a conscious deviation and should be presented as one** — documented with its rationale rather than left to be discovered as a missing requirement.

## 5. Payment gateway integration

PaymentsAPI sits behind an `IPaymentGateway` abstraction with three interchangeable implementations, composed into an ordered fallback chain (`PaymentGatewayChain`) selected at runtime by a ConfigMap value (`PaymentGateway:Providers` / `PAYMENT_GATEWAY_PROVIDERS`) — a comma-separated, ordered list, not a single value:

| Implementation | Config name | Purpose |
|---|---|---|
| `SimulatedPaymentGateway` | `simulated` | **Default**, and always the guaranteed last resort in a real chain. Deterministic, fully offline. Used by tests, CI, and as the demo fallback. |
| `AbacatePayGateway` | `abacatepay` | Real sandbox integration (Pix QR Code API), tried first when configured. |
| `MercadoPagoGateway` | `mercadopago` | Real sandbox integration (classic Payments API), tried if AbacatePay is unavailable. |

`PaymentGateway:Providers=simulated` (the default) reproduces a single-gateway setup exactly; `abacatepay,mercadopago,simulated` tries each real gateway in turn per order — falling through on `PaymentGatewayUnavailableException` (missing credentials, timeout, non-2xx) — before falling back to the simulator. This is Strategy + Chain of Responsibility selected by configuration, still swappable with no rebuild. Adding a future provider needs one new class + its `ProviderName` appended to the list — see [`notes.md`](notes.md) 38 for why this stays a string list rather than an enum. It also turns the mandatory-ConfigMap requirement ([§9](#9-kubernetes-requirements)) into config that actually does something, rather than a token env var.

### 5.1 Simulated gateway (default)

The outcome must be **deterministic, never random** — a random approve/reject makes every scenario in [`bdd.md`](bdd.md) flaky. The rule keys off the order price, mirroring how real PSP test cards work (Stripe's `4242…` always approves, `4000…0002` always declines):

| Condition | Outcome |
|---|---|
| `Price > 999.00` | Rejected — insufficient funds |
| `Price` ends in `.13` | Rejected — generic decline |
| anything else | Approved |

An artificial delay (`PAYMENT_PROCESSING_DELAY_SECONDS`, ConfigMap, default `5`) keeps the intermediate `Pending` state observable. Without it the whole cascade completes in ~50ms and the async design is invisible in a demo.

### 5.2 Real gateways (AbacatePay and Mercado Pago, dev/sandbox mode)

Both verified viable and free, and both implemented (`notes.md` 38):

- Dev/sandbox mode is **the default on signup** for both; payments are simulated, nothing is charged, and test activity never touches production data.
- Each has its own API-triggered approval endpoint (AbacatePay's `/pixQrCode/simulate-payment`, Mercado Pago's `/sandbox/simulate`), called immediately after charge creation so the cascade stays **fully autonomous** — no human opens a checkout page and clicks anything.
- **Provider API keys live in a k8s Secret** — never in a manifest, an image layer, or a commit. Dev-mode keys are still credentials.

**Confirmation is by polling, not by webhook, in this local topology.** The design below (webhook receiver, signature verification) is still built and fully wired — it's the architecturally correct model, and the one exception `notes.md` 3 carves out of "no webhooks between our own services" — but this project's local cluster has no stable public URL for either provider to call back to, so nothing reaches that endpoint today. The **active** confirmation path is `PaymentStatusPollingService`, an exponential-backoff `BackgroundService` that calls each provider's own GET status-check endpoint until a charge resolves or times out. See `notes.md` 38 for the full rationale.

The webhook path, as built (dormant until a real public Ingress exists):

- **Verify the webhook signature before parsing the body.** An unverified webhook endpoint lets anyone who finds the URL grant themselves a free game library. This is the one piece of genuine production security work in the project — do not skip it. (Mercado Pago's real `x-signature` manifest format still needs verifying against live docs before this is activated — see `../features/payment-gateway-simulate.md`.)
- **A public URL would be required** to actually receive traffic — the local cluster isn't reachable from the internet. Still open — see [§13](#13-open-decisions).
- The webhook handler translates the vendor payload into `PaymentProcessedEvent` and publishes it to the broker, converging on the same idempotent finalize path the poller uses. The vendor's vocabulary stops at PaymentsAPI's edge.

### 5.3 Why dev mode is sufficient

Sandbox/test mode is the documented, intended first step for both providers, costs nothing, and moves no real money. Nothing in this project needs more than that:

- **AbacatePay** — production activation requires an **active CNPJ**; they don't issue CPF-only production accounts, citing banking-partner compliance. Dev mode is therefore not merely sufficient, it's the only route available without opening a MEI — and the project needs nothing that production offers.
- **Mercado Pago** — test credentials come free and renewable with any account. Production requires separate activation plus a monthly integration-quality review that only applies once real payments exist. Staying in test avoids all of it.

The system must never present sandbox results as real transactions. This is a simulated store, and that framing stays in the UI and the docs.

## 6. How Orders learns the price

OrdersAPI needs two facts from CatalogAPI at order time, per requested game: that it exists, and what it costs. **The client must never supply a price** — a client-supplied price is the classic e-commerce exploit (buy a R$ 299 game for R$ 0.01). `POST /api/orders` accepts `GameIds: Guid[]` — a cart can hold several games — and nothing else pricing-related. See `notes.md` 51.

**Decided: a synchronous HTTP read, once per requested game.** OrdersAPI calls CatalogAPI at order time to confirm each game exists and to read its current price, then snapshots that price onto the corresponding `OrderItem`. The Order's total is the sum of its items' snapshotted prices — never re-derived from a live catalog read.

This introduces a runtime dependency — CatalogAPI down means no new orders can be placed. That's accepted, and arguably correct: an order *should* fail rather than be created against a game nobody could verify. The rejected alternative was a local read model fed by `GamePriceChangedEvent`, which removes the coupling but adds a projection to maintain and can price an order from a stale cache.

CatalogAPI's base URL is non-secret configuration → **ConfigMap**.

Note this is a **synchronous read for validation**, which is a different thing from the rule in [§8](#8-event-driven-communication) that the purchase *flow* is event-driven. Fetching a price is a query that happens before the flow starts; the flow itself never blocks on another service.

## 7. Frontend

**React + Vite**, built to static files and served by nginx in a multi-stage container (Vite build stage → nginx runtime stage). Screens: registration/login, catalog browsing, a `localStorage`-backed cart and checkout confirmation, order status, and the user's library.

Checkout has two entry points — `Add to Cart` then a `/cart` review, or a `Buy Now` shortcut on the catalog — but both land on the same `/checkout` confirmation page before an order is actually placed; neither skips the review step. See [`notes.md`](notes.md) 51.

The **`Pending` → `Paid` transition must be visible in the UI** — that's what makes the asynchronous architecture legible to someone watching, instead of an implementation detail they have to take on faith. This is delivered by subscribing to the order's Server-Sent Events stream ([§4.3](#43-ordersapi)) rather than polling, so the transition appears the moment it happens instead of on the next poll tick. See [`notes.md`](notes.md) 53.

The frontend talks to a **single base URL** and relies on Ingress path routing ([§9.1](#91-ingress-routing)), so it never needs to know that five backend services exist. It obtains a JWT from UsersAPI and sends it as `Authorization: Bearer <token>` on every subsequent call.

## 8. Event-driven communication

Asynchronous messaging via **RabbitMQ**. Chosen over Kafka because this is pub/sub plus work queues, not stream processing: nothing here needs replay, partition ordering, or log retention, and RabbitMQ runs as one lightweight pod instead of a broker-plus-controller cluster. Its management UI is also a genuine demo asset — you can watch messages move between services in real time.

Full flows are specified as Gherkin scenarios in [`bdd.md`](bdd.md); this section fixes the contract-level rules.

**Registration flow**

```
UsersAPI ──UserCreatedEvent──▶ NotificationsAPI          ("welcome email")
```

**Purchase flow**

```
                                 ┌──▶ OrdersAPI          (Pending → Paid | Failed)
OrdersAPI ──OrderPlacedEvent──▶ PaymentsAPI ──PaymentProcessedEvent──┤
                                 └──▶ NotificationsAPI   (confirmation | failure notice)
```

CatalogAPI appears in neither. It publishes and consumes nothing.

Contract rules:

- Events are the only cross-service communication *for these flows* — no service calls another's HTTP API synchronously as part of them. (The price lookup in [§6](#6-how-orders-learns-the-price) precedes the flow; it isn't part of it.)
- Event names and payloads are fixed by this spec:
  - `UserCreatedEvent { UserId, Name, Email }`
  - `OrderPlacedEvent { OrderId, UserId, GameIds, TotalPrice }`
  - `PaymentProcessedEvent { OrderId, UserId, Status }`
- **`OrderId` is the correlation key** across the whole purchase flow. It's the idempotency key for both consumers and the join key for tracing a flow through the logs — which is why it belongs on both events.
- Queue/exchange names and connection details are non-secret configuration → **ConfigMap**, never hardcoded.
- Broker credentials are secret → **Secret**.

## 9. Kubernetes requirements

Local-only for now (see [§11](#11-local-only-scope)) — the cluster is expected to be Docker Desktop's Kubernetes, kind, or minikube.

Mandatory rules, non-negotiable per the assignment:

- **No bare Pods.** Every workload is managed by a **Deployment**. A Pod created in isolation is not acceptable.
- **ConfigMaps are mandatory** for all non-sensitive configuration — queue/exchange names, service base URLs (including Orders → Catalog), `PAYMENT_GATEWAY_PROVIDERS`, `PAYMENT_PROCESSING_DELAY_SECONDS`, `PAYMENT_POLLING_*`, log levels.
- **Secrets are mandatory** for all sensitive configuration — Postgres connection strings, the JWT signing key, broker credentials, payment provider API keys and webhook signing secrets.
- Each backend repo and the frontend repo carries its **own** `/k8s` folder at its root, holding that service's **Helm subchart** — Deployment, Service, ConfigMap, and Secret template. The chart's `templates/` *are* the manifests the requirement asks for; packaging them as a chart doesn't replace them.
- The `orchestration` repo is the **Helm umbrella chart** that depends on all six subcharts, plus what no single service owns: namespace, RabbitMQ, PostgreSQL, the Ingress, and bring-up tooling.

### 9.1 Ingress routing

One entry point (nginx-ingress), path-routed, so the frontend sees a single base URL:

| Path | Service |
|---|---|
| `/api/users/*` | users-api |
| `/api/games/*` | catalog-api |
| `/api/orders/*`, `/api/library` | orders-api |
| `/api/payments/*` | payments-api — admin-only (`GET /api/payments/{orderId}`); `/api/payments/webhooks/{provider}` is built and would be the one *externally*-reachable path (per [§5.2](#52-real-gateways-abacatepay-and-mercado-pago-devsandbox-mode)) once a real public Ingress exists — dormant in this local topology, where confirmation is by polling instead |
| `/api/notifications/*` | notifications-api — admin-only (`GET /api/notifications?orderId=`) |
| `/*` | frontend |

### 9.2 Shared infrastructure

RabbitMQ and PostgreSQL are deployed by the `orchestration` chart as Deployments with persistent storage. Neither is exposed through the Ingress — services reach them by in-cluster DNS. Their credentials live in Secrets, and each service receives only its own Postgres role ([§10](#10-non-functional-requirements)).

## 10. Non-functional requirements

Carried over from `base-project`'s spec, now applied per service instead of per module:

- **Validation**: request DTOs validated before reaching domain/service logic; `400` with field-level errors on invalid input.
- **Global error handling**: unhandled exceptions never leak a stack trace; logged in full server-side, returned as a generic `500` with a trace id.
- **Structured logging**: one JSON log line per request; each service also logs its publish/consume events with `{OrderId}` as a named property, so a purchase can be traced end to end across five services.
- **Result pattern**: expected failures modeled explicitly and mapped to HTTP status codes, not thrown as exceptions.
- **Persistence isolation by schema**: all five services share **one PostgreSQL instance**, but each owns a dedicated schema (`users`, `catalog`, `orders`, `payments`, `notifications`) and connects with **its own database role, granted access only to that schema**. No service can read another's tables — the boundary is enforced by Postgres itself, not by convention. Cross-service state still only ever arrives via an event or an explicit API call. Each service keeps its own EF Core migration history table inside its own schema.
- **Transactional outbox**: OrdersAPI writes the `Pending` order and enqueues `OrderPlacedEvent` in a **single Postgres transaction** via an outbox table; a relay publishes from that table to RabbitMQ and marks rows sent. Either both happen or neither does. This closes the dual-write gap that would otherwise leave an order `Pending` forever with no event to drive it — and it is the specific reason a relational database was chosen.
- **Idempotent consumers**: brokers redeliver, and so would real payment webhooks if activated (AbacatePay and Mercado Pago both retry on non-2xx) — and the active polling path can observe the same terminal status more than once across ticks. Every consumer, and `PaymentFinalizationService`, must tolerate processing the same outcome twice without duplicating its effect — no second welcome email, no double library entry, no double-publish. `OrderId` is the dedupe key for the purchase flow.
- **Order state transitions are one-way**: `Pending → Paid` and `Pending → Failed` only. A late or duplicated `PaymentProcessedEvent` must never move an order backwards or flip a settled one.
- **Approval never originates from the client.** An order becomes `Paid` only via `PaymentProcessedEvent`, never by a frontend call claiming success. This is the single most-exploited flaw in real e-commerce, and the design must preserve the property deliberately, not by accident.

## 11. Local-only scope

This phase runs **exclusively on a local Kubernetes cluster**. Unlike `base-project`, there is no Terraform/Azure target — that remains a possible future phase, not part of this spec. Do not carry `infra/terraform` forward.

The one exception to "local" would be the optional payment webhook ([§5.2](#52-real-gateways-abacatepay-and-mercado-pago-devsandbox-mode)), which requires an inbound tunnel — it's built but not activated, because this phase confirms real gateway charges by polling instead, which needs no inbound connectivity at all. `simulated` keeps the system fully functional offline regardless.

## 12. Out of scope

- **Production payment processing** — real money never moves; the real-gateway path runs exclusively against sandbox/dev credentials ([§5.3](#53-why-dev-mode-is-sufficient)).
- ~~Real email delivery~~ — now optional via Resend, console remains the default; see [§4.5](#45-notificationsapi) and [`notes.md`](notes.md) 29.
- Cloud deployment / managed Kubernetes (see [§11](#11-local-only-scope)).
- **Refunds, chargebacks, gifting, and key redemption.** Their absence is what makes "library = paid orders" true ([§4.3](#43-ordersapi)); reintroducing any of them means splitting a Library service out.
- The full auth→capture→settle lifecycle. Outcomes collapse to `Approved`/`Rejected` — a deliberate simplification, appropriate for digital goods where authorization and capture would be immediate anyway.
- ~~A shopping cart / multi-item order. One order is one game.~~ — built; see [§4.3](#43-ordersapi) and [`notes.md`](notes.md) 51.
- Multi-tenancy.
- Real-time features (websockets/SignalR) beyond what the event flows require internally — the one exception is order/payment status, which the client now consumes via Server-Sent Events rather than polling ([§7](#7-frontend), [`notes.md`](notes.md) 53); that's a status *push* to the client, not a bidirectional channel or an internal service-to-service mechanism, so it doesn't reopen this line.

## 13. Open decisions

**Closed.** Rationale for each is in [`notes.md`](notes.md).

| Decision | Resolution |
|---|---|
| Ownership of the purchase flow | A dedicated OrdersAPI ([§4.3](#43-ordersapi)) |
| Where the library lives | Inside OrdersAPI, as a projection over `Paid` orders |
| Payment approval mechanism | Swappable gateway, simulated by default ([§5](#5-payment-gateway-integration)) |
| Rejected-payment notification | NotificationsAPI sends a failure notice |
| Correlation id | `OrderId`, carried on both purchase events |
| Message broker | **RabbitMQ** ([§8](#8-event-driven-communication)) |
| Persistence | **PostgreSQL** — one shared instance, one schema and one role per service ([§10](#10-non-functional-requirements)) |
| Orders → Catalog price lookup | **Synchronous HTTP read** ([§6](#6-how-orders-learns-the-price)) |
| Outbox pattern | **Implemented** — transactional outbox in OrdersAPI ([§10](#10-non-functional-requirements)) |
| Frontend | **React + Vite**, nginx container ([§7](#7-frontend)) |
| Client entry point | **Single nginx-ingress with path routing** ([§9.1](#91-ingress-routing)) |
| Service-to-service JWT trust | **Shared symmetric secret** in a k8s Secret, as in `base-project` |
| Manifest tooling | **Helm** — a subchart per repo, umbrella chart in `orchestration` ([§9](#9-kubernetes-requirements)) |
| Sharing the kernel/auth code | **Duplicated per service**, not packaged |
| Messaging library | **MassTransit** — supplies the EF Core outbox as configuration |
| Repo location | **`repos/`** in this workspace |
| Running a single service | **Per-repo `docker-compose.yml`**, alongside the Helm path ([§3.1](#31-two-ways-to-run)) |
| Local cluster | **kind** — runs inside Docker, lightest of the three |

Still open:

1. **Tunnel tooling** (only if the real payment gateway is used): named cloudflared tunnel vs. ngrok — stability vs. setup cost.

## 14. Acceptance criteria

- [x] Each of the five backend services builds, runs, and passes its own test suite independently. — 122 tests total (users 18, catalog 17, orders 30, payments 52, notifications 5).
- [x] Each backend repo and the frontend repo has its own `Dockerfile` and its own `/k8s` folder at its root.
- [x] `helm install` of the `orchestration` umbrella chart brings up the full environment (Postgres + RabbitMQ + five services + frontend + Ingress) from a clean cluster in one command. — verified: a fresh `helm uninstall` + PVC delete + `helm install` brought up all **8** pods (5 backends + Postgres + RabbitMQ + frontend) with **zero restarts** (an initContainer per DB-backed service waits for Postgres before migrating, closing a cold-start race — see `notes.md` 25).
- [x] No Pod exists outside a Deployment anywhere in the system.
- [x] All non-sensitive config comes from ConfigMaps; all sensitive config from Secrets — no hardcoded connection strings or keys in any manifest or image. — re-audited via `helm template | grep` across all five services (including the new admin/resend/google config); literal values appear only inside Secret resources themselves.
- [x] End-to-end registration flow: `UsersAPI` publishes `UserCreatedEvent`, `NotificationsAPI` logs the simulated welcome email.
- [x] End-to-end purchase flow works for both outcomes: `OrdersAPI` → `OrderPlacedEvent` → `PaymentsAPI` → `PaymentProcessedEvent` → `OrdersAPI` (order `Paid` only on `Approved`) → `NotificationsAPI` (correct message for each outcome).
- [x] CatalogAPI publishes no events and consumes none; the purchase flow completes without it being in the message path.
- [x] Each service connects to Postgres with its own role and **cannot** read any other service's schema — verifiable by attempting a cross-schema query with that service's credentials and being refused. — verified for all five roles, including payments-api and notifications-api now that both persist data (`notes.md` 27, 30).
- [x] OrdersAPI's order write and its `OrderPlacedEvent` are atomic: killing the process between them leaves neither an order without its event nor an event without its order. — verified by scaling RabbitMQ to zero, placing an order (the write succeeded, the event sat durably in the outbox table), then scaling RabbitMQ back up and confirming the relay delivered it and the cascade completed — with no `orders-api` restart. A crash before the transaction commits is covered by ordinary Postgres ACID semantics (nothing to persist to fail).
- [x] An order's `Price` is fixed at creation — changing a game's price in CatalogAPI afterwards does not alter any existing order.
- [x] `POST /api/orders` ignores any client-supplied price; the order is priced from CatalogAPI.
- [x] `GET /api/library` returns exactly the caller's `Paid`, not-removed order items — never `Pending`, `Failed`, or library-removed ones.
- [x] `DELETE /api/library/{gameId}` removes a game from the caller's library and immediately allows repurchasing it — verified live: the game disappears from `/library`, `POST /api/orders` for the same `gameId` then succeeds instead of `409`, and the original `Paid` order is untouched (`notes.md` 54).
- [x] The frontend reaches every service through one base URL; no service-specific port or host appears in its build. — the app calls only relative `/api/*` paths; the Ingress routes each to the right service by path on the same origin the frontend is served from.
- [x] The `Pending` → `Paid` transition is visible in the UI without a manual refresh. — verified live in Chrome: placed an order, watched the badge flip from `Pending` to `Paid` on its own, pushed over the order status page's Server-Sent Events stream (`notes.md` 53), no poll in the Network tab.
- [ ] Switching `PaymentGateway:Providers` in the ConfigMap changes the active chain with **no code change and no image rebuild**. — `AbacatePayGateway` and `MercadoPagoGateway` are now both built and unit-tested (HTTP mocked; see `notes.md` 38), and the config-driven chain composition is exercised by `PaymentGatewayChainTests`. Not yet verified live against a running cluster with real credentials — that's the explicit follow-up `notes.md` 38 and `../features/payment-gateway-simulate.md` flag.
- [x] The simulated gateway returns the same outcome for the same price on every run. — unit tested at the exact `999.00` boundary plus 8 other cases, and confirmed live.
- [ ] With a real gateway active, a webhook with an invalid or missing signature is rejected and **no** `PaymentProcessedEvent` is published. — `IPaymentWebhookHandler` and the signature checks are built and unit-tested for both providers, but the endpoint is dormant in this local topology (no public Ingress reaches it) — confirmation currently happens via polling instead. See `notes.md` 38.
- [x] No payment provider API key appears in any manifest, image layer, or commit. — `AbacatePay`/`MercadoPago` credentials are wired via k8s Secrets (`abacatepay-credentials`, `mercadopago-credentials`), empty by default; verified by rendering the Helm templates and confirming no literal key ever appears in a ConfigMap or Deployment manifest.
- [x] A JWT issued by `UsersAPI` is accepted by the other four services; a missing/invalid token is rejected with `401`/`403` by each independently. — verified for all five: users/catalog/orders reject with `401` unauthenticated; payments-api and notifications-api's new admin endpoints reject with `401` unauthenticated and `403` for a non-admin token.
- [x] Re-delivering the same event to a consumer does not duplicate its effect, keyed on `OrderId`. — verified live with a throwaway MassTransit replay tool: redelivered `PaymentProcessedEvent` was ignored by both `orders-api` (state guard, already-Paid) and `notifications-api` (persistent dedupe store, keyed on `OrderId`); redelivered `UserCreatedEvent` was skipped by the same dedupe mechanism keyed on `UserId`.
- [x] A single `OrderId` traces the whole purchase across all five services' logs. — within the services the purchase flow actually touches: `orders-api`, `payments-api`, `notifications-api` all log `OrderId` as a named property end to end. `UsersAPI`/`CatalogAPI` are outside the purchase event path by design (see §4.6, criterion above).
- [x] An admin can view every user's orders and, per order, the full cross-service trail — the actual event payloads, the payment gateway's request/response, and the notification(s) sent — while a non-admin gets `403` from all four admin endpoints. — verified live end to end with a seeded admin account.
- [x] Redelivering `OrderPlacedEvent` to `payments-api` does not create a second `Payment` row for the same order. — verified live with the replay tool; same `Payment.Id` before and after.
- [ ] Google sign-in issues the same JWT shape as password login, and auto-links to an existing password account by matching email. — the frontend's login page conditionally renders the button (confirmed hidden when unconfigured, via `GET /api/users/config`) and the backend endpoint fails cleanly (`401`, not a crash) with no client ID configured; the success path still needs a real Google OAuth client ID, which this local/academic deployment doesn't have.
