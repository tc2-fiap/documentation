[English](DEPLOY_VERIFICATION.en-US.md) · **Português**

# Verificação de Deploy

Como levar o código novo de um serviço até o cluster já rodando, e — a parte que realmente importa — como *provar* que ele chegou lá, em vez de simplesmente assumir que sim. Para a subida inicial do cluster em si, veja [`GETTING_STARTED.pt-BR.md`](../getting-started/GETTING_STARTED.pt-BR.md).

**O fluxo padrão não precisa de nada disso.** Todo `k8s/values.yaml` aponta para `ghcr.io/tc2-fiap/<serviço>:latest` com `imagePullPolicy: Always`, e o CI publica uma imagem nova ali a cada commit na `main` daquele repositório — então um simples `kubectl rollout restart deployment/<serviço> -n fiap-games` (sem reconstruir nada, sem `kind load`) já basta para um redeploy normal de "pegar o código mais recente publicado"; o pod que sobe repuxa o `latest` do GHCR sozinho. Tudo abaixo é para o caso em que esse commit novo *ainda não* está no GHCR — testar uma alteração local antes de dar push, o que exige **Docker com o plugin de CLI `buildx`** (o Docker Desktop já vem com ele; uma instalação Linux só com o Engine geralmente não vem — `docker buildx version`, e `sudo apt install docker-buildx` no Debian/Ubuntu se aparecer `unknown command`) para que o `docker build` use o BuildKit em vez do builder antigo, já depreciado.

## Testando uma alteração local, ainda não commitada

Como o `imagePullPolicy` padrão de todo chart agora é `Always` contra uma tag do GHCR, o `kind load docker-image` sozinho não faz mais nada aqui — o kubelet contata o registry a cada (re)início de pod, independente do que esteja no containerd, então uma imagem carregada localmente é silenciosamente ignorada a menos que você também aponte o `image.*` daquele serviço de volta pra ela. Construa, carregue **e** sobrescreva no mesmo `helm upgrade`:

```bash
docker build -t frontend:latest frontend
# o contexto de build de um serviço backend é a raiz do próprio repositório,
# com -f apontando pro Dockerfile aninhado — por exemplo:
docker build -t catalog-api:latest -f catalog-api/src/FiapGames.Catalog.Api/Dockerfile catalog-api

kind load docker-image frontend:latest catalog-api:latest --name fiap-games

cd orchestration
helm upgrade fiap-games . -n fiap-games \
  --set frontend.image.repository=frontend --set frontend.image.tag=latest --set frontend.image.pullPolicy=IfNotPresent \
  --set catalog-api.image.repository=catalog-api --set catalog-api.image.tag=latest --set catalog-api.image.pullPolicy=IfNotPresent
kubectl rollout status deployment/frontend deployment/catalog-api -n fiap-games
```

As três flags `--set` por serviço são o que realmente importa — são elas que fazem o `IfNotPresent` (e portanto sua imagem carregada via `kind load`) valer de novo, em vez do `Always` repuxar o `latest` do GHCR e ignorá-la. Repita o trio de `--set` para cada serviço em teste; um `helm upgrade fiap-games .` puro, sem overrides (ou `--reset-values`), devolve todo mundo direto pro `latest` do GHCR.

Se todos os deployments do namespace estiverem escalados para zero (por exemplo, depois de um desligamento por ociosidade que manteve o release em vez de desinstalá-lo), não existe pod pro `rollout restart`/`helm upgrade` substituir — escale de volta primeiro:

```bash
kubectl scale deployment --all -n fiap-games --replicas=1
kubectl wait --namespace fiap-games --for=condition=ready pod --all --timeout=180s
```

## Testando as alterações locais de todos os serviços de uma vez

Às vezes uma mudança realmente afeta as sete imagens — um arquivo do kernel compartilhado editado na cópia duplicada de cada serviço (`notes.md` 21), ou uma mudança transversal como adicionar um novo evento consumido em todo mundo. O ciclo de um único serviço acima continua valendo, só que em loop, com os overrides de todos os serviços reunidos em um único `helm upgrade`:

