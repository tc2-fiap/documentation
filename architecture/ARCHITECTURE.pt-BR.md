[English](ARCHITECTURE.en-US.md) · **Português**

# FIAP Games — Arquitetura

Veja [`../context/OVERVIEW.pt-BR.md`](../context/OVERVIEW.pt-BR.md) para entender por que este sistema existe e o que ele entrega; este documento é o aprofundamento técnico — como ele foi de fato construído.

## 1. Arquitetura da solução

```
users-api/            # cadastro, login, login com Google, emissão de JWT, papéis
catalog-api/          # o catálogo de jogos — apenas dado de referência de produto
orders-api/           # o agregado Order, o ciclo de vida da compra, a biblioteca, o log de auditoria
payments-api/         # abstração do gateway de pagamento, registros de pagamento persistidos
notifications-api/    # e-mails de boas-vindas, confirmações de compra (console ou Resend)
frontend/             # React + Vite — o único serviço acessado diretamente pelo navegador; também guarda o design/, os ativos de marca
orchestration/        # Postgres, RabbitMQ, Ingress, chart Helm guarda-chuva
```

Sete repositórios independentes sob [`github.com/tc2-fiap`](https://github.com/tc2-fiap), clonados como irmãos, lado a lado — veja `GETTING_STARTED.md` §1. O `documentation` (este repositório) é publicado separadamente (também em `github.com/tc2-fiap/documentation`) e não faz parte desse layout local — é material de referência, não algo que o sistema em execução ou sua build precisem em disco (`notes.md` 47, 49).

Cada um desses oito repositórios tem seu próprio `README.md` (`notes.md` 34, 44). Cada repositório de backend segue a mesma camada interna usada pelos módulos do `base-project` — `Domain` → `Application` → `Infrastructure`, com `Endpoints` na camada mais externa, dependências apontando para dentro — mas como a estrutura do repositório *inteiro*, não um módulo dentro de um host compartilhado. Código de kernel livre de framework (`Result`, `Entity`, `IRepository`, paginação) e infraestrutura de JWT/tratamento de erros são **duplicados por serviço**, não empacotados (`notes.md` 21) — cinco cópias de ~13 arquivos pequenos, escolha feita em vez de um pacote NuGet compartilhado especificamente para evitar reintroduzir o acoplamento que a separação pretendia eliminar.

Veja o diagrama: [Topologia dos serviços](../diagrams/service-topology.md).

Toda seta em direção ao Postgres usa um role restrito a exatamente um schema — uma consulta entre schemas é recusada pelo próprio Postgres, verificado ao vivo ao tentar uma com as credenciais de outro serviço (`instructions.md` §14).

## 2. Modelo de domínio

| Serviço | Agregado/entidade | Campos principais |
|---|---|---|
| `users-api` | `User` | Id, Name, Email (único), PasswordHash (anulável — nulo para contas somente-Google), GoogleSubjectId, Role (`Player`/`Admin`) |
| `users-api` | `UserEvent` | Id, UserId, EventType, Payload (JSON bruto), OccurredAtUtc — o log de auditoria de eventos de todo o sistema (`notes.md` 43) |
| `catalog-api` | `Game` | Id, Title, Genre, Platform, Description, Price, ReleaseDate, CoverImageUrl (anulável — só exibição, ver §5.3) |
| `orders-api` | `Order` | Id, UserId, Status (`Pending → Paid \| Failed`, unidirecional), CreatedAtUtc, UpdatedAtUtc — um checkout de carrinho registra um único `Order` para vários jogos, então preço e identidade por jogo vivem em `OrderItem`, não aqui (`notes.md` 51) |
| `orders-api` | `OrderItem` | Id, OrderId, GameId, Price (**snapshot**, nunca ao vivo); UserId e Status são cópias desnormalizadas do `Order` pai, mantidas em sincronia por `Order.MarkPaid`/`MarkFailed` — necessárias só para viabilizar as duas restrições de banco abaixo; `Order.Status` continua sendo a fonte única da verdade (`notes.md` 51, 52); RemovedFromLibraryAtUtc (opcional, unidirecional) marca o item como removido da biblioteca — um campo independente de `Status`, que nunca reverte o pedido (`notes.md` 54) |
| `orders-api` | `OrderEvent` | Id, OrderId, EventType, Payload (JSON bruto), OccurredAtUtc — o log de auditoria por pedido |
| `payments-api` | `Payment` | Id, OrderId (único), UserId, Price, Status (`Processing`/`Approved`/`Rejected` — só interno, nunca trafega entre serviços), Gateway, ExternalReference, RequestPayload, ResponsePayload, PixCopyPasteCode, PixQrCodeBase64 (ambos anuláveis — só gateways PIX reais, ver §5.3), NextPollAtUtc, PollAttempts |
| `notifications-api` | `Notification` | Id, DedupeKey (único), Type, UserId, OrderId (anulável), Recipient, Subject, Body, Channel, Status, ProviderRequestPayload, ProviderResponsePayload |
| `notifications-api` | `UserProjection` | UserId (chave), Name, Email — um read-model local, não uma leitura entre serviços (ver §5) |

## 3. Stack tecnológico

- **.NET 10**, ASP.NET Core Minimal APIs.
- **PostgreSQL** via EF Core/Npgsql — uma instância, um schema e um role por serviço (`notes.md` 13, 14).
- **RabbitMQ** via **MassTransit** — escolhido em vez do `RabbitMQ.Client` bruto especificamente pelo seu outbox transacional com EF Core (`notes.md` 12, 15, 22).
- **JWT** com um segredo simétrico compartilhado (`notes.md` 19) — `users-api` é o único emissor, cada serviço valida de forma independente.
- **Google Identity Services** (verificação de ID token, não o fluxo de redirecionamento OAuth) para login opcional (`notes.md` 28).
- **FluentValidation**, **Serilog** (logging estruturado em JSON), **BCrypt** para hash de senha.
- **xUnit** + **NSubstitute** — todo teste é de camada de serviço com repositórios mockados; nenhum banco ou broker ao vivo em nenhuma suíte de testes.
- **React 19 + Vite + TypeScript**, `react-router-dom`, sem framework de UI — o frontend chama apenas caminhos relativos `/api/*`.
- **Um `LocaleContext` feito à mão** (alternância de UI Inglês/Português, espelhando o padrão já existente do `AuthContext` em vez de adicionar uma biblioteca de i18n) e `Intl.NumberFormat`, um formatador para `pt-BR`/BRL e outro para `en-US`/USD, para exibição de preços (`notes.md` 35, 36, 39) — todo preço no backend continua sendo um `decimal` em BRL; o valor em USD é uma conversão só para exibição, nunca um valor que algum serviço armazena ou confia.
- **Helm** — um subchart por repositório, um chart guarda-chuva em `orchestration` (`notes.md` 20).
- **kind** — o cluster Kubernetes local (`notes.md`, conforme o pré-requisito de ambiente do plano).
- **Docker Compose** — cada repositório de serviço também roda de forma independente, sem o cluster (`notes.md` 24).

## 4. Persistência

Diferente do MongoDB sem schema do `base-project` (apenas migrações de índice), este sistema usa **migrações relacionais comuns do EF Core** por serviço, com a tabela `__EFMigrationsHistory` de cada serviço vivendo dentro do seu próprio schema. As migrações rodam automaticamente na inicialização do container (`db.Database.Migrate()`), protegidas por um init container `wait-for-postgres` em todo serviço com banco — sem ele, uma subida verdadeiramente fria do cluster entra em corrida com o Postgres e o processo do serviço quebra antes que o Postgres esteja pronto para aceitar conexões (`notes.md` 25).

O `orders-api` também carrega as tabelas do **outbox transacional do MassTransit com EF Core** (`InboxState`, `OutboxMessage`, `OutboxState`) no seu próprio schema — o mecanismo que torna atômicos a escrita do pedido e o `OrderPlacedEvent` (§5).

## 5. Comunicação orientada a eventos

Dois fluxos, ambos contratos fixos entre serviços (`instructions.md` §8) — renomear ou remodelar qualquer um deles casualmente quebra cinco serviços de uma vez. O contrato do fluxo de compra *foi* deliberadamente remodelado uma vez, para suportar pedidos com múltiplos itens — a única exceção documentada a essa regra, não um desvio casual (`notes.md` 51).

### 5.1 Cadastro

Veja o diagrama: [Fluxo de cadastro](../diagrams/registration-flow.md).

### 5.2 Compra

Veja o diagrama: [Fluxo de compra](../diagrams/purchase-flow.md).

A leitura síncrona de preço (§ Orders → Catalog) é uma consulta que acontece *antes* do fluxo começar, uma vez por jogo pedido — o fluxo em si nunca bloqueia esperando outro serviço depois que o `OrderPlacedEvent` é publicado (`instructions.md` §6, §8). Antes até dessa leitura, o `OrderService.CreateAsync` verifica `IOrderRepository.GetConflictingGameIdsAsync(userId, gameIds)` — um usuário que já possui qualquer jogo pedido (`Paid`) ou tem uma compra em andamento para ele (`Pending`) tem o pedido *inteiro* rejeitado com `409 Conflict`, tudo ou nada em vez de cumprimento parcial; um item anterior `Failed` nunca bloqueia uma nova tentativa (`notes.md` 42). Essa verificação sozinha é vulnerável a condição de corrida (TOCTOU) sob requisições concorrentes, então ela é reforçada por dois índices únicos reais do Postgres em `order_items` — `(order_id, game_id)` e um índice parcial `(user_id, game_id) WHERE status <> 'Failed'` — que o `OrderService.CreateAsync` também precisa capturar como uma violação de unicidade (`DbUpdateException`/`PostgresException`) e traduzir para o mesmo `409`, para o caso raro de duas requisições concorrentes passarem ambas pela verificação da aplicação (`notes.md` 52, verificado contra uma instância real do Postgres).

**Idempotência, na prática.** Todo consumer acima ou verifica uma chave de dedupe antes de agir (`notifications-api`, nos dois eventos) ou é naturalmente idempotente pelo próprio estado do domínio (`MarkPaid`/`MarkFailed` do `orders-api` viram no-op assim que o pedido sai de `Pending`; `payments-api` verifica se já existe um `Payment` antes de cobrar). Verificado ao vivo com uma ferramenta descartável de replay do MassTransit: reentregar qualquer um desses três eventos produz zero efeitos duplicados — nenhum segundo e-mail, nenhum segundo `Payment`, nenhum pedido revertendo de estado.

**Quando um gateway real está configurado** (`PaymentGateway:Providers` inclui `abacatepay` e/ou `mercadopago`), o passo `P->>P: IPaymentGateway.ChargeAsync` do diagrama de fluxo de compra passa a resolver via `PaymentGatewayChain` e pode retornar `Processing` em vez de um resultado final — o `payments-api` ainda persiste a linha `Payment`, mas **não** publica `PaymentProcessedEvent` ainda. Um `PaymentStatusPollingService` em background então consulta o provedor real com backoff exponencial até resolver (ou expirar em um `Rejected` forçado), publicando só então. O `Order` permanece `Pending` o tempo todo do ponto de vista do `orders-api` — nenhum contrato de evento mudou. Veja `notes.md` 38 para o design completo, incluindo por que o polling substitui o receptor de webhook, que está construído mas atualmente inativo.

### 5.3 Exibição da cotação, o checkout e o status ao vivo (adições síncronas/HTTP, sem eventos novos)

Quatro adições posteriores, puramente síncronas/HTTP, convivem com os fluxos acima sem mudar nenhum dos dois contratos de evento do broker (`notes.md` 39, 40, 53, 54):

- **`GET /api/quotations/usd-brl`** (`catalog-api`) faz proxy de uma cotação USD→BRL ao vivo — Frankfurter primeiro, ExchangeRate-API como fallback automático, ambos sem chave, com a cotação resolvida cacheada em memória por uma hora. O frontend é o único consumidor que converte algo: o preço em BRL de um jogo só é exibido em USD quando o idioma da interface está em inglês, voltando ao BRL nativo se a cotação estiver indisponível. `Game.Price`/`Order.Price`/`Payment.Price` nunca mudam de significado — isto é só exibição, do início ao fim.
- **`GET /api/payments/checkout/{orderId}`** (`payments-api`) é uma segunda rota, voltada ao jogador, ao lado da já existente `GET /api/payments/{orderId}` (admin-only): qualquer usuário autenticado pode chamá-la para o *próprio* pedido (posse verificada contra `Payment.UserId`; uma divergência retorna `404`), recebendo uma visão restrita (status, gateway, preço e — só quando um gateway PIX real gerou um — uma imagem de QR code e o código copia-e-cola) em vez dos payloads brutos completos da rota de admin. O `Order` em si não carrega nenhum detalhe do jogo além dos `GameId`s dos seus itens, então o frontend resolve o título de cada jogo comprado separadamente no `catalog-api`. Essa chamada agora acontece uma única vez, quando o pedido é observado pela primeira vez como `Pending` (com uma nova tentativa curta e limitada, já que a linha `Payment` é criada de forma assíncrona e pode ainda não existir), não repetidamente — veja o item de SSE abaixo para saber por que não existe mais um ciclo periódico para prender essa chamada. Com o gateway padrão `simulated` não há nada para escanear; o pedido ainda assim se resolve sozinho pelo mecanismo de atraso já existente.
- **`GET /api/orders/{id}/stream`** (`orders-api`, Server-Sent Events) entrega a transição `Pending → Paid | Failed` para a página de status do pedido no frontend sem polling: envia o status atual do pedido imediatamente, depois mais uma atualização no momento em que o pedido sai de `Pending`, e então encerra. Sustentado por um novo singleton `IOrderStatusBroadcaster` (um canal `System.Threading.Channels` por `OrderId` em andamento), alimentado pelo `PaymentProcessedConsumer` logo depois que ele confirma uma transição para `Paid`/`Failed` — implementado com o suporte nativo do ASP.NET Core a SSE em Minimal APIs (`TypedResults.ServerSentEvents`, já disponível no target `net10.0` do serviço, sem pacote novo). O frontend não pode usar a API `EventSource` do navegador aqui, já que ela não consegue enviar um cabeçalho `Authorization` e colocar o JWT na URL vazaria para logs do servidor e para o histórico do navegador — em vez disso, um leitor feito à mão (`src/api/sse.ts`) envia a requisição via `fetch` com o mesmo cabeçalho de bearer de qualquer outra chamada e interpreta sozinho os frames `text/event-stream`. Isso só é seguro como mecanismo em memória, por pod, porque o `orders-api` roda em uma única réplica; escalá-lo horizontalmente exigiria um backplane de verdade (por exemplo, uma fila do RabbitMQ em modo fanout) em vez disso. O Ingress (`repos/orchestration/templates/ingress.yaml`) precisou de `nginx.ingress.kubernetes.io/proxy-buffering: "off"` e de `proxy-read-timeout`/`proxy-send-timeout` maiores (`"120"`, acima do padrão de 60s do nginx) — ambos fatais para uma conexão de longa duração, e antes ausentes já que o Ingress não tinha nenhuma anotação. Veja `notes.md` 53.
- **`DELETE /api/library/{gameId}`** (`orders-api`, confirmado no frontend por um modal) remove um jogo da biblioteca do usuário. Define `OrderItem.RemovedFromLibraryAtUtc` — um campo totalmente separado de `Order.Status`/`OrderItem.Status`, então o pedido subjacente continua `Paid` no registro e nenhum `Payment` é alterado; é uma decisão de visibilidade da biblioteca, não um reembolso. O mesmo campo é excluído do índice único parcial `order_items(user_id, game_id)` (`notes.md` 52), então remover um jogo libera imediatamente uma recompra a preço cheio. Veja `notes.md` 54.

## 6. RBAC e a trilha de auditoria entre serviços

Adicionado depois do fluxo de compra principal (`notes.md` 26–30, 32), quando surgiu o requisito de que um admin precisa ver, para qualquer pedido, seu ciclo de vida completo nos três serviços que o tocam — não apenas um resumo.

O `users-api` semeia uma conta `Admin` na inicialização a partir de configuração vinda de um Secret (promoção exige um admin existente, então o primeiro não pode ser self-service); todo outro admin é promovido via `PUT /api/users/{id}/role`. Todo endpoint de admin é `RequireAuthorization(Roles: "Admin")` — confirmado ao vivo retornando `403` para um token não-admin e `401` sem autenticação.

A visão de auditoria é **composta na camada de visualização, nunca unida na camada de banco de dados** — consultas entre schemas continuam recusadas. Quatro endpoints independentes, admin-only, cada um retornando o dado do seu próprio serviço com o payload *real* trocado, não um resumo:

Veja o diagrama: [Trilha de auditoria por pedido](../diagrams/audit-trail.md).

A única peça arquitetural nova exigida por isso: o `UserProjection` do `notifications-api`, um read-model local mantido atualizado a partir do `UserCreatedEvent`, porque o contrato fixo do `PaymentProcessedEvent` carrega apenas um `UserId`, e este serviço precisa de um endereço de e-mail para de fato enviar (`notes.md` 30).

**Eventos de todo o sistema, não só de um pedido (`notes.md` 43).** A visão por pedido acima responde "o que aconteceu com este pedido" — ela não responde "mostre todo `UserCreatedEvent` já publicado". Uma segunda página de admin, `/admin/events`, faz isso: filtrável por origem, tipo, categoria e intervalo de datas, composta da mesma forma a partir de quatro endpoints *diferentes* de admin "listar tudo":

Veja o diagrama: [Eventos de todo o sistema](../diagrams/system-events.md).

Só o `users-api` precisou de uma tabela nova (`UserEvent`) — era o único serviço sem nenhum registro de que o `UserCreatedEvent` havia sido publicado. `orders-api`, `payments-api` e `notifications-api` já tinham uma tabela com tudo o que era necessário (`OrderEvent`, `Payment` e `Notification`, respectivamente); cada um só ganhou uma nova consulta paginada e sem escopo por pedido sobre dados que já persistia. O `catalog-api` está ausente dos dois diagramas — ele não publica nem consome nada.

## 7. Qualidade: testes, tratamento de erros e observabilidade

- **Suítes de teste de backend** — todas na camada de serviço com repositório mockado, nenhum Postgres ou RabbitMQ ao vivo em nenhuma suíte. A fatia do `orders-api` cresceu com a migração para pedidos com múltiplos itens: cobertura para registrar um pedido com vários jogos, o caminho de rejeição do pedido inteiro por qualquer conflito, o comportamento do `OrderStatusBroadcaster` por trás do endpoint de SSE (`notes.md` 51, 53), e a remoção definitiva de `RemoveFromLibrary` mais seu caminho na camada de serviço `OrderService.RemoveFromLibraryAsync` (`notes.md` 54). A regra determinística de preço do `payments-api` é testada exatamente no limite de `999.00`; o mapeamento de vocabulário do `AbacatePayGateway`/`MercadoPagoGateway` e a extração dos campos do QR code PIX, o comportamento de fallback do `PaymentGatewayChain`, a lógica de backoff/timeout do `PaymentStatusPollingWorker` e o fallback/cache do `QuotationService` do `catalog-api` também têm cobertura (HTTP mockado via um `HttpMessageHandler` falso, nenhuma chamada real ao sandbox — veja `notes.md` 38, 39). Veja [`../test-coverage/TEST_COVERAGE.pt-BR.md`](../test-coverage/TEST_COVERAGE.pt-BR.md) para a contagem real de testes e a cobertura de linhas medida por serviço — mantida em um único lugar em vez de repetida (e potencialmente desatualizada) aqui também.
- **Tratamento global de exceções**: um `GlobalExceptionHandler` idêntico (copiado por serviço) captura qualquer coisa não tratada, registra tudo no lado do servidor e retorna um `ProblemDetails` genérico com um trace id — nunca um stack trace para o cliente.
- **Logging estruturado**: cada serviço emite uma linha de log JSON por requisição (Serilog + `CompactJsonFormatter`), e toda publicação/consumo registra `{OrderId}` (ou `{UserId}` no cadastro) como propriedade nomeada — confirmado ao vivo que um único `OrderId` rastreia uma compra pelos logs de `orders-api`, `payments-api` e `notifications-api`.
- **Padrão Result**: toda falha esperada (não encontrado, conflito, não autorizado, validação) é um `Result`/`Result<T>`, mapeado para um status HTTP pela extensão compartilhada `ToHttpResult()` — nunca uma exceção para um caso que o chamador deveria razoavelmente esperar.

## 8. Implantação

- **Helm**: um subchart por repositório (`Chart.yaml`, `values.yaml`, `templates/`), `orchestration` como o chart guarda-chuva dependendo de todos os seis (`notes.md` 20).
- **kind**: o cluster local, configurado com `extraPortMappings` e o label de nó `ingress-ready` para que o nginx-ingress possa se ligar diretamente às portas 80/443 do host.
- **Roteamento do Ingress** (uma única URL base):

  | Caminho | Serviço |
  |---|---|
  | `/api/users/*` | `users-api` |
  | `/api/games/*`, `/api/quotations/*` | `catalog-api` |
  | `/api/orders/*`, `/api/library` | `orders-api`; `/api/orders/{id}/stream` é o endpoint de Server-Sent Events por trás da UI de status de pedido ao vivo (`notes.md` 53); `DELETE /api/library/{gameId}` remove um jogo da biblioteca do usuário e libera a recompra, sem reverter o pedido `Paid` subjacente (`notes.md` 54) |
  | `/api/payments/{orderId}`, `/api/payments/admin` | `payments-api`, admin-only; `/api/payments/checkout/{orderId}` é uma rota separada, voltada ao jogador e com verificação de posse, no mesmo prefixo (`notes.md` 40); `/api/payments/webhooks/{provider}` está construído e conectado, mas inativo — esta topologia local confirma cobranças de gateway real por polling, não por webhook; veja `notes.md` 38 |
  | `/api/notifications/*` | `notifications-api` (admin-only) |
  | `/*` | `frontend` |

- **Anotações do Ingress**: o único recurso Ingress não tinha nenhuma até o endpoint de SSE precisar delas — `nginx.ingress.kubernetes.io/proxy-buffering: "off"` e `proxy-read-timeout`/`proxy-send-timeout: "120"`, já que o buffering e o timeout padrão de 60s do nginx quebrariam uma conexão de longa duração em `/api/orders/{id}/stream`. Elas se aplicam a todo o sistema (um único Ingress para todas as rotas), mas são inofensivas para o tráfego comum de requisição/resposta (`notes.md` 53).
- **Auditoria de secrets**: `helm template | grep -iE "password\|secret\|apikey"` só expõe valores literais dentro dos próprios recursos `Secret` — toda env var de `Deployment` que referencia um deles usa `secretKeyRef`, confirmado em todos os seis charts.
- **Dois ambientes de execução** (`notes.md` 24): todo repositório de backend também carrega seu próprio `docker-compose.yml`, subindo apenas aquele serviço mais a infraestrutura que ele sozinho precisa, tornando-o desenvolvível de forma independente, sem o cluster.
- **CI/CD**: cada um dos seis repositórios carrega seu próprio `.github/workflows/ci.yml`, adaptado do formato de dois jobs do [`base-project/.github/workflows/ci-cd.yml`](https://github.com/KainanGuerra/fiap-games/blob/main/.github/workflows/ci-cd.yml): `build-and-test` (restore/build/test para os repositórios .NET, install/lint/build para o frontend) roda em todo push e PR; `docker-build-and-push` roda apenas em push para `main`, depois que o primeiro job passa, publicando no GHCR. Nenhum deles rodou durante a construção em si — a raiz deste workspace nunca foi inicializada com `git init` durante essa fase, deliberadamente. Desde então, todos os sete repositórios em execução foram publicados individualmente no GitHub sob uma organização dedicada, `tc2-fiap` (`notes.md` 47). Cada um veio inicialmente com o branch padrão `master` (padrão local do `git init`), enquanto esse gatilho é restrito ao `main` — então o CI nunca disparou silenciosamente em nenhum deles; o branch padrão de cada repositório foi renomeado para `main` logo depois, especificamente para corrigir isso (`notes.md` 48), e o CI agora roda a cada push. O `documentation` (este repositório) foi publicado mais tarde ainda, direto em `main`, mas não carrega workflow de CI próprio (`notes.md` 49).
