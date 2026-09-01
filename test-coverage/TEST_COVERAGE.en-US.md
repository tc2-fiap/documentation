**English** · [Português](TEST_COVERAGE.pt-BR.md)

# FIAP Games — Test Coverage

Based on [`base-project/docs/DOCUMENTATION.md` §7.1](https://github.com/KainanGuerra/fiap-games/blob/main/docs/DOCUMENTATION.md), adapted for five independent backend services instead of one monolithic solution — there is no single aggregate `dotnet test --collect` run here, since each service is its own solution, schema, and test project.

## Measuring it

```bash
cd repos/<service>-api/tests/FiapGames.<Service>.Tests
dotnet test --collect:"XPlat Code Coverage"
```

Each run drops a Cobertura report under that project's `TestResults/`; the numbers below were produced this way, one run per service, on 2026-09-01.

## Aggregate line coverage: 16.0%

1,184 of 7,392 lines, summed across all five services' Cobertura reports (not an average of the five percentages, since the services are far from equal in size).

| Service | Coverage | Tests |
|---|---|---|
| `users-api` | 16.1% | 18 |
| `catalog-api` | 15.9% | 17 |
| `orders-api` | 8.3% | 30 |
| `payments-api` | 32.4% | 52 |
| `notifications-api` | 14.3% | 5 |
| **Total** | **16.0%** | **122** |

## Where coverage is, and isn't

Same pattern as the monolith this system replaced: coverage is concentrated in `Application` (services, validators) and `Domain` entities — every suite here mocks its repository interface and never touches a real Postgres or RabbitMQ. `Endpoints`, `Infrastructure/Persistence` (repositories, `DbContext`, EF migrations), and consumer/broadcaster plumbing sit at or near 0%, validated manually instead — live against the kind cluster in a browser, or against a throwaway Postgres container for anything constraint-shaped (`notes.md` 52, 54).

**`payments-api` is the outlier, at more than double the next-highest service.** Its gateway abstraction — `SimulatedPaymentGateway`, `AbacatePayGateway`, `MercadoPagoGateway`, `PaymentGatewayChain`, `PaymentStatusPollingWorker` — is unit-tested with HTTP mocked via a fake `HttpMessageHandler` (`notes.md` 38), something no other service's `Infrastructure` layer has an equivalent of.

**`orders-api` is the lowest, despite having the most tests of any service.** Its `Infrastructure/Persistence` layer is also the largest — `OrderRepository`, `OrdersDbContext`, and four EF migrations (`InitialCreate`, `AddOrderEventAuditLog`, `MultiItemOrders`, `RemoveFromLibrary`) — all left at 0%, since the properties that matter there (a real partial unique index rejecting a concurrent duplicate purchase, or freeing a removed game up for repurchase) were verified against a real, disposable Postgres instance rather than a mock, per `notes.md` 52 and 54.

## Frontend

`repos/frontend` has no automated test suite (`npm run build`/`npm run lint` are its only scripts — no `vitest`/`jest`). UI changes are verified manually in a live browser against the running cluster instead, per this project's own standing rule for frontend work.
