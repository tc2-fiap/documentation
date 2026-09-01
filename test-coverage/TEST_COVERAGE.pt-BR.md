[English](TEST_COVERAGE.en-US.md) · **Português**

# FIAP Games — Cobertura de Testes

Baseado em [`base-project/docs/DOCUMENTATION.md` §7.1](https://github.com/KainanGuerra/fiap-games/blob/main/docs/DOCUMENTATION.md), adaptado para cinco serviços de backend independentes em vez de uma única solução monolítica — não existe aqui uma única execução agregada de `dotnet test --collect`, já que cada serviço é sua própria solução, schema e projeto de testes.

## Como medir

```bash
cd <service>-api/tests/FiapGames.<Service>.Tests
dotnet test --collect:"XPlat Code Coverage"
```

(Sem o prefixo `repos/` — esse é o layout deste workspace específico, não o reproduzível. Conforme `notes.md` 50, `documentation` é o único repositório nunca clonado como irmão; os sete repositórios em execução, incluindo `<service>-api`, são clonados lado a lado em um único diretório pai, conforme `../getting-started/GETTING_STARTED.pt-BR.md` §1.)

Cada execução gera um relatório Cobertura em `TestResults/` daquele projeto; os números abaixo foram produzidos assim, uma execução por serviço, em 01/09/2026.

## Cobertura de linhas agregada: 16,0%

1.184 de 7.392 linhas, somadas entre os cinco relatórios Cobertura (não uma média das cinco porcentagens, já que os serviços estão longe de ter o mesmo tamanho).

| Serviço | Cobertura | Testes |
|---|---|---|
| `users-api` | 16,1% | 18 |
| `catalog-api` | 15,9% | 17 |
| `orders-api` | 8,3% | 30 |
| `payments-api` | 32,4% | 52 |
| `notifications-api` | 14,3% | 5 |
| **Total** | **16,0%** | **122** |

## Onde a cobertura está, e onde não está

Mesmo padrão do monolito que este sistema substituiu: a cobertura se concentra em `Application` (serviços, validadores) e nas entidades de `Domain` — toda suíte aqui mocka sua interface de repositório e nunca toca um Postgres ou RabbitMQ real. `Endpoints`, `Infrastructure/Persistence` (repositórios, `DbContext`, migrations do EF) e a infraestrutura de consumers/broadcaster ficam em ou perto de 0%, validados manualmente em vez disso — ao vivo contra o cluster kind no navegador, ou contra um container Postgres descartável para tudo relacionado a constraints (`notes.md` 52, 54).

**`payments-api` é o outlier, com mais do que o dobro do próximo serviço mais coberto.** Sua abstração de gateway — `SimulatedPaymentGateway`, `AbacatePayGateway`, `MercadoPagoGateway`, `PaymentGatewayChain`, `PaymentStatusPollingWorker` — é testada unitariamente com HTTP mockado via um `HttpMessageHandler` falso (`notes.md` 38), algo que nenhum outro serviço tem equivalente na sua camada `Infrastructure`.

**`orders-api` é o mais baixo, apesar de ter mais testes que qualquer outro serviço.** Sua camada `Infrastructure/Persistence` também é a maior — `OrderRepository`, `OrdersDbContext` e quatro migrations do EF (`InitialCreate`, `AddOrderEventAuditLog`, `MultiItemOrders`, `RemoveFromLibrary`) — todas deixadas em 0%, já que as propriedades que importam ali (um índice único parcial de verdade rejeitando uma compra duplicada concorrente, ou liberar um jogo removido para recompra) foram verificadas contra uma instância Postgres real e descartável, não contra um mock, conforme `notes.md` 52 e 54.

## Frontend

`repos/frontend` não tem suíte de testes automatizada (`npm run build`/`npm run lint` são seus únicos scripts — sem `vitest`/`jest`). Mudanças de UI são verificadas manualmente em um navegador ao vivo contra o cluster em execução, seguindo a regra já estabelecida deste projeto para trabalho de frontend.
