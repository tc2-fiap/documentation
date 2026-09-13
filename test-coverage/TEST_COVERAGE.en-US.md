**English** · [Português](TEST_COVERAGE.pt-BR.md)

# FIAP Games — Test Coverage

Based on [`base-project/docs/DOCUMENTATION.md` §7.1](https://github.com/KainanGuerra/fiap-games/blob/main/docs/DOCUMENTATION.md), adapted for six independent backend services instead of one monolithic solution — there is no single aggregate `dotnet test --collect` run here, since each service is its own solution, schema (where it has one), and test project.

## Measuring it

```bash
cd <service>-api/tests/FiapGames.<Service>.Tests
dotnet test --collect:"XPlat Code Coverage"
```

(No `repos/` prefix — that's this workspace's own layout, not the reproducible one. Per `notes.md` 50, `documentation` is the one repo never cloned as a sibling; the eight runtime repos, including `<service>-api`, are cloned flat into one parent directory, per `../getting-started/GETTING_STARTED.en-US.md` §1.)

Each run drops a Cobertura report under that project's `TestResults/`; the numbers below were produced this way, one run per service, on 2026-09-13 (re-measured after `platform-api` was added — see `notes.md` 75).

## Aggregate line coverage: 14.9%

1,223 of 8,218 lines, summed across all six services' Cobertura reports (not an average of the six percentages, since the services are far from equal in size).

| Service | Coverage | Tests |
|---|---|---|
| `users-api` | 16.1% | 19 |
| `catalog-api` | 15.0% | 19 |
| `orders-api` | 8.7% | 35 |
| `payments-api` | 32.2% | 52 |
| `notifications-api` | 14.1% | 5 |
| `platform-api` | 2.1% | 2 |
| **Total** | **14.9%** | **132** |

## Where coverage is, and isn't

Same pattern as the monolith this system replaced: coverage is concentrated in `Application` (services, validators) and `Domain` entities — every suite here mocks its repository interface and never touches a real Postgres or RabbitMQ. `Endpoints`, `Infrastructure/Persistence` (repositories, `DbContext`, EF migrations), and consumer/broadcaster plumbing sit at or near 0%, validated manually instead — live against the kind cluster in a browser, or against a throwaway Postgres container for anything constraint-shaped (`notes.md` 52, 54).

**`payments-api` is the outlier, at more than double the next-highest service.** Its gateway abstraction — `SimulatedPaymentGateway`, `AbacatePayGateway`, `MercadoPagoGateway`, `PaymentGatewayChain`, `PaymentStatusPollingWorker` — is unit-tested with HTTP mocked via a fake `HttpMessageHandler` (`notes.md` 38), something no other service's `Infrastructure` layer has an equivalent of.

**`orders-api` has the most tests of any service, but not the lowest coverage** — its `Infrastructure/Persistence` layer is the largest of the five database-backed services (`OrderRepository`, `OrdersDbContext`, and four EF migrations: `InitialCreate`, `AddOrderEventAuditLog`, `MultiItemOrders`, `RemoveFromLibrary`) — all left at 0%, since the properties that matter there (a real partial unique index rejecting a concurrent duplicate purchase, or freeing a removed game up for repurchase) were verified against a real, disposable Postgres instance rather than a mock, per `notes.md` 52 and 54.

**`platform-api` is the actual lowest, and for a different reason than any database-backed service.** It has no `Domain`/`Infrastructure/Persistence` layer to skew the denominator down (no database at all — `notes.md` 75) — its two tests cover `PodService`, the one piece of business logic it has, mocking `IPodReader`. Everything else — `KubernetesPodReader` (the real Kubernetes API client) and `Endpoints/PlatformEndpoints.cs` — is unverified by an automated test the same way every other service's `Endpoints`/`Infrastructure` sits at ~0%, and was instead confirmed live against the running cluster (real pod data returned through the Ingress with an admin token, `403` with a player token).

## Frontend

`repos/frontend` has no automated test suite (`npm run build`/`npm run lint` are its only scripts — no `vitest`/`jest`). UI changes are verified manually in a live browser against the running cluster instead, per this project's own standing rule for frontend work.
