# Fluxo de cadastro

`UserCreatedEvent`, do cadastro até o e-mail de boas-vindas, com o dedupe do `notifications-api` cobrindo uma reentrega. Referenciado em `../architecture/ARCHITECTURE.en-US.md`/`.pt-BR.md` §5.1.

```mermaid
sequenceDiagram
    participant U as users-api
    participant R as RabbitMQ
    participant N as notifications-api

    U->>U: cria User (Postgres)
    U->>R: publica UserCreatedEvent
    R->>N: UserCreatedEvent
    N->>N: upsert de UserProjection
    N->>N: TryClaim da chave de dedupe ("user-created:{UserId}")
    alt primeira entrega
        N->>N: envia e-mail de boas-vindas (console ou Resend)
    else reentrega
        N-->>N: ignora — já enviado
    end
```
