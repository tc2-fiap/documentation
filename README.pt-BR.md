[English](README.en-US.md) · **Português**

# FIAP Games

Um monólito modular em .NET, rearquitetado como um sistema distribuído: cinco serviços de backend, um frontend em React e um chart Helm de orquestração, rodando em Kubernetes local com mensageria orientada a eventos.

Este repositório — `documentation` — reúne a especificação, o registro de decisões e a documentação narrativa do projeto. Ele é lido no GitHub, não clonado: diferente dos sete repositórios abaixo, ele nunca é clonado como irmão para rodar nada, já que nada na execução do sistema precisa dele em disco (`notes.md` 50).

## O sistema distribuído

Sete repositórios independentes sob [`github.com/tc2-fiap`](https://github.com/tc2-fiap), cada um clonado como irmão lado a lado dos *outros seis* — nunca deste repositório. Cada um tem seu próprio `README.md` bilíngue.

| Repositório | Responsabilidade |
|---|---|
| [`users-api`](https://github.com/tc2-fiap/users-api) | Cadastro, login, login com Google, emissão de JWT, papéis (Player/Admin) |
| [`catalog-api`](https://github.com/tc2-fiap/catalog-api) | O catálogo de jogos — apenas dado de referência de produto, fora do fluxo de eventos de compra |
| [`orders-api`](https://github.com/tc2-fiap/orders-api) | O agregado `Order`, o ciclo de vida da compra, a biblioteca, o log de auditoria por pedido |
| [`payments-api`](https://github.com/tc2-fiap/payments-api) | A abstração do gateway de pagamento (gateway simulado determinístico por padrão), registros de pagamento persistidos |
| [`notifications-api`](https://github.com/tc2-fiap/notifications-api) | E-mails de boas-vindas e confirmações de compra — console por padrão, envio real via Resend opcionalmente |
| [`frontend`](https://github.com/tc2-fiap/frontend) | O app React — catálogo com capas de jogos, um passo de checkout real (QR code PIX quando um gateway real está ativo), biblioteca, uma seção de admin para a trilha de auditoria entre serviços, e uma alternância de idioma Inglês/Português que mostra R$ ou um preço em USD convertido ao vivo |
| [`orchestration`](https://github.com/tc2-fiap/orchestration) | O chart Helm guarda-chuva: Postgres, RabbitMQ, Ingress e todos os subcharts de serviço |

Para subir o sistema inteiro em um cluster Kubernetes local, clone os sete repositórios acima em um único diretório pai vazio (mantendo os nomes de pasta padrão — o chart Helm do `orchestration` espera que os outros seis sejam caminhos irmãos literais), depois, a partir de `orchestration/`:

```bash
helm dependency update
helm install fiap-games .
```

Isso sobe todos os oito pods (cinco serviços de backend + Postgres + RabbitMQ + frontend) a partir de um cluster limpo, sem nenhum reinício, acessados por uma única URL base de Ingress. Veja [`GETTING_STARTED.pt-BR.md`](getting-started/GETTING_STARTED.pt-BR.md) ([English](getting-started/GETTING_STARTED.en-US.md)) para o passo de clonagem completo, os pré-requisitos, a verificação e um passo a passo de demonstração.

Cada repositório de serviço também roda de forma independente via seu próprio `docker-compose.yml` — veja o `README.md` daquele repositório.

## O monólito de referência

O `base-project` — o monólito modular original em .NET 10 (Usuários + Jogos, JWT, MongoDB, Docker Compose, CI/CD, Terraform) que este sistema substitui — vive em [`github.com/KainanGuerra/fiap-games`](https://github.com/KainanGuerra/fiap-games), material de referência somente leitura, não modificado durante este projeto. Veja o próprio [`docs/DOCUMENTATION.md`](https://github.com/KainanGuerra/fiap-games/blob/main/docs/DOCUMENTATION.md) daquele repositório para entender como ele foi construído, e seu `README.md` para saber como executá-lo.

## Os documentos de design

Este repositório reúne toda especificação, registro de decisões e documentação narrativa do projeto (`notes.md` 34, 44, 46, 50, 55):

| Documento | Conteúdo | Idiomas |
|---|---|---|
| [`instructions.md`](spec/instructions.md) | A especificação — arquitetura, responsabilidades de cada serviço, contratos de eventos, requisitos de Kubernetes, critérios de aceitação (marcados conforme verificados) | Somente inglês |
| [`notes.md`](spec/notes.md) | Registro de decisões — por que cada escolha foi feita, o que foi rejeitado, e o que a reabriria | Somente inglês |
| [`bdd.md`](spec/bdd.md) | Cenários de aceitação em Gherkin cobrindo os fluxos de eventos, o ciclo de vida do pedido e a forma da implantação | Somente inglês |
| [`OVERVIEW.en-US.md`](context/OVERVIEW.en-US.md) / [`.pt-BR`](context/OVERVIEW.pt-BR.md) | Por que este sistema existe e o que ele entrega | Inglês, Português |
| [`ARCHITECTURE.en-US.md`](architecture/ARCHITECTURE.en-US.md) / [`.pt-BR`](architecture/ARCHITECTURE.pt-BR.md) | Como foi efetivamente construído — arquitetura da solução, diagramas de fluxo de eventos, detalhamento por serviço, implantação | Inglês, Português |
| [`GETTING_STARTED.en-US.md`](getting-started/GETTING_STARTED.en-US.md) / [`.pt-BR`](getting-started/GETTING_STARTED.pt-BR.md) | Pré-requisitos, subida do cluster, verificação, passo a passo de demonstração | Inglês, Português |
| [`TEST_COVERAGE.en-US.md`](test-coverage/TEST_COVERAGE.en-US.md) / [`.pt-BR`](test-coverage/TEST_COVERAGE.pt-BR.md) | Cobertura de linhas medida, por serviço | Inglês, Português |
| [`frontend/design/`](https://github.com/tc2-fiap/frontend/tree/main/design) | Identidade visual — tokens de cor, marca, logomarca, favicon — aplicados literalmente no frontend, mantidos no único repositório que os usa (`notes.md` 44) | Somente inglês |

`instructions.md` é *o quê*. `notes.md` é *o porquê*. `bdd.md` é *como você saberia que funciona*. `OVERVIEW.md` é *por que existe*, `ARCHITECTURE.md` é *o que de fato foi construído*. `instructions.md`/`notes.md`/`bdd.md` permanecem em um único idioma — são uma especificação e um log de engenharia somente-acréscimo escritos durante a construção, não documentação narrativa voltada ao leitor (`notes.md` 35).

### Notas de construção de funcionalidades

[`features/`](features/) começou como especificações de trabalho ainda não construído (`notes.md` 37) — ambas já foram implementadas, com seus banners de status atualizados no próprio arquivo em vez de deixados desatualizados. Ficam guardadas como notas de construção e referência de API de terceiros (rotas, mapeamento de vocabulário de status, o que já foi verificado ao vivo e o que ainda está pendente), em vez de incorporadas ao `ARCHITECTURE.md`, já que quem lê esse documento não precisa dos formatos brutos de request/response de terceiros. Somente inglês.

| Documento | Status |
|---|---|
| [`features/quotation-feature.md`](features/quotation-feature.md) | **Implementado** (`notes.md` 39) — `GET /api/quotations/usd-brl` no `catalog-api`, Frankfurter com fallback para ExchangeRate-API, somente para exibição |
| [`features/payment-gateway-simulate.md`](features/payment-gateway-simulate.md) | **Implementado, verificação em sandbox real pendente** (`notes.md` 38) — `AbacatePayGateway`/`MercadoPagoGateway` atrás de um `PaymentGatewayChain` ordenado. Existem dois caminhos de confirmação: um `BackgroundService` de polling com backoff exponencial, o caminho ativo nesta topologia local (sem Ingress público para um provedor chamar de volta); e um receptor de webhook (`POST /api/payments/webhooks/{provider}`, verificado por HMAC) totalmente construído e conectado para uma implantação real, apenas inativo aqui. Todas as chamadas HTTP dos gateways são mockadas nos testes — nada chamou um provedor real ainda. |

## Escolhas de design notáveis

Algumas que divergem da leitura óbvia dos requisitos, cada uma justificada em `notes.md`:

- **Um OrdersAPI dedicado.** O briefing original colocava o início da compra no catálogo; um catálogo é dado de referência de produto, um pedido é um agregado transacional. Eles são separados aqui, deliberadamente.
- **PostgreSQL, não MongoDB.** Uma mudança em relação ao projeto de referência, motivada pela necessidade de transações reais para um outbox transacional.
- **Um banco de dados, um schema e um role por serviço.** Fronteiras impostas por concessões do Postgres, não por disciplina do desenvolvedor — verificado ao vivo, não apenas afirmado.
- **Um gateway de pagamento substituível.** Simulação determinística por padrão (`notes.md` 4); `AbacatePayGateway` e `MercadoPagoGateway` também já foram construídos, compostos em uma cadeia de fallback ordenada atrás do valor de ConfigMap `PaymentGateway:Providers` — a verificação em sandbox real contra uma conta de provedor de verdade é a única coisa ainda pendente (`notes.md` 38).
- **Uma trilha de auditoria de admin composta na camada de visualização.** Um admin pode ver todo pedido, seus eventos, seu registro de pagamento e suas notificações — cada serviço expondo apenas seu próprio dado através do seu próprio endpoint de admin, nunca uma consulta entre schemas.
- **Login com Google sem um fluxo de redirecionamento OAuth.** A verificação de ID token não precisa de uma URL pública de callback, contornando o mesmo problema de túnel que mantém o gateway de pagamento real como opcional.
- **A documentação vive em seu próprio repositório, publicado separadamente e nunca clonado junto com os sete repositórios de execução** — sua camada narrativa (este arquivo, `OVERVIEW.md`, `ARCHITECTURE.md`, `GETTING_STARTED.md`, `TEST_COVERAGE.md`, o `README.md` de cada repositório) é bilíngue inglês/português; a especificação e o registro de decisões permanecem somente em inglês (`notes.md` 34, 35, 44, 50).

## Contexto

Projeto acadêmico (FIAP). O briefing original de requisitos está em português; esta documentação é bilíngue (inglês/português) em sua camada narrativa, e somente em inglês para a especificação técnica e o registro de decisões.