```bash
cd repos   # ou onde quer que os sete repos de serviço estejam, lado a lado

for svc in users-api catalog-api orders-api payments-api notifications-api platform-api; do
  case "$svc" in
    users-api)         name="Users" ;;
    catalog-api)        name="Catalog" ;;
    orders-api)          name="Orders" ;;
    payments-api)        name="Payments" ;;
    notifications-api)   name="Notifications" ;;
    platform-api)        name="Platform" ;;
  esac
  docker build -t "$svc:latest" -f "$svc/src/FiapGames.$name.Api/Dockerfile" "$svc"
done

docker build -t frontend:latest frontend

kind load docker-image users-api:latest catalog-api:latest orders-api:latest \
  payments-api:latest notifications-api:latest platform-api:latest frontend:latest \
  --name fiap-games

cd orchestration
helm upgrade fiap-games . -n fiap-games \
  --set users-api.image.repository=users-api --set users-api.image.tag=latest --set users-api.image.pullPolicy=IfNotPresent \
  --set catalog-api.image.repository=catalog-api --set catalog-api.image.tag=latest --set catalog-api.image.pullPolicy=IfNotPresent \
  --set orders-api.image.repository=orders-api --set orders-api.image.tag=latest --set orders-api.image.pullPolicy=IfNotPresent \
  --set payments-api.image.repository=payments-api --set payments-api.image.tag=latest --set payments-api.image.pullPolicy=IfNotPresent \
  --set notifications-api.image.repository=notifications-api --set notifications-api.image.tag=latest --set notifications-api.image.pullPolicy=IfNotPresent \
  --set platform-api.image.repository=platform-api --set platform-api.image.tag=latest --set platform-api.image.pullPolicy=IfNotPresent \
  --set frontend.image.repository=frontend --set frontend.image.tag=latest --set frontend.image.pullPolicy=IfNotPresent

kubectl wait --namespace fiap-games --for=condition=ready pod --all --timeout=180s
```

**Cuidado com corrupção de tag dentro do loop.** Rodar esse `for`/`case` já produziu, pelo menos uma vez, uma tag mal expandida — `orders-api:latest` virando silenciosamente `orders-apiatest:latest`, uma imagem real com o nome errado, não só um glitch de exibição (detectado via `docker images --format "{{.Repository}}:{{.Tag}}"`, já que o ID da tag corrompida batia exatamente com o hash do manifest no log do build). A causa não foi identificada; construir cada serviço como um comando isolado (fora do loop) evitou o problema de forma confiável. Na dúvida, confira as tags com `docker images --format "{{.Repository}}:{{.Tag}}"` antes do `kind load`, e dê `docker rmi` em qualquer coisa com a tag errada.

**Um override de values só é suficiente para uma mudança puramente de código.** Se a mudança também tocou algo em `*/k8s/templates/` (um novo bloco `securityContext`, uma nova chave de ConfigMap, uma variável de ambiente), o `helm upgrade` acima ainda renderiza de novo cada chart a partir dos templates atuais (então as edições de template chegam ao cluster) — mas confira a saída renderizada com `helm template` primeiro se não tiver certeza de como o override e a mudança de template interagem, já que os dois estão indo juntos no mesmo `helm upgrade`.

**Voltando pro GHCR.** Depois de dar push no commit, remova todos os overrides `--set` (ou rode `helm upgrade fiap-games . --reset-values -n fiap-games`) — os valores padrão do chart assumem de novo e cada serviço volta a puxar `ghcr.io/tc2-fiap/<nome>:latest`.

## Verificando que uma reconstrução realmente chegou ao sistema em execução

Um `rollout status` dizendo "successfully rolled out" só prova que um pod novo iniciou — não que esse pod está rodando o código que você imagina. O `docker build` terminando limpo e o `kind load` imprimindo um ID novo também parecem prova, e pro frontend eles realmente são; para um serviço backend em .NET, não são — por motivos que vale entender, em vez de contornar às cegas.

### Frontend: comparar os nomes de arquivo com hash de conteúdo do Vite

O Vite nomeia o resultado do build com um hash do conteúdo do próprio arquivo — `dist/assets/index-<hash>.js`. Isso significa que comparar o nome de arquivo que seu build local acabou de gerar com o nome que o sistema em execução está de fato servindo é uma prova real, byte a byte, não um palpite:

