**English** · [Português](DEPLOY_VERIFICATION.pt-BR.md)

# Deploy Verification

How to get one service's new code into the already-running cluster, and — the part that actually matters — how to *prove* it got there instead of assuming it did. For the initial cluster bring-up itself, see [`GETTING_STARTED.en-US.md`](../getting-started/GETTING_STARTED.en-US.md).

**The default flow needs none of this.** Every `k8s/values.yaml` points at `ghcr.io/tc2-fiap/<service>:latest` with `imagePullPolicy: Always`, and CI pushes a new image there on every commit to that repo's `main` — so a plain `kubectl rollout restart deployment/<service> -n fiap-games` (no rebuild, no `kind load`) is all a normal "pick up the latest pushed code" redeploy needs; the pod that comes up re-pulls `latest` from GHCR on its own. Everything below is for the case that new commit *isn't* on GHCR yet — testing an uncommitted local change before you push it.

## Testing a local, uncommitted change

Because every chart's default `imagePullPolicy` is now `Always` against a GHCR tag, `kind load docker-image` alone does nothing here anymore — kubelet contacts the registry on every pod (re)start regardless of what's sitting in containerd, so a locally-loaded image is silently ignored unless you also point that service's `image.*` values back at it. Build, load, **and** override in the same `helm upgrade`:

```bash
docker build -t frontend:latest frontend
# a backend service's build context is its own repo root, with -f pointing
# at the nested Dockerfile — e.g.:
docker build -t catalog-api:latest -f catalog-api/src/FiapGames.Catalog.Api/Dockerfile catalog-api

kind load docker-image frontend:latest catalog-api:latest --name fiap-games

cd orchestration
helm upgrade fiap-games . -n fiap-games \
  --set frontend.image.repository=frontend --set frontend.image.tag=latest --set frontend.image.pullPolicy=IfNotPresent \
  --set catalog-api.image.repository=catalog-api --set catalog-api.image.tag=latest --set catalog-api.image.pullPolicy=IfNotPresent
kubectl rollout status deployment/frontend deployment/catalog-api -n fiap-games
```

The three `--set` flags per service are what actually matter — they're what makes `IfNotPresent` (and therefore your `kind load`ed image) apply again instead of `Always` re-fetching GHCR's `latest` and ignoring it. Repeat the `--set` triplet for whichever services you're testing; a plain `helm upgrade fiap-games .` with no overrides (or `--reset-values`) snaps every service straight back to pulling GHCR's `latest`.

If every deployment in the namespace is scaled to zero (e.g. after an idle teardown that kept the release instead of uninstalling it), there's no pod for `rollout restart`/`helm upgrade` to replace — scale back up first:

```bash
kubectl scale deployment --all -n fiap-games --replicas=1
kubectl wait --namespace fiap-games --for=condition=ready pod --all --timeout=180s
```

## Testing every service's local changes at once

Sometimes a change genuinely touches all seven images — a shared-kernel file edited in every service's own duplicated copy (`notes.md` 21), or a cross-cutting change like adding a new consumed event to every service. The single-service loop above still applies, just looped, with every service's three `--set` overrides collected into one `helm upgrade`:

```bash
cd repos   # or wherever the seven service repos live, side by side

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

**Watch out for tag corruption in the loop.** Running the `for`/`case` construct above has, at least once, produced a mis-expanded tag — `orders-api:latest` silently becoming `orders-apiatest:latest`, a real image under a wrong name, not just a display glitch (caught via `docker images --format "{{.Repository}}:{{.Tag}}"`, since the corrupted tag's ID matched the build log's manifest hash exactly). The cause wasn't pinned down; building each service as its own standalone command (not inside the loop) reliably avoided it. If in doubt, verify tags with `docker images --format "{{.Repository}}:{{.Tag}}"` before `kind load`, and `docker rmi` anything mis-tagged.

**A values override is only enough for a pure code change.** If the change also touched anything under `*/k8s/templates/` (a new `securityContext` block, a new ConfigMap key, an env var), the `helm upgrade` above still re-renders every chart from its current templates (so template edits do reach the cluster) — but double-check the rendered output with `helm template` first if you're unsure the override and the template change interact the way you expect, since both are landing in the same `helm upgrade`.

**Reverting to GHCR.** Once you push the commit, drop every `--set` override (or run `helm upgrade fiap-games . --reset-values -n fiap-games`) — the chart's own defaults take back over and every service goes back to pulling `ghcr.io/tc2-fiap/<name>:latest`.

## Verifying a rebuild actually reached the running system

A `rollout status` that says "successfully rolled out" only proves a new pod started — not that the pod is running the code you think it is. `docker build` looking clean and `kind load` printing a new ID both feel like proof too, and for the frontend they actually are; for a .NET backend service, they aren't, for reasons worth understanding rather than working around blindly.

### Frontend: compare Vite's content-hashed filenames

Vite names its build output after a hash of the file's own content — `dist/assets/index-<hash>.js`. That means comparing the filename your local build just produced against the filename the running system is actually serving is a real, byte-for-byte proof, not a guess:

```bash
# Right after `npm run build`:
ls dist/assets/
# index-5Do0QEo6.js
# index-DZwwwKOy.css

# After kind load and the matching helm upgrade override (see above), from anywhere:
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

This is computed by the Dockerfile itself, not passed in from outside. Each backend's build context is its own **repo root** (not just the service subfolder, specifically so `.git` is present), and a `RUN` step in the `build` stage runs `git rev-parse HEAD` + `date -u` and writes the result to `/build-info.json`, copied into the `runtime` stage alongside the published app — see e.g. `catalog-api/src/FiapGames.Catalog.Api/Dockerfile`, `Shared/Infrastructure/BuildInfo.cs` (reads that file), and the `app.MapGet("/version", ...)` line in that service's `Program.cs`, right after `app.MapHealthChecks("/health")`:

```bash
docker build -t catalog-api:latest -f catalog-api/src/FiapGames.Catalog.Api/Dockerfile catalog-api
```

No `--build-arg` to pass — an earlier version of this mechanism used `ARG BUILD_SHA`/`ARG BUILD_TIME` passed from outside, which this project hit real trouble with in practice: easy to forget to pass at all, or to compute against a not-yet-committed `HEAD` right before committing (both happened). Computing it inside the image instead means it can only ever reflect whatever was actually `COPY`'d in, which — as long as you commit before you build — is always the code that's actually running. `BuildInfo.Read()` falls back to `"unknown"` for either field rather than failing if `/build-info.json` is missing or unreadable (e.g. a build context without `.git`, or a base image swap that dropped the file).

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
