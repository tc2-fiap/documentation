# FIAP Games — Microservices Behavior (BDD)

Gherkin scenarios describing the observable, cross-service behavior of the distributed system specified in [`instructions.md`](instructions.md). Same role [`base-project/docs/behavior/behavior.md`](https://github.com/KainanGuerra/fiap-games/blob/main/docs/behavior/behavior.md) plays for the monolith: this is the acceptance layer, not implementation detail.

Per-service CRUD/auth behavior (register, login, get/list/update/delete, pagination) already has a proven template in that document — restate it per service when each repo is scaffolded. This file covers what's *new* in the distributed system: event flows crossing service boundaries, order lifecycle, gateway selection, cross-service authorization, and Kubernetes deployment shape.

---

## Feature: User registration triggers a welcome notification

```gherkin
Feature: User registration event flow
  As a new user
  I want a welcome notification when I register
  So that I know my account was created

  Scenario: Successful registration publishes an event
    Given no user is registered with the email "player@example.com"
    When I POST /api/users/register on UsersAPI with a valid name, email, and password
    Then the response status is 201 Created
    And UsersAPI publishes a UserCreatedEvent containing the new user's id, name, and email

  Scenario: NotificationsAPI consumes the event
    Given a UserCreatedEvent has been published for user "player@example.com"
    When NotificationsAPI consumes that event
    Then NotificationsAPI logs a simulated welcome email to the console, addressed to "player@example.com"
    And no real email is sent

  Scenario: Duplicate email never reaches the event bus
    Given a user already exists with the email "player@example.com"
    When I POST /api/users/register with the email "player@example.com"
    Then the response status is 409 Conflict
    And no UserCreatedEvent is published
```

## Feature: Placing an order

```gherkin
Feature: Order creation
  As an authenticated user
  I want to place an order for one or more games in a single checkout
  So that the purchase flow starts

  Scenario: Placing an order prices it from the catalog
    Given I am authenticated
    And game "The Witcher 3" exists in CatalogAPI with price 29.99
    When I POST /api/orders on OrdersAPI with GameIds containing only that game
    Then the response status is 202 Accepted
    And an Order is persisted with Status "Pending" and one OrderItem priced 29.99
    And OrdersAPI publishes an OrderPlacedEvent containing OrderId, UserId, GameIds, and TotalPrice

  Scenario: Placing a cart checkout with multiple games in one order
    Given I am authenticated
    And game "Hollow Knight" exists in CatalogAPI with price 34.99
    And game "Elden Ring" exists in CatalogAPI with price 249.90
    When I POST /api/orders with GameIds containing both games
    Then the response status is 202 Accepted
    And a single Order is persisted with Status "Pending", two OrderItems, and TotalPrice 284.89
    And OrdersAPI publishes one OrderPlacedEvent for that OrderId containing both GameIds and TotalPrice 284.89

  Scenario: A client-supplied price is ignored
    Given I am authenticated
    And a game exists in CatalogAPI with price 299.00
    When I POST /api/orders with that game's id and a body also claiming a price of 0.01
    Then the created Order's OrderItem has Price 299.00, taken from CatalogAPI
    And the published OrderPlacedEvent carries TotalPrice 299.00

  Scenario: Ordering a game that does not exist
    Given I am authenticated
    When I POST /api/orders with a GameId that CatalogAPI does not recognize
    Then the response status is 404 Not Found
    And no Order is persisted
    And no OrderPlacedEvent is published

  Scenario: The order price is a snapshot
    Given I have a Pending order for a game priced at 29.99
    When the game's price is changed to 9.99 in CatalogAPI
    Then my existing order's OrderItem still has Price 29.99
    And the change does not propagate to any existing order

  Scenario: A duplicate game id in the same checkout is rejected
    Given I am authenticated
    And game "Hollow Knight" exists in CatalogAPI
    When I POST /api/orders with GameIds listing that same game twice
    Then no Order is persisted
    And this is enforced by a unique index on order_items(order_id, game_id), not only request validation

  Scenario: Ordering a game I already own or have pending is rejected
    Given I am authenticated
    And I have a Paid order containing "Elden Ring"
    When I POST /api/orders with GameIds containing "Elden Ring" again
    Then the response status is 409 Conflict
    And no new Order is persisted
    And a concurrent duplicate request for the same user and game is rejected by a partial unique index on order_items(user_id, game_id) even if both requests pass the initial application-level check

  Scenario: Placing an order requires authentication
    When I POST /api/orders without a bearer token
    Then the response status is 401 Unauthorized

  Scenario: CatalogAPI unavailable blocks new orders
    Given I am authenticated
    And CatalogAPI is unreachable
    When I POST /api/orders with a valid GameId
    Then the order is not created
    And no OrderPlacedEvent is published
    And the response reports the failure rather than creating an unpriced order
```

## Feature: Persistence boundaries and the outbox

```gherkin
Feature: Schema isolation
  As an operator
  I want the database to enforce service boundaries
  So that isolation does not depend on developers remembering a rule

  Scenario: A service can read its own schema
    Given all five services share one PostgreSQL instance
    When OrdersAPI queries its own tables with its own role
    Then the query succeeds

  Scenario Outline: A service cannot read another service's schema
    Given all five services share one PostgreSQL instance
    When I connect with <service>'s credentials and query the <other> schema
    Then Postgres refuses the query with a permission error
    And the refusal comes from the database, not from application code

    Examples:
      | service       | other    |
      | orders-api    | users    |
      | catalog-api   | orders   |
      | payments-api  | orders   |

  Scenario: Each service owns its migration history
    Given the system has been deployed to a clean database
    When I inspect the migration history tables
    Then each service's history table lives inside that service's own schema
    And no service's migrations appear in another's history
```

```gherkin
Feature: Transactional outbox
  As an operator
  I want the order write and its event to be atomic
  So that a crash can never leave an order nobody will process

  Scenario: The order and its event are written together
    Given I am authenticated
    When I POST /api/orders successfully
    Then the Order row and the outbox row are written in the same transaction
    And the relay subsequently publishes OrderPlacedEvent to RabbitMQ and marks the outbox row sent

  Scenario: A crash before publishing loses neither
    Given an order has been committed with its outbox row
    When OrdersAPI is killed before the relay publishes
    And OrdersAPI restarts
    Then the relay publishes the pending OrderPlacedEvent
    And the order is processed normally, with no manual intervention

  Scenario: A crash before commit creates nothing
    Given a POST /api/orders is in flight
    When OrdersAPI is killed before the transaction commits
    Then no Order row exists
    And no OrderPlacedEvent is ever published for it

  Scenario: The relay does not publish twice
    Given the relay has published an OrderPlacedEvent and marked its outbox row sent
    When the relay runs again
    Then that row is not republished
```

## Feature: Purchase flow — approved payment

```gherkin
Feature: Purchase flow (approved)
  As an authenticated user
  I want an approved payment to add the game to my library
  So that I can play what I bought

  Scenario: PaymentsAPI processes the order and approves it
    Given an OrderPlacedEvent has been published for OrderId "<id>"
    When PaymentsAPI consumes that event and the payment is approved
    Then PaymentsAPI publishes a PaymentProcessedEvent with Status "Approved" for OrderId "<id>"

  Scenario: OrdersAPI marks the order paid
    Given a PaymentProcessedEvent with Status "Approved" has been published for OrderId "<id>"
    When OrdersAPI consumes that event
    Then the order's Status becomes "Paid"
    And a subsequent GET /api/library includes that game

  Scenario: NotificationsAPI confirms the purchase
    Given a PaymentProcessedEvent with Status "Approved" has been published for OrderId "<id>"
    When NotificationsAPI consumes that event
    Then NotificationsAPI logs a simulated purchase-confirmation email to the console

  Scenario: CatalogAPI is not in the message path
    Given a full purchase flow completes for OrderId "<id>"
    When I inspect what CatalogAPI published and consumed during that flow
    Then CatalogAPI published no events and consumed no events
```

## Feature: Purchase flow — rejected payment

```gherkin
Feature: Purchase flow (rejected)
  As an authenticated user
  I want a rejected payment to leave my library unchanged
  So that I never receive a game I didn't pay for

  Scenario: PaymentsAPI rejects the order
    Given an OrderPlacedEvent has been published for OrderId "<id>"
    When PaymentsAPI consumes that event and the payment is rejected
    Then PaymentsAPI publishes a PaymentProcessedEvent with Status "Rejected" for OrderId "<id>"

  Scenario: OrdersAPI marks the order failed
    Given a PaymentProcessedEvent with Status "Rejected" has been published for OrderId "<id>"
    When OrdersAPI consumes that event
    Then the order's Status becomes "Failed"
    And a subsequent GET /api/library does not include that game

  Scenario: NotificationsAPI reports the failure instead of confirming
    Given a PaymentProcessedEvent with Status "Rejected" has been published for OrderId "<id>"
    When NotificationsAPI consumes that event
    Then NotificationsAPI logs a simulated payment-failed email to the console
    And no purchase-confirmation email is logged
```

## Feature: Order state machine

```gherkin
Feature: Order state transitions
  As an operator
  I want order state to move only forward
  So that a late or duplicated event can never corrupt a settled order

  Scenario Outline: The library reflects only paid orders
    Given I have an order with Status <status>
    When I GET /api/library
    Then the game is <visibility> in my library

    Examples:
      | status  | visibility |
      | Pending | absent     |
      | Paid    | present    |
      | Failed  | absent     |

  Scenario: A settled order does not move backwards
    Given an order has Status "Paid"
    When a PaymentProcessedEvent with Status "Rejected" arrives late for that same OrderId
    Then the order remains "Paid"
    And the anomaly is logged

  Scenario: An unknown OrderId is handled safely
    Given a PaymentProcessedEvent arrives for an OrderId OrdersAPI has never seen
    When OrdersAPI consumes it
    Then no order is created from the event
    And the event is acknowledged without crashing the consumer
```

## Feature: Remove a game from the library

```gherkin
Feature: Remove a game from the library
  As a player
  I want to remove a game from my library
  So that I can repurchase it later without a stale entitlement blocking me

  Scenario: Removing an owned game frees it up for repurchase
    Given I own a game via a Paid order
    When I DELETE /api/library/{gameId} and confirm
    Then the game no longer appears in a subsequent GET /api/library
    And a subsequent POST /api/orders for that same gameId succeeds instead of returning 409
    And the original order's Status remains "Paid"

  Scenario: Removal is not a refund
    Given I own a game via a Paid order
    When I DELETE /api/library/{gameId}
    Then the order's Status stays "Paid"
    And no Payment record is modified

  Scenario: A game not owned cannot be removed
    Given I do not own the requested game
    When I DELETE /api/library/{gameId}
    Then the request returns 404
```

## Feature: Payment gateway selection

```gherkin
Feature: Swappable payment gateway chain
  As an operator
  I want to choose an ordered list of payment gateways by configuration
  So that the system runs offline in tests and falls back gracefully against real sandboxes in a demo

  Scenario Outline: Simulated outcomes follow the price rule
    Given PaymentGateway:Providers is set to "simulated"
    When PaymentsAPI processes an OrderPlacedEvent with TotalPrice <price>
    Then the payment outcome is <status>
    And processing the same order again produces <status> every time, never a random result

    Examples:
      | price   | status   |
      | 29.99   | Approved |
      | 0.00    | Approved |
      | 1500.00 | Rejected |
      | 49.13   | Rejected |

  Scenario: The pending state is observable
    Given PaymentGateway:Providers is "simulated" and PAYMENT_PROCESSING_DELAY_SECONDS is 5
    And I am authenticated
    When I POST an order and immediately GET that order
    Then its Status is "Pending"
    And the game is not yet in my library
    And after the delay elapses, the order's Status is "Paid" and the game appears in my library

  Scenario: Switching the gateway chain requires no rebuild
    Given the system is deployed with PaymentGateway:Providers set to "simulated"
    When PaymentGateway:Providers is changed to "abacatepay,mercadopago,simulated" in the ConfigMap and the Deployment is restarted
    Then PaymentsAPI tries AbacatePay first for each new order
    And no container image was rebuilt and no source file was changed

  Scenario: An unavailable gateway falls through to the next one in the chain
    Given PaymentGateway:Providers is "abacatepay,mercadopago,simulated"
    And AbacatePay has no API key configured
    When PaymentsAPI processes an OrderPlacedEvent
    Then AbacatePay is skipped without failing the order
    And Mercado Pago is tried next
    And if Mercado Pago is also unavailable, the order is still charged via the simulated gateway
```

## Feature: Real payment gateway confirmation via polling

This is the **active** confirmation path for real gateways in this local topology — see `notes.md` 38 for why polling stands in for the webhook design below, which is built but currently unreachable (no public Ingress in this environment).

```gherkin
Feature: Poll a real gateway until the charge resolves
  As the system
  I want to confirm a real gateway's charge without an inbound public URL
  So that PaymentsAPI can run entirely on a local cluster with no tunnel

  Scenario: A charge starts Processing and is confirmed by polling
    Given PaymentGateway:Providers includes "abacatepay"
    When PaymentsAPI processes an OrderPlacedEvent
    Then a Payment row is persisted with Status "Processing" and no PaymentProcessedEvent is published yet
    And PaymentsAPI immediately triggers AbacatePay's dev-mode "simulate-payment" endpoint so no human has to click anything
    And PaymentStatusPollingService polls AbacatePay's status endpoint on an exponential backoff
    When the provider reports the charge as paid
    Then PaymentsAPI publishes a PaymentProcessedEvent with Status "Approved" for that OrderId

  Scenario: A charge that never resolves eventually settles as Rejected
    Given a Payment row has been Processing for PaymentGateway:Polling:MaxAttempts polling attempts
    When PaymentStatusPollingService polls again and the provider still reports it unresolved
    Then the Payment is forced to Status "Rejected"
    And a PaymentProcessedEvent with Status "Rejected" is published
    And the order never stays Pending forever
```

## Feature: Real payment gateway webhook (built, scoped for a future public deployment)

This describes `IPaymentWebhookHandler` and the `POST /api/payments/webhooks/{provider}` endpoint as actually built — still architecturally correct, and the one exception `notes.md` 3 carves out of "no webhooks between our own services." Nothing in this project's local topology can reach this route today (no public Ingress); the scenarios above under "confirmation via polling" describe the path that's actually active. See `notes.md` 38.

```gherkin
Feature: AbacatePay sandbox webhook
  As an operator
  I want payment outcomes to arrive from the real provider
  So that the approval genuinely originates outside the system, once a public Ingress exists

  Scenario: A signed webhook produces a domain event
    Given PaymentGateway:Providers includes "abacatepay"
    And a payment has been created in AbacatePay dev mode for OrderId "<id>"
    When AbacatePay POSTs a payment-completed webhook with a valid signature
    Then PaymentsAPI verifies the signature before parsing the body
    And PaymentsAPI publishes a PaymentProcessedEvent with Status "Approved" for OrderId "<id>"
    And the event carries no AbacatePay-specific vocabulary, only OrderId, UserId, and Status
    And if PaymentStatusPollingService had already finalized the same Payment first, this webhook is a harmless no-op

  Scenario: An unsigned or tampered webhook is rejected
    Given PaymentGateway:Providers includes "abacatepay"
    When a webhook arrives with a missing, malformed, or incorrect signature
    Then the request is rejected
    And no PaymentProcessedEvent is published
    And the rejection is logged as a security event

  Scenario: The provider retries until acknowledged
    Given AbacatePay POSTs a valid webhook to PaymentsAPI
    When PaymentsAPI processes it successfully
    Then PaymentsAPI responds with HTTP 200
    And the provider does not retry the notification

  Scenario: Provider credentials never appear outside a Secret
    Given the system is deployed with PaymentGateway:Providers including "abacatepay"
    When I inspect the manifests, the container image, and the repository history
    Then the AbacatePay API key and webhook signing secret appear only as references to a Kubernetes Secret
    And neither value is hardcoded anywhere
```

## Feature: Cross-service authorization

```gherkin
Feature: Shared JWT authorization across services
  As an API consumer
  I want the same token to work across every service
  So that I authenticate once with UsersAPI and use the whole system

  Scenario: A token issued by UsersAPI is accepted elsewhere
    Given I have logged in via UsersAPI and hold a valid access token
    When I call a protected CatalogAPI or OrdersAPI endpoint with "Authorization: Bearer <token>"
    Then the response status is 200 OK, not 401

  Scenario: A missing token is rejected independently by each service
    When I call any protected endpoint on UsersAPI, CatalogAPI, OrdersAPI, PaymentsAPI, or NotificationsAPI without a bearer token
    Then each responds 401 Unauthorized on its own, without calling UsersAPI synchronously to check

  Scenario: An expired or malformed token is rejected
    When I call a protected endpoint on any service with an expired or malformed bearer token
    Then the response status is 401 Unauthorized

  Scenario: A user sees only their own library
    Given user A and user B each have Paid orders
    When user A calls GET /api/library with their own token
    Then only user A's games are returned
    And none of user B's games are visible

  Scenario: A client cannot grant itself a game
    Given I am authenticated
    And I have a Pending order
    When I call OrdersAPI directly claiming the payment succeeded, with no PaymentProcessedEvent having been published
    Then the order remains "Pending"
    And the game never becomes owned on the client's say-so
```

## Feature: Event consumer resilience

```gherkin
Feature: Idempotent event consumption
  As an operator
  I want redelivered events to be safe to reprocess
  So that a broker retry or a provider webhook retry never duplicates a side effect

  Scenario: A redelivered UserCreatedEvent does not send two welcome emails
    Given NotificationsAPI has already processed a UserCreatedEvent for user "player@example.com"
    When the same event is redelivered
    Then no second welcome email is logged for that user

  Scenario: A redelivered PaymentProcessedEvent does not duplicate a library entry
    Given OrdersAPI has already marked OrderId "<id>" as "Paid"
    When the same event is redelivered for that OrderId
    Then the order remains "Paid"
    And the game appears exactly once in the user's library

  Scenario: A redelivered OrderPlacedEvent does not charge twice
    Given PaymentsAPI has already processed OrderId "<id>"
    When the same OrderPlacedEvent is redelivered
    Then no second PaymentProcessedEvent is published for that OrderId

  Scenario: One OrderId traces the whole flow
    Given a purchase completes for OrderId "<id>"
    When I search the structured logs of all five services for that OrderId
    Then each service that participated logged at least one line carrying it
    And the full sequence can be reconstructed in order
```

## Feature: Kubernetes deployment shape

```gherkin
Feature: Kubernetes deployment rules
  As an operator running the system locally
  I want every workload managed correctly
  So that the cluster matches the mandatory deployment rules

  Scenario: No bare Pods exist
    Given the local cluster has the system's namespace applied
    When I list all Pods in that namespace
    Then every Pod is owned by a Deployment
    And no Pod exists without an owning Deployment

  Scenario: Non-sensitive configuration comes from a ConfigMap
    Given all five backend services are deployed
    When I inspect each Deployment's environment configuration
    Then queue/topic names, service base URLs, and PAYMENT_GATEWAY_PROVIDERS are sourced from a ConfigMap
    And none are hardcoded in the Deployment spec or the container image

  Scenario: Sensitive configuration comes from a Secret
    Given all five backend services are deployed
    When I inspect each Deployment's environment configuration
    Then database connection strings, the JWT signing key, broker credentials, and payment provider keys are sourced from a Secret
    And none are hardcoded in the Deployment spec or the container image

  Scenario: The full local environment comes up from a clean cluster
    Given a clean local Kubernetes cluster with no prior state
    When the orchestration repo's bring-up procedure is applied
    Then the message broker, all five backend services, and the frontend become ready
    And each backend service exposes a working health check
```
