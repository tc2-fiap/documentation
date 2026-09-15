# Topologia dos serviços

Como uma requisição do navegador chega a cada serviço através do Ingress, e como os serviços se conectam entre si (leitura síncrona de preço, eventos via RabbitMQ, cada um com seu próprio schema no Postgres). Referenciado em `../architecture/ARCHITECTURE.en-US.md`/`.pt-BR.md` §1.

Estas setas não são mais só a forma orgânica do sistema — são também, literalmente, a lista de permissões de quatro recursos `NetworkPolicy` (`orchestration/templates/network-policy.yaml`), de fato aplicados pelo Calico desde que o CNI padrão do `kind` foi trocado (§8, `notes.md` 93). Qualquer conexão não desenhada aqui agora é de fato bloqueada, não só evitada por convenção.

```mermaid
flowchart LR
    Browser["Navegador"] -->|"uma única URL base"| Ingress["nginx-ingress"]
    Ingress -->|"/"| Frontend["frontend"]
    Ingress -->|"/api/users"| Users["users-api"]
    Ingress -->|"/api/catalog, /api/quotations"| Catalog["catalog-api"]
    Ingress -->|"/api/orders, /api/library"| Orders["orders-api"]
    Ingress -->|"/api/payments"| Payments["payments-api"]
    Ingress -->|"/api/notifications"| Notifications["notifications-api"]
    Ingress -->|"/api/platform"| Platform["platform-api"]

    Orders -.->|consulta de preço, HTTP síncrono| Catalog
    Catalog -.->|"cotação USD/BRL, cacheada"| Frankfurter[("Frankfurter /\nExchangeRate-API")]
    Orders -.->|"OrderPlacedEvent"| RabbitMQ(("RabbitMQ"))
    Payments -.->|"PaymentProcessedEvent"| RabbitMQ
    RabbitMQ -.-> Orders
    RabbitMQ -.-> Notifications
    Users -.->|"UserCreatedEvent"| RabbitMQ
    RabbitMQ -.->|"TokenRevokedEvent (logout, transversal)"| Users & Catalog & Orders & Payments & Notifications & Platform
    Platform -.->|"lista pods (RBAC)"| K8sAPI[("Kubernetes API")]

    Users --> Postgres[("PostgreSQL\n(1 instância, 1 schema+role por serviço)")]
    Catalog --> Postgres
    Orders --> Postgres
    Payments --> Postgres
    Notifications --> Postgres
```
