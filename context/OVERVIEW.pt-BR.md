[English](OVERVIEW.en-US.md) · **Português**

# FIAP Games — Visão Geral

## 1. Introdução

Este documento descreve o que foi efetivamente construído quando o [`base-project/`](https://github.com/KainanGuerra/fiap-games) — um monólito modular em .NET responsável por Usuários e Jogos — foi rearquitetado como um sistema distribuído: cinco serviços implantáveis de forma independente, um fluxo de compra orientado a eventos, isolamento de dados por serviço e um frontend em React, tudo rodando em um cluster Kubernetes local.

O foco aqui é o mesmo do [`base-project/docs/DOCUMENTATION.md`](https://github.com/KainanGuerra/fiap-games/blob/main/docs/DOCUMENTATION.md): não apenas *o quê* foi construído, mas *como*, e por que tomou essa forma. O restante dessa história — o aprofundamento técnico — mora logo ao lado, em `architecture/`.

### Navegação

| Arquivo | Conteúdo |
|---|---|
| [`ARCHITECTURE.pt-BR.md`](../architecture/ARCHITECTURE.pt-BR.md) | Como foi construído — arquitetura da solução, modelo de domínio, fluxos de eventos, RBAC, implantação |
| [`GETTING_STARTED.pt-BR.md`](../getting-started/GETTING_STARTED.pt-BR.md) | Pré-requisitos, subida do cluster, verificação, um passo a passo de demonstração |
| [`instructions.md`](../spec/instructions.md) | A especificação — arquitetura, responsabilidades de cada serviço, contratos de eventos, critérios de aceitação (em inglês) |
| [`notes.md`](../spec/notes.md) | Registro de decisões — 57 entradas, cada uma com a alternativa rejeitada e o que a reabriria (em inglês) |
| [`bdd.md`](../spec/bdd.md) | Cenários de aceitação em Gherkin — a camada de aceitação do projeto (em inglês) |
| [`TEST_COVERAGE.pt-BR.md`](../test-coverage/TEST_COVERAGE.pt-BR.md) | Cobertura de linhas medida, por serviço |
| [`frontend/design/`](https://github.com/tc2-fiap/frontend/tree/main/design) | Identidade visual — tokens de cor, marca, logomarca, favicon (aplicados literalmente no frontend) |
| [`base-project/docs/DOCUMENTATION.md`](https://github.com/KainanGuerra/fiap-games/blob/main/docs/DOCUMENTATION.md) | Como o monólito de referência que este sistema substitui foi construído |

## 2. Por que um sistema distribuído, e como foi construído

O `base-project` já aplicava DDD (contextos delimitados), Clean Architecture e BDD dentro de um único deployável. Este projeto mantém as três práticas, mas muda a *fronteira*: cada contexto delimitado — Usuários, Catálogo, Pedidos, Pagamentos, Notificações — passa a ser seu próprio repositório, schema e deployável, comunicando-se com os demais apenas por eventos (ou, no único caso em que a consistência síncrona realmente importa, por uma chamada HTTP simples).

Essa é uma superfície de falha materialmente maior que a de um monólito: cinco serviços, um message broker, bancos de dados por serviço e manifests Kubernetes, em vez de um único processo e um único banco. A ordem de construção foi escolhida especificamente para gerenciar esse risco — primeiro um **walking skeleton** (o caminho mais estreito possível por toda a infraestrutura — cluster, Helm, Ingress, Postgres, RabbitMQ, logging estruturado — comprovado apenas com `users-api` e `notifications-api`), depois amplitude (completando os serviços mais simples, provando a confiança JWT entre serviços), depois o núcleo real do projeto (o fluxo de compra), depois o endurecimento contra os critérios de aceitação, depois funcionalidades que surgiram mais tarde (RBAC de admin e a trilha de auditoria entre serviços), depois o frontend, nessa ordem. Integrar tudo apenas depois que cada serviço estivesse "completo" foi tratado como o principal risco a evitar — ver a tabela de riscos do `notes.md`.

## 3. Entregáveis

- Cinco serviços de backend implantáveis e testáveis de forma independente, e um frontend React, cada um com seu próprio Dockerfile, subchart Helm e `docker-compose.yml` independente.
- Um fluxo de compra orientado a eventos abrangendo três serviços com outbox transacional, verificado para os dois resultados (`Approved`/`Rejected`) e para reentrega idempotente.
- Isolamento de schema do Postgres por serviço, garantido por concessões de role e verificado ao vivo por uma consulta entre schemas recusada.
- Um papel de admin com trilha de auditoria entre serviços composta a partir de quatro endpoints independentes, e login com Google e envio real de e-mail opcionais, ambos degradando graciosamente para "desligado" quando não configurados.
- Subida do cluster em um único comando (`helm install`) verificada com zero reinícios a partir de um estado genuinamente limpo.
- Uma especificação escrita (`instructions.md`), um registro de decisões com 57 entradas (`notes.md`) e cenários de aceitação em Gherkin (`bdd.md`).
- Documentação narrativa bilíngue (inglês/português) — esta visão geral, `ARCHITECTURE.md`, `GETTING_STARTED.md`, `TEST_COVERAGE.md` e o `README.md` de cada repositório — e uma alternância de idioma inglês/pt-BR no próprio frontend; todo preço continua sendo um `decimal` em BRL de ponta a ponta, exibido em R$ ou convertido para um valor em USD só de exibição, dependendo da alternância (`notes.md` 35, 36, 39).
- Um passo de checkout real com resumo do produto, preço em duas moedas e um QR code/código copia-e-cola PIX quando um gateway real está ativo — além de uma trava contra compra duplicada, para que um jogo já possuído ou em andamento não possa ser comprado de novo (`notes.md` 40, 42).
- Uma segunda visão de admin, `/admin/events`, listando e filtrando todo evento/mensagem entre os cinco serviços — não restrita a um pedido — composta a partir de quatro endpoints de admin "listar tudo", da mesma forma que a trilha de auditoria por pedido (`notes.md` 43).
- Um carrinho em `localStorage` e uma página de confirmação `/checkout` pelas quais passam tanto o checkout do carrinho quanto o atalho `Comprar agora` do catálogo — nenhum caminho de compra pula a etapa de revisão — sustentados por `Order` ter se tornado um agregado real com múltiplos itens, com as regras de jogo duplicado no mesmo pedido e de posse duplicada agora reforçadas por índices únicos reais do Postgres em `order_items`, não apenas por verificações da aplicação (`notes.md` 51, 52).
- Status de pedido/pagamento ao vivo via Server-Sent Events (`GET /api/orders/{id}/stream`) em vez do cliente fazer polling a cada dois segundos (`notes.md` 53).
- A possibilidade de remover um jogo da biblioteca (liberando-o para recompra) com um modal de confirmação, sem nunca reverter o pedido `Paid` subjacente (`notes.md` 54).

## 4. Conclusão

As mesmas três práticas que moldaram o `base-project` — DDD, Clean Architecture, BDD — permaneceram inalteradas; o que mudou foi a unidade à qual foram aplicadas, de um módulo dentro de um único processo para cinco serviços implantáveis de forma independente, confiando uns nos outros apenas por meio de eventos e de um segredo JWT compartilhado. A ordem de construção em walking skeleton, o outbox transacional e o isolamento de schema do Postgres por serviço foram as três decisões que tornaram essa separação sustentável, e não apenas aspiracional — cada uma verificada ao vivo contra o sistema em execução, não apenas afirmada em um documento.
