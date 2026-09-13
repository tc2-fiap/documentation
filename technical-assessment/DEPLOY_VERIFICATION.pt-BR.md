[English](DEPLOY_VERIFICATION.en-US.md) · **Português**

# Verificação de Deploy

Como levar o código novo de um serviço até o cluster já rodando, e — a parte que realmente importa — como *provar* que ele chegou lá, em vez de simplesmente assumir que sim. Para a subida inicial do cluster em si, veja [`GETTING_STARTED.pt-BR.md`](../getting-started/GETTING_STARTED.pt-BR.md).

## Reimplantando um serviço depois de uma alteração de código

O cluster já está rodando — este é o ciclo para levar código novo até ele, não uma instalação do zero. Reconstrua a imagem daquele serviço, carregue-a no containerd do `kind` (mesma ideia do passo "Construir e carregar as imagens" do `GETTING_STARTED.md`, só que para uma imagem) e force o Deployment a de fato usá-la:

```bash
docker build -t frontend:latest frontend   # ou o comando de build de qualquer outro serviço
kind load docker-image frontend:latest --name fiap-games
kubectl rollout restart deployment/frontend -n fiap-games
kubectl rollout status deployment/frontend -n fiap-games
```

O passo `rollout restart` não é opcional. Todo chart define `imagePullPolicy: IfNotPresent` (`k8s/values.yaml`), então um pod já em execução nunca percebe que o `kind load docker-image` substituiu o que `<service>:latest` aponta no containerd — só um pod *novo* reavalia a tag, e só o `rollout restart` cria um. Pular esse passo deixa a build antiga rodando silenciosamente, sem nenhum erro em lugar nenhum.

Se todos os deployments do namespace estiverem escalados para zero (por exemplo, depois de um desligamento por ociosidade que manteve o release em vez de desinstalá-lo), o `rollout restart` não tem o que reiniciar — escale de volta primeiro, depois reinicie só o serviço que você reconstruiu:

```bash
kubectl scale deployment --all -n fiap-games --replicas=1
kubectl rollout restart deployment/frontend -n fiap-games
kubectl wait --namespace fiap-games --for=condition=ready pod --all --timeout=180s
```

## Verificando que uma reconstrução realmente chegou ao sistema em execução

Um `rollout status` dizendo "successfully rolled out" só prova que um pod novo iniciou — não que esse pod está rodando o código que você imagina. O `docker build` terminando limpo e o `kind load` imprimindo um ID novo também parecem prova, e pro frontend eles realmente são; para um serviço backend em .NET, não são — por motivos que vale entender, em vez de contornar às cegas.

### Frontend: comparar os nomes de arquivo com hash de conteúdo do Vite

O Vite nomeia o resultado do build com um hash do conteúdo do próprio arquivo — `dist/assets/index-<hash>.js`. Isso significa que comparar o nome de arquivo que seu build local acabou de gerar com o nome que o sistema em execução está de fato servindo é uma prova real, byte a byte, não um palpite:

```bash
# Logo depois do `npm run build`:
ls dist/assets/
# index-5Do0QEo6.js
# index-DZwwwKOy.css

# Depois do kind load + rollout restart, de qualquer lugar:
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

**2. O endpoint `/version`.** Os cinco serviços de backend (`users-api`, `catalog-api`, `orders-api`, `payments-api`, `notifications-api`) expõem `GET /version`, retornando o commit exato e o horário de build embutidos naquela imagem específica:

```json
{"sha": "5eb124839f1e4962b2e911e64a1241daad5df858", "buildTime": "2026-09-13T00:23:41Z"}
```

Isso é preenchido em tempo de build via dois `ARG`s do Dockerfile no estágio `runtime` (`ARG BUILD_SHA=unknown`, `ARG BUILD_TIME=unknown`, ambos também definidos como `ENV` para a aplicação em execução conseguir lê-los) — veja por exemplo `catalog-api/src/FiapGames.Catalog.Api/Dockerfile` e a linha `app.MapGet("/version", ...)` no `Program.cs` daquele serviço, logo depois de `app.MapHealthChecks("/health")`. O contexto de build de cada serviço é só a pasta do próprio projeto, sem `.git` dentro dela, então nenhum dos dois valores pode ser embutido automaticamente — passe os dois explicitamente:

```bash
SHA=$(git rev-parse HEAD)
TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)
docker build -t catalog-api:latest \
  --build-arg BUILD_SHA="$SHA" \
  --build-arg BUILD_TIME="$TIME" \
  catalog-api/src/FiapGames.Catalog.Api
```

Sem os dois `--build-arg`, o endpoint continua funcionando — só reporta `"unknown"` para o que foi omitido, em vez de falhar.

## Como alcançar `/health` e `/version` — e por que precisa de `kubectl port-forward`

Nenhum dos dois endpoints é alcançável pelo navegador via `http://localhost/...`, e isso é proposital, não um descuido: o Ingress (`orchestration/templates/ingress.yaml`) só roteia prefixos de path específicos para cada backend — `/api/users`, `/api/games`, `/api/quotations`, `/api/orders`, `/api/library`, `/api/payments`, `/api/notifications`, e `/` (o frontend). `/health` e `/version` ficam na raiz de cada serviço, fora de todos esses prefixos, então uma requisição a, por exemplo, `/api/catalog/health` não chega ao `catalog-api` de jeito nenhum — ela cai na regra `/` e acaba no roteador do próprio frontend, que então redireciona para algum lugar sensato por não reconhecer essa rota.

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
