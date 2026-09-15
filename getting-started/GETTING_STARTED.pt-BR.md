[English](GETTING_STARTED.en-US.md) · **Português**

# Primeiros Passos

Suba todo o sistema distribuído em um cluster Kubernetes local, verifique se está saudável e percorra uma compra completa — incluindo a trilha de auditoria de admin — pela linha de comando.

Para entender a arquitetura, veja [`ARCHITECTURE.pt-BR.md`](../architecture/ARCHITECTURE.pt-BR.md). Para entender por que ela tem essa forma, veja [`notes.md`](../spec/notes.md) (em inglês).

## Pré-requisitos

- **Docker** — o kind roda o cluster como um container.
- **kind** (Kubernetes in Docker) — o cluster local.
- **kubectl**
- **Helm** (v3)
- **jq** e **curl** — usados no passo a passo abaixo para manter tokens fora do seu histórico do shell de forma visível; não são exigidos pelo sistema em si.
- **Acesso de saída à internet para `ghcr.io`** — a imagem de cada serviço é puxada do GHCR no momento da instalação (veja o [passo 3](#3-as-imagens-são-puxadas-automaticamente)); o cluster já precisa desse mesmo tipo de acesso para chegar ao Docker Hub para `postgres`/`rabbitmq`, então isso não é uma nova categoria de requisito, só um novo host.
- **Portas 80 e 443 livres no host** — o passo 2 as mapeia diretamente para o ingress controller; se alguma já estiver ocupada (outro servidor local, um cluster `kind` anterior ainda de pé, etc.), o mapeamento fica silenciosamente sem efeito e tudo que passa por `http://localhost` falha sem erro claro. Verifique antes: `lsof -i :80` / `lsof -i :443` (ou `ss -ltn | grep -E ':80|:443'`) — ambos devem retornar vazio.

Necessário apenas se você também quiser rodar um único serviço de forma independente, sem o cluster inteiro: **.NET 10 SDK**, **Node 22+**, **Docker Compose** (veja o próprio `README.md` daquele repositório).

## 1. Clonar os repositórios

Este projeto está dividido em nove repositórios independentes no GitHub, sob [`github.com/tc2-fiap`](https://github.com/tc2-fiap) — veja [`../README.pt-BR.md`](../README.pt-BR.md) para a visão completa e o que cada um possui. Para rodar o sistema em si, só oito são necessários: os seis serviços de backend, o `frontend` e o `orchestration` (`documentation` — este repositório — e o `base-project`, o monólito de referência à parte, não fazem parte do sistema em execução).

O chart Helm do `orchestration` espera que os outros sete estejam como **diretórios irmãos** no disco — as dependências do seu `Chart.yaml` são caminhos relativos literais (`file://../users-api/k8s`, e assim por diante para cada serviço), não uma busca em um registro. Clone os oito em um diretório pai vazio, mantendo os nomes de pasta padrão que o `git clone` já usa:

```bash
mkdir fiap-games && cd fiap-games
for repo in users-api catalog-api orders-api payments-api notifications-api platform-api frontend orchestration; do
  git clone https://github.com/tc2-fiap/$repo.git
done
```

Todo comando a partir daqui roda a partir desse diretório pai (o que agora contém os oito como irmãos), a menos que um passo diga o contrário.

## 2. Criar o cluster

```bash
kind create cluster --config orchestration/kind/cluster-config.yaml
```

Isso cria um cluster de um nó chamado `fiap-games`, com as portas 80/443 do host mapeadas e o label de nó `ingress-ready` definido, para que um controlador de ingress possa se ligar diretamente a essas portas — sem necessidade de `kubectl port-forward` para nada acessado pelo Ingress.

Instale o nginx-ingress nele:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

Esse `kubectl wait` geralmente leva cerca de 30 segundos — a imagem do controlador precisa ser baixada e os jobs do admission webhook precisam terminar antes do pod reportar prontidão. Deixe rodar até o fim em vez de interromper — instalar o chart antes do controlador (e do Service do seu webhook) estar realmente pronto falha com `connection refused` ao chamar `ingress-nginx-controller-admission`.

## 3. As imagens são puxadas automaticamente

Nada para construir. O `k8s/values.yaml` de cada serviço já aponta para `ghcr.io/tc2-fiap/<nome>:latest` com `imagePullPolicy: Always` — o CI publica uma imagem nova ali a cada commit na `main` de cada repositório (job `docker-build-and-push` no próprio `.github/workflows/ci.yml` daquele repo). O `helm install` do próximo passo puxa as sete direto do GHCR, do mesmo jeito que já puxa as imagens de `postgres`/`rabbitmq` do Docker Hub — sem build local de Dockerfile, sem `kind load docker-image`.

Isso significa que o cluster precisa de acesso de saída à internet para `ghcr.io` (o mesmo tipo de acesso que já é necessário para chegar ao Docker Hub em busca de `postgres`/`rabbitmq`, não uma nova categoria de dependência) e, como a tag é flutuante e a política é `Always`, cada (re)início de pod busca de novo o que estiver atualmente na `main` daquele serviço — não existe aqui o atalho "já carregada, pode pular" que `postgres`/`rabbitmq` têm com `IfNotPresent`.

Um `helm install` do zero dispara essas sete buscas de uma vez, sem autenticação (os pacotes são públicos, não há credencial envolvida) — o endpoint de token para pull anônimo do GHCR já foi observado respondendo a uma rajada dessas com um `denied` transitório em uma ou duas imagens, indistinguível à primeira vista de um erro de permissão de verdade. Isso se resolve sozinho: o kubelet tenta de novo o pull com backoff, então um pod que aparece rapidamente como `ImagePullBackOff` logo depois da instalação e se recupera em um minuto ou dois não é um problema real — só trate como um se continuar falhando vários minutos depois (veja a tabela de Solução de problemas abaixo para esse caso).

Todo `GET /version` (backends) e a linha do frontend em `AdminSystemHealthPage` mostram o commit exato de onde aquela imagem puxada foi construída — cada Dockerfile calcula isso sozinho (`git rev-parse HEAD` no momento do build, gravado em `/build-info.json`) em vez de receber isso de fora, então o que aparece ali é sempre o que o GHCR publicou mais recentemente para a `main` daquele serviço.

## 4. Instalar o sistema

```bash
cd orchestration
helm dependency update
helm install fiap-games .
```

`helm dependency update` é obrigatório depois de clonar (ou depois de qualquer alteração em um subchart) — é o que efetivamente resolve as sete dependências `file://../*/k8s` do `Chart.yaml` em `charts/*.tgz` para o Helm instalar.

Isso sobe 9 pods: Postgres, RabbitMQ, os seis serviços de backend e o frontend, todos no namespace `fiap-games`, todos conectados a um único Ingress em `http://localhost`.

## 5. Verificar

```bash
kubectl get pods -n fiap-games
```

Espere 9 pods, todos `Running`, todos com `RESTARTS` em `0`. Um reinício aqui quase sempre significa que um serviço iniciou antes do Postgres ou do RabbitMQ estarem prontos — todo serviço com banco carrega um init container `wait-for-postgres` justamente para evitar isso, então um reinício é um sinal real que vale investigar, não algo transitório para simplesmente tentar de novo. O `platform-api` não tem banco de dados, então não tem esse init container nem esse modo de falha.

```bash
kubectl wait --namespace fiap-games --for=condition=ready pod --all --timeout=180s
```

Isolamento de schema, confirmado diretamente contra o Postgres — isto deve ser **recusado**:

```bash
kubectl exec -n fiap-games deploy/postgres -- \
  psql -U orders_role -d fiap_games -c "SELECT * FROM users.\"Users\";"
# ERROR: permission denied for schema users
```

(A senha do role é `orders-dev-password`, conforme `values.yaml`; o `psql` dentro do pod usa o socket local, então não pede senha.)

## 6. Passo a passo de demonstração

Tudo abaixo passa pela única URL base do Ingress — sem port-forward, sem hostname por serviço.

```bash
BASE=http://localhost
```

### Pelo navegador

```bash
open http://localhost   # ou apenas navegue até lá
```

1. Cadastre-se ou faça login com uma das contas semeadas — Admin, pra ver as telas de admin direto, ou Player, pra pular o cadastro. Um botão de login com Google aparece automaticamente só se `Google:ClientId` estiver configurado (`notes.md` 28).
2. Adicione um ou mais jogos ao carrinho a partir do catálogo e revise em `/cart`, ou use `Comprar agora` direto em um único jogo.
3. Confirme em `/checkout` — dos dois jeitos você chega na mesma página, antes de qualquer pedido ser de fato criado: uma revisão item a item (capa, título, gênero/plataforma de cada jogo) e o total em BRL e em USD.
4. Acompanhe a página do pedido: os mesmos itens, uma caixa de status do pedido e do pagamento, o total, e — se um gateway PIX real estiver configurado — um QR code para escanear. A transição `Pending` → `Paid`/`Failed` aparece no instante em que acontece, entregue por uma conexão Server-Sent Events em vez de consultada por polling (`notes.md` 53).
5. Logado como o admin semeado, explore os quatro links de navegação: a visão de todos os pedidos (com página de detalhe por pedido), "Eventos do Sistema" (`/admin/events` — filtros em dropdown por origem, tipo e categoria, intervalo de datas, payload JSON bruto expansível em cada linha), "Gerenciar Jogos" (`/admin/games` — tabela filtrável com Editar, Excluir e um botão "Criar jogo") e "Saúde do Sistema" (`/admin/system` — pods ao vivo e o `/version` de cada serviço).
6. Alterne o idioma no cabeçalho (EN/PT, visível mesmo antes de fazer login). Português sempre mostra o BRL nativo (ex.: `R$ 29,99`); inglês converte todo preço do catálogo para o equivalente em USD usando a cotação ao vivo, voltando para BRL se a cotação estiver indisponível — nunca um preço em branco ou quebrado. O carrinho, o checkout e a página do pedido sempre mostram as duas moedas juntas, independente da alternância. A escolha de idioma persiste entre recarregamentos (`notes.md` 35, 36, 39).

### Pelo terminal

O restante desta seção repete o mesmo fluxo direto pela API (`curl`), chamada a chamada — útil pra ver exatamente o que cada tela dispara por trás dos panos, e para o que a UI não expõe (a trilha de auditoria completa, a listagem de pods).

#### Cadastrar e fazer login

`POST /api/users/register` — e a página `/register` do frontend por trás dele — sempre cria uma conta com a role `Player`; não existe um jeito de se auto-cadastrar como `Admin`. A única forma de conseguir uma conta Admin é já ter uma e promover outra pessoa via `PUT /api/users/{id}/role` (admin-only) — por isso o `users-api` já semeia uma conta Admin (e, por conveniência, uma Player) na primeira subida:

| Conta | Email | Senha | Role |
|---|---|---|---|
| Admin | `admin@fiapgames.local` | `Admin-Dev-2026!` | `Admin` |
| Player | `player@fiapgames.local` | `Player-Dev-2026!` | `Player` |

(Configuradas nas chaves `admin`/`player` de `orchestration/values.yaml` — troque-as antes de qualquer implantação real.)

Qualquer senha usada em `POST /api/users/register` precisa ter pelo menos 12 caracteres e conter uma letra maiúscula, uma minúscula, um dígito e um caractere especial; uma pequena lista de senhas fracas comuns (`password123`, `qwerty123` etc.) é recusada mesmo que satisfaça essas regras (`notes.md` 83). O login não tem essa checagem — ele aceita o hash que já está gravado, incluindo qualquer conta pré-existente criada sob a regra antiga, mais curta.

O caminho mais rápido é pular o cadastro e logar direto com a conta Player já semeada:

```bash
TOKEN=$(curl -s -X POST $BASE/api/users/login -H "Content-Type: application/json" -d '{
  "email": "player@fiapgames.local",
  "password": "Player-Dev-2026!"
}' | jq -r '.accessToken')
```

Ou, para ver o próprio fluxo de cadastro (o e-mail de boas-vindas, `UserCreatedEvent` indo e voltando pelo RabbitMQ), registre uma conta nova e faça login com ela:

```bash
curl -s -X POST $BASE/api/users/register -H "Content-Type: application/json" -d '{
  "name": "Ada Lovelace",
  "email": "ada@example.com",
  "password": "CorrectHorse2026!Battery"
}' | jq

TOKEN=$(curl -s -X POST $BASE/api/users/login -H "Content-Type: application/json" -d '{
  "email": "ada@example.com",
  "password": "CorrectHorse2026!Battery"
}' | jq -r '.accessToken')
```

Acompanhe `kubectl logs -n fiap-games deploy/notifications-api -f` em outro terminal ao registrar — deve aparecer uma linha de log de e-mail de boas-vindas em um ou dois segundos (isso não acontece ao logar direto com a conta Player semeada, já que nenhum `UserCreatedEvent` novo é publicado nesse caso).

Daqui em diante, `$TOKEN` pode ser tanto o da conta Player semeada quanto o da conta recém-registrada — qualquer uma serve para o resto deste passo a passo.

#### Navegar no catálogo e comprar um jogo

O `catalog-api` se auto-semeia com 30 jogos (a maioria reais, com capas reais do Steam e preços realistas em BRL) na primeira vez que inicia contra um banco vazio, então já existe algo para navegar sem criar nada manualmente — inclusive um jogo fictício, "Corrupted Save: QA Edition", propositalmente precificado em `49.13` para sempre falhar no pagamento (veja a seção abaixo). No navegador isso passa por um carrinho e uma etapa de confirmação de checkout (veja [Pelo navegador](#pelo-navegador) acima); direto na API, uma única chamada cria o pedido:

```bash
curl -s $BASE/api/catalog -H "Authorization: Bearer $TOKEN" | jq
GAME_ID=$(curl -s $BASE/api/catalog -H "Authorization: Bearer $TOKEN" | jq -r '.items[0].id')

ORDER=$(curl -s -X POST $BASE/api/orders -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{\"gameIds\": [\"$GAME_ID\"]}")
echo $ORDER | jq
ORDER_ID=$(echo $ORDER | jq -r '.id')
```

Note que o corpo da requisição carrega apenas `gameIds` — uma lista, já que um único checkout pode gerar um pedido com vários jogos (um carrinho, no navegador) — e nunca um preço; o preço de cada item é lido do `catalog-api` e registrado como snapshot no pedido (`instructions.md` §6). Tente a mesma chamada `POST /api/orders` de novo com o mesmo `$GAME_ID` — agora ela retorna `409 Conflict` ("You already own or have a pending order for: `$GAME_ID`"), já que um usuário não pode possuir o mesmo jogo duas vezes. Isso é garantido de duas formas: uma checagem na aplicação, que gera essa mensagem amigável, e — a garantia de fato, à prova de corrida — um índice único parcial no próprio Postgres (`notes.md` 51, 52). De qualquer forma, isso só libera de novo se aquele pedido se resolver como `Failed`.

#### Observar o `Pending` virar `Paid`

```bash
watch -n1 "curl -s $BASE/api/orders/$ORDER_ID -H \"Authorization: Bearer $TOKEN\" | jq '.status'"
```

O gateway de pagamento simulado decide de forma determinística pelo preço, nunca aleatoriamente (`notes.md` 6) — um jogo com preço até `999.00`, que não termine em `.13`, é `Approved`; o pedido vira `Paid` dentro de `PAYMENT_PROCESSING_DELAY_SECONDS`. Uma vez `Paid`, ele aparece na biblioteca:

```bash
curl -s $BASE/api/library -H "Authorization: Bearer $TOKEN" | jq
```

Para ver uma compra rejeitada, compre um jogo com preço acima de `999.00`, ou cujo preço termine em `.13` — o pedido se resolve como `Failed` e nunca aparece na biblioteca. O seed já inclui um jogo pronto exatamente para isso: `"Corrupted Save: QA Edition"`, precificado em `49.13`.

#### Cotação em USD e o seu próprio checkout

```bash
curl -s $BASE/api/quotations/usd-brl -H "Authorization: Bearer $TOKEN" | jq
```

Retorna a cotação USD→BRL atual (Frankfurter, com fallback para ExchangeRate-API se estiver fora do ar), cacheada no servidor por uma hora — chame duas vezes e a segunda resposta é quase instantânea. É isso que o frontend usa para mostrar um preço equivalente em USD quando o idioma da interface está em inglês; todo preço no backend continua sendo um `decimal` em BRL de qualquer forma (`notes.md` 39).

```bash
curl -s $BASE/api/payments/checkout/$ORDER_ID -H "Authorization: Bearer $TOKEN" | jq
```

Diferente da rota admin-only `/api/payments/{orderId}` abaixo, esta rota é para o próprio dono do pedido — ela retorna o status do pagamento, o gateway, o preço e (só quando um gateway PIX real gerou um) um QR code e o código copia-e-cola, nunca o payload bruto completo do gateway. Com o gateway padrão `simulated`, `pixCopyPasteCode`/`pixQrCodeBase64` são ambos `null` — não há nada para escanear, o pedido apenas se resolve sozinho (`notes.md` 40).

#### Login de admin e a trilha de auditoria entre serviços

A conta Admin semeada (a mesma da tabela na seção "Cadastrar e fazer login" acima) pode ver os pedidos de todos os usuários e o ciclo de vida completo de qualquer um deles:

```bash
ADMIN_TOKEN=$(curl -s -X POST $BASE/api/users/login -H "Content-Type: application/json" -d '{
  "email": "admin@fiapgames.local",
  "password": "Admin-Dev-2026!"
}' | jq -r '.accessToken')

curl -s $BASE/api/orders/admin -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s $BASE/api/orders/$ORDER_ID/events -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s $BASE/api/payments/$ORDER_ID -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s "$BASE/api/notifications?orderId=$ORDER_ID" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
```

Um admin também pode listar todo Pod do cluster — o único endpoint apoiado em RBAC do Kubernetes em vez do Postgres (`notes.md` 75):

```bash
curl -s $BASE/api/platform/admin/pods -H "Authorization: Bearer $ADMIN_TOKEN" | jq
```

As respostas de pagamentos e notificações incluem os payloads reais de request/response trocados com o gateway e o provedor de e-mail — JSON real, não um resumo, mesmo para o gateway simulado (entrada de trilha de auditoria do `notes.md`). Confirme que a fronteira se mantém — as mesmas quatro chamadas com `$TOKEN` (um usuário não-admin) em vez de `$ADMIN_TOKEN` devem retornar `403`.

As quatro chamadas acima são restritas a um pedido. Para navegar por todo evento/mensagem do sistema inteiro — cada `UserCreatedEvent` já publicado, cada evento do fluxo de compra, cada pagamento, cada notificação, não só de um pedido — use os endpoints "listar tudo" de todo o sistema (`notes.md` 43):

```bash
curl -s "$BASE/api/users/admin/events" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s "$BASE/api/orders/admin/events" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s "$BASE/api/payments/admin" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s "$BASE/api/notifications/admin" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
```

Cada um é paginado (`page`/`pageSize`, limitado a 100) e aceita filtros opcionais — um intervalo de datas `from`/`to` (UTC) nos quatro, além de `eventType` (users/orders), `status` (payments) e `type`/`status` (notifications). O `catalog-api` não tem um endpoint equivalente — ele não tem nenhum desses quatro endpoints "listar tudo", então não há o que listar (ele consome o `TokenRevokedEvent`, sem relação com isso — `notes.md` 84). A mesma verificação de fronteira `403` também se aplica aqui.

## 7. Encerrar o ambiente

```bash
helm uninstall fiap-games
kubectl delete pvc -n fiap-games --all   # também apaga os dados do Postgres — só se você quiser uma próxima instalação verdadeiramente limpa
kind delete cluster --name fiap-games
```

## Rodando um serviço isolado

Todo repositório de backend e o frontend também rodam sozinhos via seu próprio `docker-compose.yml`, independente do cluster — útil para um ciclo de desenvolvimento rápido em um único serviço. Veja o próprio `README.md` daquele repositório para o comando exato; cada um sobe apenas aquele serviço mais o Postgres (e o RabbitMQ, se ele publica ou consome eventos) que ele sozinho precisa.

## Solução de problemas

| Sintoma | Causa provável |
|---|---|
| Um pod reinicia uma vez na instalação | Quase sempre é a prontidão do Postgres/RabbitMQ — verifique `kubectl logs` da instância anterior daquele pod (`kubectl logs -p`) antes de supor que é um bug de verdade; o init container `wait-for-postgres` deveria prevenir isso para o Postgres, mas o RabbitMQ não tem uma proteção equivalente (o MassTransit tenta novamente a própria conexão) |
| `helm install` reclama de um chart archive faltando | Rode `helm dependency update` em `orchestration/` primeiro — as dependências do chart guarda-chuva são caminhos locais `file://` que precisam ser resolvidos em `charts/*.tgz` |
| `helm dependency update` não consegue resolver uma dependência (`../users-api/k8s` não encontrado, etc.) | Os sete repositórios irmãos precisam estar clonados ao lado de `orchestration/`, com seus nomes de pasta padrão — veja o [passo 1](#1-clonar-os-repositórios) |
| `curl $BASE/...` dá connection refused | O controlador de ingress ainda não está pronto, ou o cluster kind não foi criado com os mapeamentos de porta em `kind/cluster-config.yaml` |
| Um pod fica brevemente em `ImagePullBackOff`/`ErrImagePull` para `ghcr.io/tc2-fiap/<service>:latest` (`denied`) logo depois do `helm install`, e se recupera sozinho em um ou dois minutos | O endpoint de token para pull anônimo do GHCR limita uma rajada de ~7 requisições simultâneas sem autenticação (confirmado ao vivo — isso é esperado, não uma falha real); o próprio kubelet tenta de novo com backoff e resolve sozinho, sem nenhuma ação sua |
| O mesmo erro `denied` persiste por vários minutos, ou o `kubectl get pods -w` nunca mostra a recuperação | Ou o pacote GHCR daquele repositório foi tornado privado de novo (GitHub → repositório → Packages → configurações daquele pacote → Change visibility → Public), ou o cluster não tem acesso de saída à internet para `ghcr.io` de forma alguma — verifique a mesma conectividade da qual os pulls de `postgres`/`rabbitmq` no Docker Hub já dependem |
| A página "Saúde do Sistema" (`/admin/system`) mostra `sha`/`buildTime` como `unknown` para algum serviço | A build mais recente do CI daquele serviço rodou sem o `.git` no contexto de build (não deveria acontecer com os Dockerfiles atuais, que constroem a partir da raiz do repositório) — verifique a execução do `docker-build-and-push` daquele repositório no CI em vez de qualquer coisa local, já que a imagem agora sempre vem do GHCR |
| O botão do Google nunca aparece | Esperado quando não há `Google:ClientId` configurado — `GET /api/users/config` reporta `googleSignInEnabled: false` e o frontend o esconde deliberadamente, em vez de mostrar um botão fadado a falhar |
| Nenhum e-mail chega apesar de `EMAIL_PROVIDER=resend` | Verifique os logs do `notifications-api` e o Secret `resend-credentials` — uma `RESEND_API_KEY` ausente/inválida faz o envio falhar, e isso fica registrado na própria linha de `Notification` (visível via o endpoint admin de notificações), não é silenciosamente engolido |
| Preços do catálogo aparecem em BRL mesmo com a alternância em inglês | `GET /api/quotations/usd-brl` retornou `409` — tanto o Frankfurter quanto o ExchangeRate-API estão inacessíveis (geralmente um cluster sem acesso de saída à internet); o frontend degrada para o BRL nativo por design, em vez de mostrar um preço quebrado — veja os logs do `catalog-api` para saber qual provedor falhou e por quê |
| `POST /api/orders` retorna `409` para um jogo que você não acha que possui | Você (ou uma execução anterior deste passo a passo) já tem um item de pedido `Pending` ou `Paid` para aquele jogo — `GET /api/library` e `GET /api/orders/admin` (como admin) mostram todos os pedidos entre as tentativas; só um pedido `Failed` permite uma nova tentativa (`notes.md` 42, 51, 52) |
