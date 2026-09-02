**English** · [Português](DISCOVERIES.pt-BR.md)

# Discoveries

Notes on concepts that came up while building this distributed system and are worth writing down — things that, in the process of explaining the "why" behind an architectural decision (see [`notes.md`](../spec/notes.md)), turned out to be older and better-established ideas than they first appeared. Written in the same spirit as `base-project/docs/discovers/discover.md`, but grounded in this system's own decisions (Postgres, Kubernetes, RabbitMQ/MassTransit, five independent services) rather than the reference monolith's (MongoDB, Terraform, Azure).

## 1. The "Transactional Outbox" is a 2000s messaging pattern, not a MassTransit feature

`OrderService.CreateAsync` publishes `OrderPlacedEvent` and inserts the `Order` row inside the same unit of work, relying on MassTransit's EF Core outbox to guarantee both commit or neither does (`notes.md` 15). This isn't a MassTransit invention — it's the **Transactional Outbox** pattern, first written up as a named solution to the classic "dual write" problem (a service can't atomically update its own database *and* publish a message to a separate broker without a distributed transaction) in *Enterprise Integration Patterns* (Hohpe & Woolf, 2003), years before RabbitMQ 1.0 or any current .NET messaging library existed. MassTransit's implementation is just a well-executed version of an old idea: write the outgoing message to a table in the *same* database transaction as the business change, then a separate process reliably relays it — atomicity is guaranteed by the database's own transaction, not by the broker.

## 2. "Idempotent consumer" borrows a 150-year-old math word, not a messaging-specific one

Every event consumer here is idempotent, keyed on `OrderId` or `UserId` (`instructions.md` §10). The pattern itself — **Idempotent Receiver** — is catalogued in the same *Enterprise Integration Patterns* book above, but the word "idempotent" predates computing entirely: mathematician Benjamin Peirce coined it in 1870 (*Linear Associative Algebra*) for an operation where applying it twice gives the same result as applying it once (`f(f(x)) = f(x)`). A consumer that re-processes a redelivered `PaymentProcessedEvent` and ends up in the exact same state as if it had only processed it once is that same 19th-century idea, applied to a RabbitMQ redelivery instead of an algebraic operation.

## 3. `Result`/`Result<T>` is a friendlier name for something functional languages have had for decades

This project returns `Result`/`Result<T>` for expected failures (not found, validation, conflict) and reserves exceptions for genuine faults (hard rules). That split — a return type that can be "ok" or "an expected problem," instead of throwing — is what functional languages call the `Either` monad, present in ML and Haskell long before .NET existed. The catchier name most people know it by, **Railway Oriented Programming**, was coined by Scott Wlaschin for the F# community around 2013 — itself just a metaphor (two parallel tracks, success and failure, that never cross) for the much older `Either` idea, now popular enough to show up in a C# minimal API without anyone calling it "functional programming."

## 4. Clean Architecture is Hexagonal Architecture plus a name change, which is DIP plus a diagram

Every service here follows `Domain → Application → Infrastructure`, dependencies pointing inward, `Endpoints` outermost (hard rules). Robert C. Martin named this **Clean Architecture** in a 2012 blog post (later a 2017 book), but the actual rule — outer, volatile layers (frameworks, databases, HTTP) depend on inner, stable ones (business logic), never the reverse — is Alistair Cockburn's **Hexagonal Architecture**, described in 2005, seven years earlier. And Cockburn's own rule is just a direct application of the **Dependency Inversion Principle**, the "D" in SOLID, formalized by Martin himself back in 1994. Three names, one increasingly specific packaging of the same idea, over eighteen years.

## 5. The `OrderPlaced` → `PaymentProcessed` chain is a Saga — the 1987 database kind, not a microservices buzzword

`OrdersAPI` publishes `OrderPlacedEvent`; `PaymentsAPI` reacts and publishes `PaymentProcessedEvent`; `OrdersAPI` reacts to that and settles the order `Paid` or `Failed` (`instructions.md` §8). This is a **Saga**: a business transaction that can't be one ACID database transaction because it spans separate services, so it's split into a sequence of local transactions instead. The term is from a 1987 ACM paper by Hector Garcia-Molina and Kenneth Salem — decades before "microservices" was a word. The one piece this project's saga doesn't need: a *compensating* transaction. A `Rejected` payment doesn't have to undo anything, because nothing was ever committed on the buyer's behalf until the order actually settles — the saga here has a failure path, just not a rollback.

## 6. Choreography vs. orchestration is a named fork, not just "how RabbitMQ happens to work"

Nothing coordinates the `OrderPlaced`/`PaymentProcessed` chain from a central place — each service reacts to events and decides its own next move. That's **choreography**, one of two standard styles for coordinating a multi-service business process (the other, **orchestration**, has a dedicated coordinator explicitly telling each participant what to do next, closer to a workflow engine). Both are named, compared, and debated at length in the microservices literature (e.g. Chris Richardson's *Microservices Patterns*, 2018) — choosing one over the other here wasn't a RabbitMQ default, it was picking a side of an established architectural tradeoff: choreography stays simple as long as the chain is short (it is, here — two hops), and gets hard to reason about the moment a third or fourth service needs to react to the same event; orchestration adds a coordinator but keeps the whole flow legible in one place once a process gets that big.

## 7. `IPaymentGateway` is an Anti-Corruption Layer, same book as `base-project`'s Shared Kernel

