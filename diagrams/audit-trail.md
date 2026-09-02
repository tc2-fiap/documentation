# Trilha de auditoria por pedido

Como a tela de admin compõe o histórico completo de um pedido a partir de quatro endpoints admin-only independentes — cada serviço devolvendo só o próprio dado, nunca uma junção entre schemas. Referenciado em `../architecture/ARCHITECTURE.en-US.md`/`.pt-BR.md` §6.

```mermaid
flowchart TB
    Admin["Admin (frontend)"] --> OA["GET /api/orders/admin\n(pedidos de todos os usuários)"]
    Admin --> OE["GET /api/orders/{id}/events\n(orders-api)"]
    Admin --> PA["GET /api/payments/{orderId}\n(payments-api)"]
    Admin --> NA["GET /api/notifications?orderId=\n(notifications-api)"]
```