```bash
# Logo depois do `npm run build`:
ls dist/assets/
# index-5Do0QEo6.js
# index-DZwwwKOy.css

# Depois do kind load e do helm upgrade com o override correspondente (veja acima), de qualquer lugar:
curl -s http://localhost/ | grep -oE '/assets/index-[^"]+\.(js|css)'
# /assets/index-5Do0QEo6.js
# /assets/index-DZwwwKOy.css
```

Se os nomes baterem, o conteúdo bate — o Vite teria escolhido um hash diferente para bytes diferentes. Como verificação ainda mais forte, dá pra fazer `grep` no bundle servido procurando por uma string literal ou objeto que você acabou de escrever (uma chave de tradução, uma constante numérica específica) para confirmar que não é só uma coincidência de nome de arquivo.

### Backend: o que parece prova mas não é

A ideia igualmente óbvia para um serviço backend é comparar o ID de imagem que o Docker calculou localmente com o ID de imagem que o Kubernetes reporta para o pod em execução:

```bash
docker image inspect catalog-api:latest --format '{{.Id}}'
kubectl get pod -n fiap-games -l app=catalog-api -o jsonpath='{.items[0].status.containerStatuses[0].imageID}'
```

Esses valores quase nunca vão bater, por dois motivos independentes, ambos confirmados ao vivo:

1. **O `kind load docker-image` muda o formato da imagem.** Ele funciona rodando `docker save` e importando o resultado no containerd via `ctr images import`, convertendo o manifesto do Docker para o formato containerd/OCI. As duas ferramentas calculam o "ID da imagem" sobre coisas diferentes, então o `.Id` do `docker image inspect` e o do containerd (o que `kubectl`/`crictl` reportam) nunca estão no mesmo espaço de digest — nem mesmo para a imagem exatamente igual.
2. **A atestação de proveniência do BuildKit torna cada build não determinístico**, independente do primeiro problema. Reconstruir o `catalog-api` duas vezes seguidas com *zero* mudanças de código-fonte — todas as camadas aparecendo como `CACHED` nas duas vezes — ainda assim produziu dois IDs de imagem finais diferentes, porque os metadados de atestação que o BuildKit anexa ao manifesto exportado embutem um timestamp de build. Um ID de imagem que mudou não é evidência de que seu código mudou.

Não use essa comparação para nada. Foi testada, parecia razoável, e não funciona.

### Backend: o que realmente funciona

**1. Ler o próprio log do `docker build`.** O cache de camadas do Docker é baseado no conteúdo real do que uma etapa `COPY` copia — diferente do digest do manifesto externo, essa parte não sofre com o ruído da atestação. Se a camada `COPY . .` e a camada `dotnet publish`/`build` aparecerem como `CACHED`, seu código-fonte é idêntico, byte a byte, ao último build (o que significa ou que sua edição não foi salva, ou que você já buildou exatamente esse conteúdo antes). Se elas executarem de verdade, sua mudança foi incorporada:

```
#9  [build 3/6] COPY FiapGames.Catalog.Api.csproj .
#9  CACHED
#10 [build 5/6] COPY . .
#10 CACHED          <- se isso aparecer CACHED logo depois de você editar código, algo está errado
#11 [build 6/6] RUN dotnet publish ...
#11 CACHED
```

**2. O endpoint `/version`.** Os seis serviços de backend (`users-api`, `catalog-api`, `orders-api`, `payments-api`, `notifications-api`, `platform-api`) expõem `GET /version`, retornando o commit exato e o horário de build embutidos naquela imagem específica:

```json
{"sha": "5eb124839f1e4962b2e911e64a1241daad5df858", "buildTime": "2026-09-13T00:23:41Z"}
```

