[English](GETTING_STARTED.en-US.md) · **Português**

# Primeiros Passos

Suba todo o sistema distribuído em um cluster Kubernetes local, verifique se está saudável e percorra uma compra completa — incluindo a trilha de auditoria de admin — pela linha de comando.

Para entender a arquitetura, veja [`DOCUMENTATION.pt-BR.md`](DOCUMENTATION.pt-BR.md). Para entender por que ela tem essa forma, veja [`notes.md`](../spec/notes.md) (em inglês).

## Pré-requisitos

- **Docker** — o kind roda o cluster como um container.
- **kind** (Kubernetes in Docker) — o cluster local.
- **kubectl**
- **Helm** (v3)
- **jq** e **curl** — usados no passo a passo abaixo para manter tokens fora do seu histórico do shell de forma visível; não são exigidos pelo sistema em si.

Necessário apenas se você também quiser rodar um único serviço de forma independente, sem o cluster inteiro: **.NET 10 SDK**, **Node 22+**, **Docker Compose** (veja o próprio `README.md` daquele repositório).

## 1. Criar o cluster

```bash
kind create cluster --config repos/orchestration/kind/cluster-config.yaml
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

## 2. Instalar o sistema

```bash
cd repos/orchestration
helm dependency update
helm install fiap-games .
```

`helm dependency update` é obrigatório depois de clonar (ou depois de qualquer alteração em um subchart) — é o que efetivamente resolve as seis dependências `file://../*/k8s` do `Chart.yaml` em `charts/*.tgz` para o Helm instalar.

Isso sobe 8 pods: Postgres, RabbitMQ, os cinco serviços de backend e o frontend, todos no namespace `fiap-games`, todos conectados a um único Ingress em `http://localhost`.

## 3. Verificar

```bash
kubectl get pods -n fiap-games
```

Espere 8 pods, todos `Running`, todos com `RESTARTS` em `0`. Um reinício aqui quase sempre significa que um serviço iniciou antes do Postgres ou do RabbitMQ estarem prontos — todo serviço com banco carrega um init container `wait-for-postgres` justamente para evitar isso, então um reinício é um sinal real que vale investigar, não algo transitório para simplesmente tentar de novo.

```bash
kubectl wait --namespace fiap-games --for=condition=ready pod --all --timeout=180s
```

Isolamento de schema, confirmado diretamente contra o Postgres — isto deve ser **recusado**:

```bash
kubectl exec -n fiap-games deploy/postgres -- \
  psql -U orders -d fiap_games -c "SELECT * FROM users.\"Users\";"
# ERROR: permission denied for schema users
```

(A senha do role é `orders-dev-password`, conforme `values.yaml`; o `psql` dentro do pod usa o socket local, então não pede senha.)

## 4. Passo a passo de demonstração

Tudo abaixo passa pela única URL base do Ingress — sem port-forward, sem hostname por serviço.

```bash
BASE=http://localhost
```

### Cadastrar e fazer login

```bash
curl -s -X POST $BASE/api/users/register -H "Content-Type: application/json" -d '{
  "name": "Ada Lovelace",
  "email": "ada@example.com",
  "password": "correct-horse-battery-staple"
}' | jq

TOKEN=$(curl -s -X POST $BASE/api/users/login -H "Content-Type: application/json" -d '{
  "email": "ada@example.com",
  "password": "correct-horse-battery-staple"
}' | jq -r '.token')
```

Acompanhe `kubectl logs -n fiap-games deploy/notifications-api -f` em outro terminal — o cadastro deve produzir uma linha de log de e-mail de boas-vindas em um ou dois segundos (`UserCreatedEvent` indo e voltando pelo RabbitMQ).

### Navegar no catálogo e comprar um jogo

O `catalog-api` se auto-semeia com 8 jogos reais (capas reais do Steam, preços realistas em BRL) na primeira vez que inicia contra um banco vazio, então já existe algo para navegar sem criar nada manualmente:

```bash
curl -s $BASE/api/games -H "Authorization: Bearer $TOKEN" | jq
GAME_ID=$(curl -s $BASE/api/games -H "Authorization: Bearer $TOKEN" | jq -r '.items[0].id')

ORDER=$(curl -s -X POST $BASE/api/orders -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{\"gameId\": \"$GAME_ID\"}")
echo $ORDER | jq
ORDER_ID=$(echo $ORDER | jq -r '.id')
```

