# Eventos de todo o sistema

Como a página `/admin/events` compõe a lista de "tudo o que já aconteceu" a partir de quatro endpoints admin "listar tudo" diferentes dos usados na trilha de auditoria por pedido. `catalog-api` está ausente aqui — não publica nem consome nada. Referenciado em `../architecture/ARCHITECTURE.en-US.md`/`.pt-BR.md` §6.

```mermaid
flowchart TB
    AdminEvents["Admin (frontend) — /admin/events"] --> UE["GET /api/users/admin/events\n(users-api, nova tabela UserEvent)"]
    AdminEvents --> OAE["GET /api/orders/admin/events\n(orders-api, tabela OrderEvent existente)"]
    AdminEvents --> PAA["GET /api/payments/admin\n(payments-api, tabela Payment existente)"]
    AdminEvents --> NAA["GET /api/notifications/admin\n(notifications-api, tabela Notification existente)"]
```