Isso é calculado pelo próprio Dockerfile, não passado de fora. O contexto de build de cada backend é a **raiz do próprio repositório** (não só a subpasta do serviço, exatamente para que o `.git` esteja presente), e uma etapa `RUN` no estágio `build` roda `git rev-parse HEAD` + `date -u` e grava o resultado em `/build-info.json`, copiado para o estágio `runtime` junto com a aplicação publicada — veja por exemplo `catalog-api/src/FiapGames.Catalog.Api/Dockerfile`, `Shared/Infrastructure/BuildInfo.cs` (lê esse arquivo) e a linha `app.MapGet("/version", ...)` no `Program.cs` daquele serviço, logo depois de `app.MapHealthChecks("/health")`:

```bash
docker build -t catalog-api:latest -f catalog-api/src/FiapGames.Catalog.Api/Dockerfile catalog-api
```

Nenhum `--build-arg` pra passar — uma versão anterior desse mecanismo usava `ARG BUILD_SHA`/`ARG BUILD_TIME` passados de fora, e este projeto teve problema de verdade com isso na prática: fácil esquecer de passar, ou calcular contra um `HEAD` ainda não commitado bem antes de commitar (os dois já aconteceram). Calcular dentro da própria imagem significa que só pode refletir o que realmente foi copiado via `COPY` — o que, desde que você commite antes de buildar, é sempre o código que está de fato rodando. `BuildInfo.Read()` cai em `"unknown"` para qualquer um dos campos em vez de falhar se `/build-info.json` estiver ausente ou ilegível (ex.: um contexto de build sem `.git`, ou uma troca de imagem base que descartou o arquivo).

## Como alcançar `/health` e `/version` — e por que precisa de `kubectl port-forward`

Nenhum dos dois endpoints é alcançável pelo navegador via `http://localhost/...`, e isso é proposital, não um descuido: o Ingress (`orchestration/templates/ingress.yaml`) só roteia prefixos de path específicos para cada backend — `/api/users`, `/api/catalog`, `/api/quotations`, `/api/orders`, `/api/library`, `/api/payments`, `/api/notifications`, `/api/platform`, e `/` (o frontend). `/health` e `/version` ficam na raiz de cada serviço, fora de todos esses prefixos, então uma requisição a, por exemplo, `/api/catalog/health` não chega ao `catalog-api` de jeito nenhum — ela cai na regra `/` e acaba no roteador do próprio frontend, que então redireciona para algum lugar sensato por não reconhecer essa rota.

As próprias sondas de liveness/readiness do Kubernetes alcançam `/health` diretamente (IP do pod, sem passar pelo Ingress), então isso nunca foi um problema pra elas — só vira um problema no momento em que uma pessoa quer checar `/health` ou `/version` manualmente. O `kubectl port-forward` abre um túnel direto de uma porta local até o pod, contornando o Ingress inteiramente.

Primeiro, liste os nomes reais dos Services — o `svc/<nome>` abaixo tem que ser um desses, não um chute:

```bash
kubectl get svc -n fiap-games
# NAME                TYPE        CLUSTER-IP      PORT(S)
# catalog-api         ClusterIP   10.96.x.x       8080/TCP
# users-api           ClusterIP   10.96.x.x       8080/TCP
# orders-api          ClusterIP   10.96.x.x       8080/TCP
# payments-api        ClusterIP   10.96.x.x       8080/TCP
# notifications-api   ClusterIP   10.96.x.x       8080/TCP
# platform-api        ClusterIP   10.96.x.x       8080/TCP
# ...
```

Depois, encaminhe uma porta local até a porta desse Service:

```bash
kubectl port-forward -n fiap-games svc/catalog-api 18080:8080 &
curl -s http://localhost:18080/version; echo
curl -s http://localhost:18080/health; echo
```

Em `18080:8080`, só o segundo número é fixo — precisa ser a porta real do Service (`8080` em todo backend aqui, batendo com o `ASPNETCORE_URLS=http://+:8080` de cada container, confirmado pela coluna `PORT(S)` acima). O primeiro número, `18080`, é totalmente arbitrário: é só em qual porta da *sua própria máquina* o túnel vai escutar, escolhida aqui só pra não colidir com algo que já esteja usando `8080` localmente. Qualquer porta local livre serve — `19999:8080`, `8888:8080`, o que estiver livre.

Encerre o port-forward quando terminar (`kill %1`, ou `pkill -f "port-forward.*<serviço>"`) — ele não sai sozinho.