Note que o corpo da requisição carrega apenas `gameId` — o preço nunca é enviado pelo cliente, apenas lido do `catalog-api` e registrado como snapshot no pedido (`instructions.md` §6). Tente a mesma chamada `POST /api/orders` de novo com o mesmo `$GAME_ID` — agora ela retorna `409 Conflict` ("You already own this game or have a pending order for it"), já que um usuário não pode possuir o mesmo jogo duas vezes (`notes.md` 42); isso só libera de novo se aquele pedido se resolver como `Failed`.

### Observar o `Pending` virar `Paid`

```bash
watch -n1 "curl -s $BASE/api/orders/$ORDER_ID -H \"Authorization: Bearer $TOKEN\" | jq '.status'"
```

O gateway de pagamento simulado decide de forma determinística pelo preço, nunca aleatoriamente (`notes.md` 6) — um jogo com preço até `999.00`, que não termine em `.13`, é `Approved`; o pedido vira `Paid` dentro de `PAYMENT_PROCESSING_DELAY_SECONDS`. Uma vez `Paid`, ele aparece na biblioteca:

```bash
curl -s $BASE/api/library -H "Authorization: Bearer $TOKEN" | jq
```

Para ver uma compra rejeitada, compre um jogo com preço acima de `999.00`, ou cujo preço termine em `.13` — o pedido se resolve como `Failed` e nunca aparece na biblioteca.

### Cotação em USD e o seu próprio checkout

```bash
curl -s $BASE/api/quotations/usd-brl -H "Authorization: Bearer $TOKEN" | jq
```

Retorna a cotação USD→BRL atual (Frankfurter, com fallback para ExchangeRate-API se estiver fora do ar), cacheada no servidor por uma hora — chame duas vezes e a segunda resposta é quase instantânea. É isso que o frontend usa para mostrar um preço equivalente em USD quando o idioma da interface está em inglês; todo preço no backend continua sendo um `decimal` em BRL de qualquer forma (`notes.md` 39).

```bash
curl -s $BASE/api/payments/checkout/$ORDER_ID -H "Authorization: Bearer $TOKEN" | jq
```

Diferente da rota admin-only `/api/payments/{orderId}` abaixo, esta rota é para o próprio dono do pedido — ela retorna o status do pagamento, o gateway, o preço e (só quando um gateway PIX real gerou um) um QR code e o código copia-e-cola, nunca o payload bruto completo do gateway. Com o gateway padrão `simulated`, `pixCopyPasteCode`/`pixQrCodeBase64` são ambos `null` — não há nada para escanear, o pedido apenas se resolve sozinho (`notes.md` 40).

### Login de admin e a trilha de auditoria entre serviços

A conta de admin semeada (`admin.email`/`admin.password` do `values.yaml` — `admin@fiapgames.local` / `admin-dev-password-change-me` por padrão) pode ver os pedidos de todos os usuários e o ciclo de vida completo de qualquer um deles:

```bash
ADMIN_TOKEN=$(curl -s -X POST $BASE/api/users/login -H "Content-Type: application/json" -d '{
  "email": "admin@fiapgames.local",
  "password": "admin-dev-password-change-me"
}' | jq -r '.token')

curl -s $BASE/api/orders/admin -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s $BASE/api/orders/$ORDER_ID/events -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s $BASE/api/payments/$ORDER_ID -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s "$BASE/api/notifications?orderId=$ORDER_ID" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
```

As respostas de pagamentos e notificações incluem os payloads reais de request/response trocados com o gateway e o provedor de e-mail — JSON real, não um resumo, mesmo para o gateway simulado (entrada de trilha de auditoria do `notes.md`). Confirme que a fronteira se mantém — as mesmas quatro chamadas com `$TOKEN` (um usuário não-admin) em vez de `$ADMIN_TOKEN` devem retornar `403`.

As quatro chamadas acima são restritas a um pedido. Para navegar por todo evento/mensagem do sistema inteiro — cada `UserCreatedEvent` já publicado, cada evento do fluxo de compra, cada pagamento, cada notificação, não só de um pedido — use os endpoints "listar tudo" de todo o sistema (`notes.md` 43):

