# Fluxo de compra

Do `POST /api/orders` até `Paid`/`Failed`: a leitura síncrona de preço no `catalog-api`, o outbox transacional publicando `OrderPlacedEvent`, a cobrança determinística no `payments-api`, e o `PaymentProcessedEvent` fechando o ciclo em `orders-api` e `notifications-api`. Referenciado em `../architecture/ARCHITECTURE.en-US.md`/`.pt-BR.md` §5.2.

```mermaid
sequenceDiagram
    participant C as Navegador
    participant O as orders-api
    participant Cat as catalog-api
    participant R as RabbitMQ
    participant P as payments-api
    participant N as notifications-api

    C->>O: POST /api/orders {GameIds}
    O->>Cat: GET /api/games/{id}  (síncrono, uma vez por jogo pedido)
    Cat-->>O: preço
    O->>O: cria Order (Pending) com um OrderItem por jogo, adiciona OrderEvent
    O->>R: publica OrderPlacedEvent {GameIds, TotalPrice} (mesma transação — outbox)
    R->>P: OrderPlacedEvent
    P->>P: TryClaim por OrderId (idempotente)
    P->>P: IPaymentGateway.ChargeAsync (regra de preço determinística)
    P->>P: persiste Payment (payload de request/response)
    P->>R: publica PaymentProcessedEvent
    R->>O: PaymentProcessedEvent
    O->>O: MarkPaid/MarkFailed (unidirecional, no-op se já resolvido)
    O->>O: adiciona OrderEvent
    R->>N: PaymentProcessedEvent
    N->>N: TryClaim por OrderId, consulta UserProjection
    N->>N: envia confirmação ou aviso de falha
```
