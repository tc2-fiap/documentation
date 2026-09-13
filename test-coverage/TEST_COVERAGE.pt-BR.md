[English](TEST_COVERAGE.en-US.md) · **Português**

# FIAP Games — Cobertura de Testes

Baseado em [`base-project/docs/DOCUMENTATION.md` §7.1](https://github.com/KainanGuerra/fiap-games/blob/main/docs/DOCUMENTATION.md), adaptado para seis serviços de backend independentes em vez de uma única solução monolítica — não existe aqui uma única execução agregada de `dotnet test --collect`, já que cada serviço é sua própria solução, schema (quando tem um) e projeto de testes.

## Como medir

```bash
cd <service>-api/tests/FiapGames.<Service>.Tests
dotnet test --collect:"XPlat Code Coverage"
```

(Sem o prefixo `repos/` — esse é o layout deste workspace específico, não o reproduzível. Conforme `notes.md` 50, `documentation` é o único repositório nunca clonado como irmão; os oito repositórios em execução, incluindo `<service>-api`, são clonados lado a lado em um único diretório pai, conforme `../getting-started/GETTING_STARTED.pt-BR.md` §1.)

Cada execução gera um relatório Cobertura em `TestResults/` daquele projeto; os números abaixo foram produzidos assim, uma execução por serviço, em 13/09/2026 (remedido após a adição do `platform-api` — ver `notes.md` 75).

## Cobertura de linhas agregada: 14,9%

1.223 de 8.218 linhas, somadas entre os seis relatórios Cobertura (não uma média das seis porcentagens, já que os serviços estão longe de ter o mesmo tamanho).

| Serviço | Cobertura | Testes |
|---|---|---|
| `users-api` | 16,1% | 19 |
| `catalog-api` | 15,0% | 19 |
| `orders-api` | 8,7% | 35 |
| `payments-api` | 32,2% | 52 |
| `notifications-api` | 14,1% | 5 |
| `platform-api` | 2,1% | 2 |
| **Total** | **14,9%** | **132** |

## Onde a cobertura está, e onde não está

Mesmo padrão do monolito que este sistema substituiu: a cobertura se concentra em `Application` (serviços, validadores) e nas entidades de `Domain` — toda suíte aqui mocka sua interface de repositório e nunca toca um Postgres ou RabbitMQ real. `Endpoints`, `Infrastructure/Persistence` (repositórios, `DbContext`, migrations do EF) e a infraestrutura de consumers/broadcaster ficam em ou perto de 0%, validados manualmente em vez disso — ao vivo contra o cluster kind no navegador, ou contra um container Postgres descartável para tudo relacionado a constraints (`notes.md` 52, 54).

**`payments-api` é o outlier, com mais do que o dobro do próximo serviço mais coberto.** Sua abstração de gateway — `SimulatedPaymentGateway`, `AbacatePayGateway`, `MercadoPagoGateway`, `PaymentGatewayChain`, `PaymentStatusPollingWorker` — é testada unitariamente com HTTP mockado via um `HttpMessageHandler` falso (`notes.md` 38), algo que nenhum outro serviço tem equivalente na sua camada `Infrastructure`.

**`orders-api` tem mais testes que qualquer outro serviço, mas não é o de menor cobertura** — sua camada `Infrastructure/Persistence` é a maior entre os cinco serviços com banco de dados (`OrderRepository`, `OrdersDbContext` e quatro migrations do EF: `InitialCreate`, `AddOrderEventAuditLog`, `MultiItemOrders`, `RemoveFromLibrary`) — todas deixadas em 0%, já que as propriedades que importam ali (um índice único parcial de verdade rejeitando uma compra duplicada concorrente, ou liberar um jogo removido para recompra) foram verificadas contra uma instância Postgres real e descartável, não contra um mock, conforme `notes.md` 52 e 54.

**`platform-api` é o de fato mais baixo, por um motivo diferente de qualquer serviço com banco de dados.** Ele não tem camada `Domain`/`Infrastructure/Persistence` para reduzir o denominador (nenhum banco de dados — `notes.md` 75) — seus dois testes cobrem `PodService`, a única lógica de negócio que ele tem, mockando `IPodReader`. Todo o resto — `KubernetesPodReader` (o cliente real da API do Kubernetes) e `Endpoints/PlatformEndpoints.cs` — não é verificado por teste automatizado, do mesmo jeito que `Endpoints`/`Infrastructure` de todo outro serviço fica perto de 0%, e foi confirmado ao vivo contra o cluster em execução (dados reais de pods retornados através do Ingress com um token de admin, `403` com um token de player).

## Frontend

`repos/frontend` não tem suíte de testes automatizada (`npm run build`/`npm run lint` são seus únicos scripts — sem `vitest`/`jest`). Mudanças de UI são verificadas manualmente em um navegador ao vivo contra o cluster em execução, seguindo a regra já estabelecida deste projeto para trabalho de frontend.