```bash
curl -s "$BASE/api/users/admin/events" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s "$BASE/api/orders/admin/events" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s "$BASE/api/payments/admin" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s "$BASE/api/notifications/admin" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
```

Cada um é paginado (`page`/`pageSize`, limitado a 100) e aceita filtros opcionais — um intervalo de datas `from`/`to` (UTC) nos quatro, além de `eventType` (users/orders), `status` (payments) e `type`/`status` (notifications). O `catalog-api` não tem um endpoint equivalente — ele não publica nem consome nada, então não há o que listar. A mesma verificação de fronteira `403` também se aplica aqui.

### O mesmo fluxo em um navegador

```bash
open http://localhost   # ou apenas navegue até lá
```

Cadastre-se ou faça login (um botão de login com Google aparece automaticamente só se `Google:ClientId` estiver configurado — ver `notes.md` 28), compre um jogo, e chegue na página de checkout: um item de linha do produto (capa, título, gênero/plataforma), o preço em BRL e em USD, e — se um gateway PIX real estiver configurado — um QR code para escanear. Ela consulta de `Pending` para `Paid`/`Failed` sem atualização manual, exatamente como o `$ORDER_ID` acima. Ao entrar como o admin semeado, aparecem dois links de navegação: a visão de todos os pedidos (e sua página de detalhe por pedido, que compõe os mesmos quatro endpoints restritos a um pedido acima em uma única tela) e "Eventos do Sistema" — a página `/admin/events` das chamadas curl acima, com filtros em dropdown por origem, tipo e categoria, mais um intervalo de datas, e um payload JSON bruto expansível em cada linha ao clicar.

O cabeçalho tem uma alternância EN/PT, visível mesmo antes de fazer login. Trocar para português sempre mostra o BRL nativo (ex.: `R$ 29,99`); trocar para inglês converte todo preço do catálogo para o equivalente em USD usando a cotação ao vivo, voltando para BRL se a cotação estiver indisponível — nunca um preço em branco ou quebrado. A página de checkout sempre mostra as duas moedas juntas, independente da alternância. A escolha de idioma em si persiste entre recarregamentos (`notes.md` 35, 36, 39).

## 5. Encerrar o ambiente

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
| `helm install` reclama de um chart archive faltando | Rode `helm dependency update` em `repos/orchestration/` primeiro — as dependências do chart guarda-chuva são caminhos locais `file://` que precisam ser resolvidos em `charts/*.tgz` |
| `curl $BASE/...` dá connection refused | O controlador de ingress ainda não está pronto, ou o cluster kind não foi criado com os mapeamentos de porta em `kind/cluster-config.yaml` |
| O botão do Google nunca aparece | Esperado quando não há `Google:ClientId` configurado — `GET /api/users/config` reporta `googleSignInEnabled: false` e o frontend o esconde deliberadamente, em vez de mostrar um botão fadado a falhar |
| Nenhum e-mail chega apesar de `EMAIL_PROVIDER=resend` | Verifique os logs do `notifications-api` e o Secret `resend-credentials` — uma `RESEND_API_KEY` ausente/inválida faz o envio falhar, e isso fica registrado na própria linha de `Notification` (visível via o endpoint admin de notificações), não é silenciosamente engolido |
| Preços do catálogo aparecem em BRL mesmo com a alternância em inglês | `GET /api/quotations/usd-brl` retornou `409` — tanto o Frankfurter quanto o ExchangeRate-API estão inacessíveis (geralmente um cluster sem acesso de saída à internet); o frontend degrada para o BRL nativo por design, em vez de mostrar um preço quebrado — veja os logs do `catalog-api` para saber qual provedor falhou e por quê |
| `POST /api/orders` retorna `409` para um jogo que você não acha que possui | Você (ou uma execução anterior deste passo a passo) já tem um pedido `Pending` ou `Paid` para aquele `GameId` — `GET /api/library` e `GET /api/orders/admin` (como admin) mostram todos os pedidos entre as tentativas; só um pedido `Failed` permite uma nova tentativa (`notes.md` 42) |
