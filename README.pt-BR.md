[English](README.en-US.md) · **Português**

# FIAP Games

Duas coisas vivem aqui: uma **implementação de referência finalizada**, e o **sistema distribuído que a substitui** — já construído e em funcionamento.

## O que foi construído

**`base-project/`** — o monólito modular original em .NET 10 (Usuários + Jogos, JWT, MongoDB, Docker Compose, CI/CD, Terraform), também espelhado publicamente em [`github.com/KainanGuerra/fiap-games`](https://github.com/KainanGuerra/fiap-games). Material de referência somente leitura; não é modificado. Os comandos abaixo rodam a cópia local já presente neste workspace:

```bash
cd base-project
cp .env.example .env
docker compose up --build      # API na :8080, Swagger em :8080/swagger
```

**`repos/`** — o sistema distribuído para o qual ele foi rearquitetado: cinco serviços de backend em .NET, um frontend em React + Vite, e um repositório `orchestration` com o chart Helm guarda-chuva. Roda em um cluster Kubernetes local via `kind`. Todo repositório abaixo tem seu próprio `README.md` bilíngue.

| Repositório | Responsabilidade |
|---|---|
| [`users-api`](https://github.com/tc2-fiap/users-api) | Cadastro, login, login com Google, emissão de JWT, papéis (Player/Admin) |
| [`catalog-api`](https://github.com/tc2-fiap/catalog-api) | O catálogo de jogos — apenas dado de referência de produto, fora do fluxo de eventos de compra |
| [`orders-api`](https://github.com/tc2-fiap/orders-api) | O agregado `Order`, o ciclo de vida da compra, a biblioteca, o log de auditoria por pedido |
| [`payments-api`](https://github.com/tc2-fiap/payments-api) | A abstração do gateway de pagamento (gateway simulado determinístico por padrão), registros de pagamento persistidos |
| [`notifications-api`](https://github.com/tc2-fiap/notifications-api) | E-mails de boas-vindas e confirmações de compra — console por padrão, envio real via Resend opcionalmente |
| [`frontend`](https://github.com/tc2-fiap/frontend) | O app React — catálogo com capas de jogos, um passo de checkout real (QR code PIX quando um gateway real está ativo), biblioteca, uma seção de admin para a trilha de auditoria entre serviços, e uma alternância de idioma Inglês/Português que mostra R$ ou um preço em USD convertido ao vivo |
| [`orchestration`](https://github.com/tc2-fiap/orchestration) | O chart Helm guarda-chuva: Postgres, RabbitMQ, Ingress e todos os subcharts de serviço |

```bash
cd repos/orchestration
helm install fiap-games .
```

Sobe todos os oito pods (cinco serviços de backend + Postgres + RabbitMQ + frontend) a partir de um cluster limpo, sem nenhum reinício. O frontend e todo backend são acessados por uma única URL base de Ingress. Veja [`GETTING_STARTED.pt-BR.md`](narrative/GETTING_STARTED.pt-BR.md) ([English](narrative/GETTING_STARTED.en-US.md)) para os pré-requisitos, a sequência completa de subida e um passo a passo de demonstração.

Cada repositório de serviço também roda de forma independente via seu próprio `docker-compose.yml` — veja o `README.md` daquele repositório.

## Os documentos de design

Este repositório — `repos/documentation/` — reúne toda especificação, registro de decisões e documentação narrativa do projeto (`notes.md` 34, 44):

| Documento | Conteúdo | Idiomas |
|---|---|---|
| [`instructions.md`](spec/instructions.md) | A especificação — arquitetura, responsabilidades de cada serviço, contratos de eventos, requisitos de Kubernetes, critérios de aceitação (marcados conforme verificados) | Somente inglês |
| [`notes.md`](spec/notes.md) | Registro de decisões — por que cada escolha foi feita, o que foi rejeitado, e o que a reabriria | Somente inglês |
| [`bdd.md`](spec/bdd.md) | Cenários de aceitação em Gherkin cobrindo os fluxos de eventos, o ciclo de vida do pedido e a forma da implantação | Somente inglês |
| [`DOCUMENTATION.en-US.md`](narrative/DOCUMENTATION.en-US.md) / [`.pt-BR`](narrative/DOCUMENTATION.pt-BR.md) | O que foi efetivamente construído — arquitetura, diagramas de fluxo de eventos, detalhamento por serviço | Inglês, Português |
| [`GETTING_STARTED.en-US.md`](narrative/GETTING_STARTED.en-US.md) / [`.pt-BR`](narrative/GETTING_STARTED.pt-BR.md) | Pré-requisitos, subida do cluster, verificação, passo a passo de demonstração | Inglês, Português |
| [`../../CLAUDE.md`](../../CLAUDE.md) | Contexto de trabalho para sessões de agente de IA | Somente inglês |
| [`frontend/design/`](https://github.com/tc2-fiap/frontend/tree/main/design) | Identidade visual — tokens de cor, marca, logomarca, favicon — aplicados literalmente no frontend, mantidos no único repositório que os usa (`notes.md` 44) | Somente inglês |

`instructions.md` é *o quê*. `notes.md` é *o porquê*. `bdd.md` é *como você saberia que funciona*. `DOCUMENTATION.md` é *o que de fato foi construído*. `instructions.md`/`notes.md`/`bdd.md` permanecem em um único idioma — são uma especificação e um log de engenharia somente-acréscimo escritos durante a construção, não documentação narrativa voltada ao leitor (`notes.md` 35).

### Funcionalidades futuras

[`features/`](features/) reúne especificações de trabalho ainda não construído — cada arquivo declara seu próprio status "não implementado" logo no início. Somente inglês; não são documentação do sistema entregue (`notes.md` 37).

| Documento | O que especifica |
|---|---|
| [`features/payment-gateway-simulate.md`](features/payment-gateway-simulate.md) | Um gateway de pagamento sandbox real AbacatePay/Mercado Pago, atrás da interface `IPaymentGateway` já existente |
| [`features/quotation-feature.md`](features/quotation-feature.md) | Uma exibição opcional de preço equivalente em USD ao lado do R$, somente informativa |

## Escolhas de design notáveis

Algumas que divergem da leitura óbvia dos requisitos, cada uma justificada em `notes.md`:

- **Um OrdersAPI dedicado.** O briefing original colocava o início da compra no catálogo; um catálogo é dado de referência de produto, um pedido é um agregado transacional. Eles são separados aqui, deliberadamente.
- **PostgreSQL, não MongoDB.** Uma mudança em relação ao projeto de referência, motivada pela necessidade de transações reais para um outbox transacional.
- **Um banco de dados, um schema e um role por serviço.** Fronteiras impostas por concessões do Postgres, não por disciplina do desenvolvedor — verificado ao vivo, não apenas afirmado.
- **Um gateway de pagamento substituível.** Simulação determinística por padrão; uma integração sandbox real opcional com o AbacatePay atrás de uma flag de ConfigMap permanece não construída (`notes.md` 4).
- **Uma trilha de auditoria de admin composta na camada de visualização.** Um admin pode ver todo pedido, seus eventos, seu registro de pagamento e suas notificações — cada serviço expondo apenas seu próprio dado através do seu próprio endpoint de admin, nunca uma consulta entre schemas.
- **Login com Google sem um fluxo de redirecionamento OAuth.** A verificação de ID token não precisa de uma URL pública de callback, contornando o mesmo problema de túnel que mantém o gateway de pagamento real como opcional.
- **A documentação vive em seu próprio repositório (`repos/documentation/`), não dentro de nenhum repositório de serviço específico**, e sua camada narrativa (este arquivo, `DOCUMENTATION.md`, `GETTING_STARTED.md`, o `README.md` de cada repositório) é bilíngue inglês/português — a especificação e o registro de decisões permanecem somente em inglês (`notes.md` 34, 35, 44).

## Contexto

Projeto acadêmico (FIAP). O briefing original de requisitos está em português; esta documentação é bilíngue (inglês/português) em sua camada narrativa, e somente em inglês para a especificação técnica e o registro de decisões.
