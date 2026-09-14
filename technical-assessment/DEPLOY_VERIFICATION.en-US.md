**English** · [Português](DEPLOY_VERIFICATION.pt-BR.md)

# Deploy Verification

How to get one service's new code into the already-running cluster, and — the part that actually matters — how to *prove* it got there instead of assuming it did. For the initial cluster bring-up itself, see [`GETTING_STARTED.en-US.md`](../getting-started/GETTING_STARTED.en-US.md).

## Redeploying a single service after a code change

The cluster is already running — this is the loop for picking up new code in it, not a fresh install. Rebuild that one service's image, load it into `kind`'s containerd (same idea as `GETTING_STARTED.md`'s "Build and load the images" step, just for one image), then force the Deployment to actually use it:

```bash
docker build -t frontend:latest frontend   # or any other service's own build command
kind load docker-image frontend:latest --name fiap-games
kubectl rollout restart deployment/frontend -n fiap-games
kubectl rollout status deployment/frontend -n fiap-games
```

The `rollout restart` step is not optional. Every chart sets `imagePullPolicy: IfNotPresent` (`k8s/values.yaml`), so an already-running pod never notices that `kind load docker-image` replaced what `<service>:latest` points to in containerd — only a *new* pod re-evaluates the tag, and only `rollout restart` creates one. Skipping it silently leaves the old build running with no error anywhere.

If every deployment in the namespace is scaled to zero (e.g. after an idle teardown that kept the release instead of uninstalling it), `rollout restart` has nothing to restart — scale back up first, then restart just the service you rebuilt:

```bash
kubectl scale deployment --all -n fiap-games --replicas=1
kubectl rollout restart deployment/frontend -n fiap-games
kubectl wait --namespace fiap-games --for=condition=ready pod --all --timeout=180s
```

## Rebuilding every service at once

Sometimes a change genuinely touches all seven images — a shared-kernel file edited in every service's own duplicated copy (`notes.md` 21), or a cross-cutting change like adding a new consumed event to every service. The single-service loop above still applies, just looped:

```bash
cd repos   # or wherever the seven service repos live, side by side

for svc in users-api catalog-api orders-api payments-api notifications-api platform-api; do
  cd "$svc"
  SHA=$(git rev-parse HEAD)
  TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)
  case "$svc" in
    users-api)         apidir="src/FiapGames.Users.Api" ;;
    catalog-api)        apidir="src/FiapGames.Catalog.Api" ;;
    orders-api)          apidir="src/FiapGames.Orders.Api" ;;
    payments-api)        apidir="src/FiapGames.Payments.Api" ;;
    notifications-api)   apidir="src/FiapGames.Notifications.Api" ;;
    platform-api)        apidir="src/FiapGames.Platform.Api" ;;
  esac
  docker build -t "$svc:latest" --build-arg BUILD_SHA="$SHA" --build-arg BUILD_TIME="$TIME" "$apidir"
  cd ..
done

cd frontend && docker build -t frontend:latest . && cd ..

kind load docker-image users-api:latest catalog-api:latest orders-api:latest \
  payments-api:latest notifications-api:latest platform-api:latest frontend:latest \
  --name fiap-games

kubectl rollout restart deployment/users-api deployment/catalog-api deployment/orders-api \
  deployment/payments-api deployment/notifications-api deployment/platform-api \
  deployment/frontend -n fiap-games

kubectl wait --namespace fiap-games --for=condition=ready pod --all --timeout=180s
```

**Watch out for tag corruption in the loop.** Running the `for`/`case` construct above has, at least once, produced a mis-expanded tag — `orders-api:latest` silently becoming `orders-apiatest:latest`, a real image under a wrong name, not just a display glitch (caught via `docker images --format "{{.Repository}}:{{.Tag}}"`, since the corrupted tag's ID matched the build log's manifest hash exactly). The cause wasn't pinned down; building each service as its own standalone command (not inside the loop) reliably avoided it. If in doubt, verify tags with `docker images --format "{{.Repository}}:{{.Tag}}"` before `kind load`, and `docker rmi` anything mis-tagged.

**`rollout restart` is only enough for a pure code change.** If the change also touched anything under `*/k8s/templates/` (a new `securityContext` block, a new ConfigMap key, an env var) `rollout restart` won't pick it up — that only forces a new pod on the *existing* Deployment spec, and the Deployment spec itself is what needs to change. Use `helm upgrade` instead, from `orchestration`:

```bash
cd orchestration
helm dependency update .
helm upgrade fiap-games . -n default
kubectl wait --namespace fiap-games --for=condition=ready pod --all --timeout=180s
```

This re-renders every chart (so template edits actually reach the cluster) and — because `imagePullPolicy: IfNotPresent` means an unchanged image never gets picked up on its own — a service whose *only* change was its image still needs the loop above run first, `helm upgrade` alone won't rebuild or reload anything. The two are independent: rebuild+reload gets new code into containerd, `helm upgrade`/`rollout restart` gets a new pod to actually run it.

## Verifying a rebuild actually reached the running system

A `rollout status` that says "successfully rolled out" only proves a new pod started — not that the pod is running the code you think it is. `docker build` looking clean and `kind load` printing a new ID both feel like proof too, and for the frontend they actually are; for a .NET backend service, they aren't, for reasons worth understanding rather than working around blindly.

### Frontend: compare Vite's content-hashed filenames

Vite names its build output after a hash of the file's own content — `dist/assets/index-<hash>.js`. That means comparing the filename your local build just produced against the filename the running system is actually serving is a real, byte-for-byte proof, not a guess:

```bash
# Right after `npm run build`:
ls dist/assets/
# index-5Do0QEo6.js
# index-DZwwwKOy.css

# After kind load + rollout restart, from anywhere:
curl -s http://localhost/ | grep -oE '/assets/index-[^"]+\.(js|css)'
# /assets/index-5Do0QEo6.js
# /assets/index-DZwwwKOy.css
```

If the names match, the content matches — Vite would have picked a different hash for different bytes. As an even stronger check, `grep` the served bundle for a literal string or object you just wrote (a translation key, a specific numeric constant) to confirm it's not just a coincidentally-matching filename.

### Backend: what looks like proof but isn't

The equally-obvious idea for a backend service is to compare the image ID Docker computed locally against the image ID Kubernetes reports for the running pod:

```bash
docker image inspect catalog-api:latest --format '{{.Id}}'
kubectl get pod -n fiap-games -l app=catalog-api -o jsonpath='{.items[0].status.containerStatuses[0].imageID}'
```

These will almost never match, for two unrelated reasons, both confirmed live:

1. **`kind load docker-image` changes the image's format.** It works by running `docker save` and importing the result into containerd via `ctr images import`, converting Docker's manifest into containerd/OCI's. The two tools compute their "image ID" over different things, so `docker image inspect`'s `.Id` and containerd's (what `kubectl`/`crictl` report) are never in the same digest space — not even for the exact same image.
2. **BuildKit's build-provenance attestation makes every build non-deterministic**, independent of the first problem. Rebuilding `catalog-api` twice in a row with *zero* source changes — every single layer logged `CACHED` both times — still produced two different final image IDs, because the attestation metadata BuildKit attaches to the exported manifest embeds a build timestamp. An image ID that changed is not evidence that your code changed.

Don't use this comparison for anything. It was tried, it looked reasonable, and it doesn't work.

### Backend: what actually works

**1. Read the `docker build` log itself.** Docker's layer cache is keyed on the actual content of what a `COPY` step copies — unlike the outer manifest digest, this part isn't affected by attestation noise. If the `COPY . .` layer and the `dotnet publish`/`build` layer both say `CACHED`, your source is byte-identical to the last build (which either means your edit didn't save, or you already built this exact content). If they execute for real, your change was picked up:

```
#9  [build 3/6] COPY FiapGames.Catalog.Api.csproj .
#9  CACHED
#10 [build 5/6] COPY . .
#10 CACHED          <- if this is CACHED right after you edited code, something is wrong
#11 [build 6/6] RUN dotnet publish ...
#11 CACHED
```

**2. The `/version` endpoint.** All six backend services (`users-api`, `catalog-api`, `orders-api`, `payments-api`, `notifications-api`, `platform-api`) expose `GET /version`, returning the exact commit and build time baked into that specific image:

```json
{"sha": "5eb124839f1e4962b2e911e64a1241daad5df858", "buildTime": "2026-09-13T00:23:41Z"}
```

This is populated at build time via two Dockerfile `ARG`s in the `runtime` stage (`ARG BUILD_SHA=unknown`, `ARG BUILD_TIME=unknown`, both set as `ENV` so the running app can read them) — see e.g. `catalog-api/src/FiapGames.Catalog.Api/Dockerfile` and the `app.MapGet("/version", ...)` line in that service's `Program.cs`, right after `app.MapHealthChecks("/health")`. The build context for each service is just that service's own project folder, with no `.git` in it, so neither value can be baked in automatically — pass both explicitly:

```bash
SHA=$(git rev-parse HEAD)
TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)
docker build -t catalog-api:latest \
  --build-arg BUILD_SHA="$SHA" \
  --build-arg BUILD_TIME="$TIME" \
  catalog-api/src/FiapGames.Catalog.Api
```

Without both `--build-arg`s, the endpoint still works — it just reports `"unknown"` for whichever one was omitted, rather than failing.

## Reaching `/health` and `/version` at all: why `kubectl port-forward`

Neither endpoint is reachable from the browser via `http://localhost/...`, and that's by design, not an oversight: the Ingress (`orchestration/templates/ingress.yaml`) only routes specific path prefixes to each backend — `/api/users`, `/api/catalog`, `/api/quotations`, `/api/orders`, `/api/library`, `/api/payments`, `/api/notifications`, `/api/platform`, and `/` (the frontend). `/health` and `/version` sit at each service's own root, outside every one of those prefixes, so a request to e.g. `/api/catalog/health` doesn't reach `catalog-api` at all — it falls through to the `/` rule and lands on the frontend's own router instead, which then redirects somewhere sensible for a route it doesn't recognize.

Kubernetes' own liveness/readiness probes reach `/health` directly (pod IP, no Ingress involved), so this was never a problem for them — it only becomes one the moment a person wants to check `/health` or `/version` by hand. `kubectl port-forward` opens a direct tunnel from a local port straight to the pod, bypassing the Ingress entirely.

First, list the actual Service names — `svc/<name>` below has to be one of these, not a guess:

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

Then forward a local port to that Service's port:

```bash
kubectl port-forward -n fiap-games svc/catalog-api 18080:8080 &
curl -s http://localhost:18080/version; echo
curl -s http://localhost:18080/health; echo
```

In `18080:8080`, only the second number is fixed — that has to be the Service's actual port (`8080` for every backend here, matching each container's own `ASPNETCORE_URLS=http://+:8080`, confirmed by the `PORT(S)` column above). The first number, `18080`, is completely arbitrary: it's just which port on *your own machine* the tunnel listens on, picked here only to avoid clashing with anything already using `8080` locally. Any free local port works — `19999:8080`, `8888:8080`, whatever's free.

Kill the port-forward once done (`kill %1`, or `pkill -f "port-forward.*<service>"`) — it doesn't exit on its own.