`PaymentsAPI` never lets AbacatePay's or Mercado Pago's own status vocabulary leak past `IPaymentGateway` — every implementation translates its provider's specific response into this system's own `PaymentProcessedEvent` (`instructions.md` §5). That's Eric Evans' **Anti-Corruption Layer**, from the same 2003 *Domain-Driven Design* book that named the Shared Kernel pattern `base-project`'s own discoveries doc already covers — a boundary that exists specifically so an external system's model can never dictate your own domain's shape.

## 8. JWT bearer auth is an IETF standard from the early 2010s, not a REST-API-specific convention

Every service here validates the same `Authorization: Bearer <token>` header, issued once by `users-api`. JWT itself is **RFC 7519** (2015); the "Bearer" scheme is **RFC 6750** (2012), part of the OAuth 2.0 framework (**RFC 6749**, 2012). None of this is tied to microservices, Kubernetes, or even REST specifically — it's general-purpose HTTP auth machinery that predates this entire architecture by a decade.

## 9. The `order_items` partial unique index enforces a business rule the same way Codd said to, in 1970

Two Postgres constraints — a plain unique index and a *partial* one (`WHERE status <> 'Failed'`) — guarantee no duplicate item in an order and no double-active-ownership across orders (`notes.md` 52), backing up the application-level check that's inherently racy under concurrency. Letting the database itself be the authority for a data-integrity rule, rather than trusting every code path to check first, is the core idea of the **relational model** E. F. Codd introduced in 1970 — a constraint is just a formal, enforced statement about what data is allowed to exist, and the database enforcing it atomically is exactly what a 55-year-old theory said a database should do.

## 10. Server-Sent Events predates WebSocket as a browser standard

`GET /api/orders/{id}/stream` pushes order status over SSE instead of polling (`notes.md` 53). It's easy to assume WebSocket is the "old" real-time option and SSE a newer, simpler alternative built for cases that don't need two-way traffic — it's the other way round. `EventSource`/SSE's roots are in the mid-2000s HTML5 drafts, and it was implemented in browsers years before the WebSocket protocol was even standardized as **RFC 6455**, in December 2011. SSE looking "simpler" isn't a design retrofit — it came first, and WebSocket was built afterward specifically for the two-way case SSE was never meant to solve.

## 11. "Exactly-once delivery" doesn't exist — this system gets the same effect by combining two older guarantees

RabbitMQ (like every message broker before it, going back to 1990s tools like IBM MQSeries) can only really promise **at-least-once** delivery across a network — a redelivery after a crash or a dropped ack is always possible. This system never tries to prevent that; instead, every consumer is idempotent (discovery 2), so a redelivered event is processed safely rather than never delivered at all. "At-least-once delivery + an idempotent consumer" is the actual, decades-old recipe usually marketed as "exactly-once" — there's no messaging trick underneath it, just a duplicate that quietly does nothing the second time.

## 12. `orchestration`'s Helm subcharts are the "umbrella chart" convention, not a five-services-specific structure

`orchestration/Chart.yaml` declares each service's `/k8s` folder as a subchart dependency (`file://../catalog-api/k8s`, and so on), composing five independently-versioned charts into one deployable release. This is Helm's own long-documented **umbrella chart** pattern, in use since the Helm v2 era (community docs on chart dependencies date to ~2016) for exactly this reason: let each component version and template itself independently, then let one parent chart declare "these, together, are the release" — a monorepo's root workspace file, for Kubernetes manifests instead of npm packages.

## Glossary

- **ACID** — Atomicity, Consistency, Isolation, Durability: the four guarantees a single database transaction makes; a Saga exists specifically because a cross-service business transaction can't get all four for free.
- **Anti-Corruption Layer (ACL)** — a DDD pattern: a translation boundary that keeps an external system's model from leaking into your own domain's model.
- **Choreography** — coordinating a multi-service process with no central coordinator; each service reacts to events on its own.
- **Compensating transaction** — the "undo" step a Saga runs if a later step fails, to reverse an earlier step's effect; this project's saga doesn't need one (see discovery 5).
- **DIP (Dependency Inversion Principle)** — the "D" in SOLID: high-level code should depend on abstractions, not on low-level concrete implementations.
- **Either monad** — a functional-programming type representing "one of two outcomes" (commonly success/failure), without throwing; `Result<T>` is this idea under an OOP-friendly name.
- **Hexagonal Architecture** — Alistair Cockburn's 2005 architecture style: the business logic sits at the center, isolated from frameworks and infrastructure by ports and adapters.
- **Idempotent** — an operation that produces the same result whether applied once or many times; from 19th-century abstract algebra, not computing.
- **IETF / RFC** — the Internet Engineering Task Force and its numbered standards documents (Request for Comments) — the process that produced JWT, OAuth 2.0, and WebSocket as open, vendor-neutral standards.
- **Orchestration (in this context)** — coordinating a multi-service process via a central coordinator that explicitly tells each participant what to do next.
- **Partial index** — a database index that only covers rows matching a `WHERE` condition, letting a constraint apply to a subset of a table instead of all of it.
- **Saga** — a business transaction spanning multiple services, implemented as a sequence of local transactions with (optionally) a compensating step for each.
- **SSE (Server-Sent Events)** — a one-way, HTTP-based push mechanism from server to browser (`text/event-stream`), older than WebSocket as a browser standard.
- **Transactional Outbox** — writing an outgoing message to a table inside the same database transaction as the business change it describes, so a separate relay can publish it reliably afterward.
