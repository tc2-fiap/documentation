**English** · [Português](GETTING_STARTED.pt-BR.md)

# Getting Started

Bring the whole distributed system up on a local Kubernetes cluster, verify it's healthy, and walk through a full purchase — including the admin audit trail — from the command line.

For what you're looking at architecturally, see [`ARCHITECTURE.en-US.md`](../architecture/ARCHITECTURE.en-US.md). For why it's shaped this way, see [`notes.md`](../spec/notes.md).

## Prerequisites

- **Docker** — kind runs the cluster as a container.
- **kind** (Kubernetes in Docker) — the local cluster.
- **kubectl**
- **Helm** (v3)
- **jq** and **curl** — used in the walkthrough below to keep tokens out of your shell history in plain sight; not required by the system itself.
- **Outbound internet access to `ghcr.io`** — every service's image is pulled from GHCR at install time (see [step 3](#3-images-are-pulled-automatically)); the cluster already needs the same kind of access to reach Docker Hub for `postgres`/`rabbitmq`, so this isn't a new class of requirement, just a new host.
- **Host ports 80 and 443 free** — step 2 maps them straight through to the ingress controller; if either is already bound (another local server, a leftover `kind` cluster, etc.), the mapping is silently unpublished and everything reached through `http://localhost` fails with no clear error. Check first: `lsof -i :80` / `lsof -i :443` (or `ss -ltn | grep -E ':80|:443'`) — both should print nothing.

Only needed if you also want to run a single service standalone instead of the whole cluster: **.NET 10 SDK**, **Node 22+**, **Docker Compose** (see that repo's own `README.md`).

## 1. Clone the repos

This project is split across nine independent GitHub repos under [`github.com/tc2-fiap`](https://github.com/tc2-fiap) — see [`../README.en-US.md`](../README.en-US.md) for the full picture and what each one owns. Running the system itself only needs eight: the six backend services, `frontend`, and `orchestration` (`documentation` — this repo — and the separate `base-project` reference monolith aren't part of the running system).

`orchestration`'s Helm chart expects the other seven as **sibling directories** on disk — its `Chart.yaml` dependencies are literal relative paths (`file://../users-api/k8s`, and so on for each service), not a registry lookup. Clone all eight into one empty parent directory, keeping the default folder names `git clone` gives you:

```bash
mkdir fiap-games && cd fiap-games
for repo in users-api catalog-api orders-api payments-api notifications-api platform-api frontend orchestration; do
  git clone https://github.com/tc2-fiap/$repo.git
done
```

Every command from here on runs from this parent directory (the one now containing all eight as siblings), unless a step says otherwise.

## 2. Create the cluster

```bash
kind create cluster --config orchestration/kind/cluster-config.yaml
```

This creates a one-node cluster named `fiap-games` with host ports 80/443 mapped in and the `ingress-ready` node label set, so an ingress controller can bind those ports directly — no `kubectl port-forward` needed for anything reached through the Ingress. It also disables `kind`'s default CNI (`cluster-config.yaml`'s `networking.disableDefaultCNI`) — the cluster has no pod networking at all yet, and the node reports `NotReady`, until the next step installs one that actually enforces the `NetworkPolicy` resources this system ships (`kind`'s default CNI doesn't enforce them at all). If you're not sure what a CNI even is or why that matters, see [`discovers/DISCOVERIES.en-US.md`](../discovers/DISCOVERIES.en-US.md) entry 16 — explained from scratch there, not repeated here.

Install Calico — the node stays `NotReady` and nothing else can schedule until this is done:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.2/manifests/v1_crd_projectcalico_org.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.2/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.2/manifests/custom-resources.yaml
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n calico-system --timeout=180s
```

`custom-resources.yaml`'s pod CIDR (`192.168.0.0/16`) has to match `cluster-config.yaml`'s `networking.podSubnet` — both already do, since this repo ships them together; only relevant if you ever edit one without the other. Verify the node actually came up before moving on:

```bash
kubectl get nodes
```

Expect `Ready`, not `NotReady` — if it's still `NotReady` after the `wait` above returned, something about the CNI install didn't take.

Install nginx-ingress into it:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

The `kubectl wait` typically takes around 30 seconds — the controller image needs to pull and its admission-webhook jobs need to complete before the pod reports ready. Let it run to completion rather than interrupting it; installing the chart before the controller (and its webhook Service) is actually ready fails with a `connection refused` calling `ingress-nginx-controller-admission`.

## 3. Images are pulled automatically

Nothing to build. Every service's `k8s/values.yaml` already points at `ghcr.io/tc2-fiap/<name>:latest` with `imagePullPolicy: Always` — CI pushes a fresh image there on every commit to each repo's `main` (`docker-build-and-push` job in that repo's own `.github/workflows/ci.yml`). `helm install` in the next step pulls all seven straight from GHCR, the same way it already pulls the `postgres`/`rabbitmq` images from Docker Hub — no local Dockerfile build, no `kind load docker-image`.

This does mean the cluster needs outbound internet access to `ghcr.io` (same requirement as reaching Docker Hub for `postgres`/`rabbitmq`, not a new category of dependency) and, because the tag floats and the policy is `Always`, every pod (re)start re-pulls whatever is currently on `main` for that service — there's no "already loaded, skip it" fast path here the way there is for Postgres/RabbitMQ's `IfNotPresent` images.

A fresh `helm install` fires all seven of these pulls at once, unauthenticated (the packages are public, so no credential is involved) — GHCR's anonymous-pull token endpoint has been observed to answer a burst like that with a transient `denied` on one or two images, indistinguishable at a glance from a real permission error. It clears on its own: kubelet retries a failed pull with backoff, so a pod that briefly shows `ImagePullBackOff` right after install and then recovers within a minute or so isn't a real problem — only treat it as one if it's still failing several minutes later (see the Troubleshooting table below for that case).

Every `GET /version` endpoint (backends) and `AdminSystemHealthPage`'s frontend row report the exact commit that pulled image was built from — each Dockerfile computes this itself (`git rev-parse HEAD` at build time, baked into `/build-info.json`) rather than it being passed in, so what you see there is always whatever GHCR most recently published for that service's `main`.

## 4. Install the system

```bash
cd orchestration
helm dependency update
helm install fiap-games .
```

`helm dependency update` is required after cloning (or after any subchart change) — it's what actually resolves the seven `file://../*/k8s` dependencies in `Chart.yaml` into `charts/*.tgz` for Helm to install.

This brings up 9 pods: Postgres, RabbitMQ, and the six backend services plus the frontend, all in the `fiap-games` namespace, all wired to one Ingress at `http://localhost`.

## 5. Verify

```bash
kubectl get pods -n fiap-games
```

Expect 9 pods, all `Running`, all `RESTARTS` at `0`. A restart here almost always means a service started before Postgres or RabbitMQ was ready — every DB-backed service carries a `wait-for-postgres` init container specifically to prevent that, so a restart is a real signal worth investigating, not a transient to retry past. `platform-api` has no database, so it has no such init container and no such failure mode.

```bash
kubectl wait --namespace fiap-games --for=condition=ready pod --all --timeout=180s
```

Schema isolation, confirmed directly against Postgres — this should be **refused**:

```bash
kubectl exec -n fiap-games deploy/postgres -- \
  psql -U orders_role -d fiap_games -c "SELECT * FROM users.\"Users\";"
# ERROR: permission denied for schema users
```

(Role password is `orders-dev-password` per `values.yaml`; `psql` inside the pod uses the local socket, so no password prompt.)

## 6. Demo walkthrough

Everything below goes through the one Ingress base URL — no port-forwarding, no per-service hostnames.

```bash
BASE=http://localhost
```

### In a browser

```bash
open http://localhost   # or just navigate there
```

1. Register or log in with one of the seeded accounts — Admin, to see the admin screens right away, or Player, to skip registration. A Google sign-in button appears automatically only if `Google:ClientId` is configured (`notes.md` 28).
2. Add one or more games to your cart from the catalog and review them on `/cart`, or use `Buy Now` directly on a single game.
3. Confirm on `/checkout` — either way you land on the same page, before anything is actually ordered: an itemized review (cover image, title, genre/platform per game) and the total in both BRL and USD.
4. Watch the order page: the same line items, an order-and-payment-status box, the total, and — if a real PIX gateway is configured — a QR code to scan. The `Pending` → `Paid`/`Failed` transition appears the moment it happens, pushed over a Server-Sent Events connection rather than polled (`notes.md` 53).
5. Logged in as the seeded admin, explore the four nav links: the all-orders view (with a per-order detail page), "System Events" (`/admin/events` — dropdown filters for source, kind, and type plus a date range, a click-to-expand raw JSON payload on every row), "Manage Games" (`/admin/games` — a filterable table with Edit, Delete, and a "Create game" button), and "System Health" (`/admin/system` — a live pod table and every service's `/version`).
6. Toggle the language in the header (EN/PT, visible even before logging in). Portuguese always shows native BRL (e.g. `R$ 29,99`); English converts every catalog price to its USD equivalent using the live quotation, falling back to BRL if the rate lookup is ever unavailable — never a blank or broken price. The cart, checkout, and order pages always show both currencies together regardless of the toggle. The language choice persists across a reload (`notes.md` 35, 36, 39).

### In the terminal

The rest of this section repeats the same flow directly against the API (`curl`), call by call — useful for seeing exactly what each screen fires under the hood, and for what the UI doesn't expose (the full audit trail, the pod listing).

#### Register and log in

`POST /api/users/register` — and the frontend's `/register` page behind it — always creates a `Player` account; there is no way to self-register as `Admin`. The only way to get an Admin account is to already have one and promote someone else via `PUT /api/users/{id}/role` (admin-only) — which is why `users-api` already seeds one Admin (and, for convenience, one Player) account on first startup:

| Account | Email | Password | Role |
|---|---|---|---|
| Admin | `admin@fiapgames.local` | `Admin-Dev-2026!` | `Admin` |
| Player | `player@fiapgames.local` | `Player-Dev-2026!` | `Player` |

(Configured via `orchestration/values.yaml`'s `admin`/`player` keys — change them before any real deployment.)

Any password used in `POST /api/users/register` must be at least 12 characters long and contain an uppercase letter, a lowercase letter, a digit, and a special character; a short list of common weak passwords (`password123`, `qwerty123`, etc.) is rejected outright even if it happens to satisfy those rules (`notes.md` 83). Login has no such check — it accepts whatever hash is already on file, including any pre-existing account created under the old, shorter rule.

The fastest path is to skip registration entirely and log in directly with the seeded Player account:

```bash
TOKEN=$(curl -s -X POST $BASE/api/users/login -H "Content-Type: application/json" -d '{
  "email": "player@fiapgames.local",
  "password": "Player-Dev-2026!"
}' | jq -r '.accessToken')
```

Or, to see the registration flow itself (the welcome email, `UserCreatedEvent` round-tripping through RabbitMQ), register a fresh account and log in with it instead:

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

Watch `kubectl logs -n fiap-games deploy/notifications-api -f` in another terminal while registering — a welcome-email log line should appear within a second or two (this doesn't happen when logging straight into the seeded Player account, since no new `UserCreatedEvent` is published that way).

From here on, `$TOKEN` can be either the seeded Player's or the freshly-registered account's — either works for the rest of this walkthrough.

#### Browse the catalog and buy a game

`catalog-api` seeds itself with 30 games (mostly real ones, with real Steam cover art and realistic BRL prices) the first time it starts against an empty database, so there's already something to browse without creating anything by hand — including one fictional one, "Corrupted Save: QA Edition," deliberately priced to always fail at payment (see below). In the browser this goes through a cart and a checkout confirmation step (see [In a browser](#in-a-browser) above); against the API directly, one call places the order:

```bash
curl -s $BASE/api/catalog -H "Authorization: Bearer $TOKEN" | jq
GAME_ID=$(curl -s $BASE/api/catalog -H "Authorization: Bearer $TOKEN" | jq -r '.items[0].id')

ORDER=$(curl -s -X POST $BASE/api/orders -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{\"gameIds\": [\"$GAME_ID\"]}")
echo $ORDER | jq
ORDER_ID=$(echo $ORDER | jq -r '.id')
```

Note the request body carries only `gameIds` — a list, since a single checkout can place one order for several games (a cart, in the browser) — and never a price; each item's price is read from `catalog-api` and snapshotted onto the order (`instructions.md` §6). Try the same `POST /api/orders` call again with the same `$GAME_ID` — it now returns `409 Conflict` ("You already own or have a pending order for: `$GAME_ID`"), since a user can't own the same game twice. That's enforced two ways: an application-level check for a friendly error message, and — the actual, race-proof guarantee — a partial unique index on Postgres itself (`notes.md` 51, 52). Either way it only unblocks again if that order later settles `Failed`.

#### Watch `Pending` become `Paid`

```bash
watch -n1 "curl -s $BASE/api/orders/$ORDER_ID -H \"Authorization: Bearer $TOKEN\" | jq '.status'"
```

The simulated payment gateway decides deterministically by price, not randomly (`notes.md` 6) — a game priced at `999.00` or under, not ending in `.13`, is `Approved`; the order flips to `Paid` within `PAYMENT_PROCESSING_DELAY_SECONDS`. Once `Paid`, it appears in the library:

```bash
curl -s $BASE/api/library -H "Authorization: Bearer $TOKEN" | jq
```

To see a rejected purchase instead, buy a game priced above `999.00`, or one whose price ends in `.13` — the order settles to `Failed` and never appears in the library. The seed already includes a game built exactly for this: `"Corrupted Save: QA Edition"`, priced at `49.13`.

#### USD quotation and your own checkout

```bash
curl -s $BASE/api/quotations/usd-brl -H "Authorization: Bearer $TOKEN" | jq
```

Returns the current USD→BRL rate (Frankfurter, falling back to ExchangeRate-API if it's down), cached server-side for an hour — call it twice and the second response is near-instant. This is what the frontend uses to show a USD-equivalent price when the language toggle is set to English; every backend price stays a plain BRL `decimal` regardless (`notes.md` 39).

```bash
curl -s $BASE/api/payments/checkout/$ORDER_ID -H "Authorization: Bearer $TOKEN" | jq
```

Unlike the admin-only `/api/payments/{orderId}` below, this route is for the order's own owner — it returns the payment's status, gateway, price, and (only when a real PIX gateway produced one) a QR code and copy-paste code, never the full raw gateway payload. With the default `simulated` gateway, `pixCopyPasteCode`/`pixQrCodeBase64` are both `null` — there's nothing to scan, the order just settles on its own (`notes.md` 40).

#### Admin login and the cross-service audit trail

The seeded Admin account (the same one from the table in "Register and log in" above) can see every user's orders and the full lifecycle of any one of them:

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

An admin can also list every Pod in the cluster — the one endpoint backed by Kubernetes RBAC instead of Postgres (`notes.md` 75):

```bash
curl -s $BASE/api/platform/admin/pods -H "Authorization: Bearer $ADMIN_TOKEN" | jq
```

The payments and notifications responses include the actual request/response payloads exchanged with the gateway and email provider — real JSON, not a summary, even for the simulated gateway (`notes.md`'s audit-trail entry). Confirm the boundary holds — the same four calls with `$TOKEN` (a non-admin) instead of `$ADMIN_TOKEN` should all return `403`.

The four calls above are scoped to one order. To browse every event/message across the whole system instead — every `UserCreatedEvent` ever published, every purchase-flow event, every payment, every notification, not just one order's — use the system-wide "list all" endpoints (`notes.md` 43):

```bash
curl -s "$BASE/api/users/admin/events" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s "$BASE/api/orders/admin/events" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s "$BASE/api/payments/admin" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s "$BASE/api/notifications/admin" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
```

Each is paginated (`page`/`pageSize`, capped at 100) and accepts optional filters — a `from`/`to` UTC date range on all four, plus `eventType` (users/orders), `status` (payments), and `type`/`status` (notifications). `catalog-api` has no equivalent endpoint — it has none of these four "list everything" admin endpoints, so there's nothing to list (it does consume `TokenRevokedEvent`, unrelated to this — `notes.md` 84). Same `403` boundary check applies here too.

## 7. Tear down

```bash
helm uninstall fiap-games
kubectl delete pvc -n fiap-games --all   # drops Postgres data too — only if you want a truly clean next install
kind delete cluster --name fiap-games
```

## Running one service standalone

Every backend repo and the frontend also run alone via their own `docker-compose.yml`, independent of the cluster — useful for a fast inner dev loop on a single service. See that repo's own `README.md` for the exact command; each brings up just that service plus the Postgres (and RabbitMQ, if it publishes or consumes events) it alone needs.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| A pod restarts once at install | Almost always Postgres/RabbitMQ readiness — check `kubectl logs` for that pod's previous instance (`kubectl logs -p`) before assuming it's a real bug; the `wait-for-postgres` init container should prevent this for Postgres, but RabbitMQ has no equivalent guard (MassTransit retries its own connection) |
| `helm install` complains about a missing chart archive | Run `helm dependency update` in `orchestration/` first — the umbrella chart's dependencies are local `file://` paths that need resolving into `charts/*.tgz` |
| `helm dependency update` can't resolve a dependency (`../users-api/k8s` not found, etc.) | The seven sibling repos need to be cloned next to `orchestration/`, with their default folder names — see [step 1](#1-clone-the-repos) |
| `curl $BASE/...` connection refused | The ingress controller isn't ready yet, or the kind cluster wasn't created with the port mappings in `kind/cluster-config.yaml` |
| A pod is briefly `ImagePullBackOff`/`ErrImagePull` for `ghcr.io/tc2-fiap/<service>:latest` (`denied`) right after `helm install`, then recovers on its own within a minute or two | GHCR's anonymous-pull token endpoint rate-limits a burst of ~7 simultaneous unauthenticated requests (confirmed live — this is expected, not a real failure); kubelet's own retry-with-backoff clears it without any action from you |
| The same `denied` error persists for several minutes, or `kubectl get pods -w` never shows it recovering | Either the GHCR package for that repo was made private again (GitHub → repo → Packages → that package's settings → Change visibility → Public), or the cluster has no outbound internet access to `ghcr.io` at all — check the same connectivity `postgres`/`rabbitmq`'s Docker Hub pulls already depend on |
| The "System Health" page (`/admin/system`) shows `sha`/`buildTime` as `unknown` for some service | That service's most recent CI build ran without `.git` in its Docker context (shouldn't happen with the current Dockerfiles, which build from repo root) — check that repo's `docker-build-and-push` CI run rather than anything local, since the image now always comes from GHCR |
| Google button never appears | Expected with no `Google:ClientId` configured — `GET /api/users/config` reports `googleSignInEnabled: false` and the frontend hides it deliberately, rather than showing a button guaranteed to fail |
| No email arrives despite `EMAIL_PROVIDER=resend` | Check `notifications-api` logs and the `resend-credentials` Secret — a missing/invalid `RESEND_API_KEY` fails the send and is recorded on the `Notification` row itself (visible via the admin notifications endpoint), not silently swallowed |
| Catalog prices show in BRL even with the toggle set to English | `GET /api/quotations/usd-brl` returned `409` — both Frankfurter and ExchangeRate-API are unreachable (usually a cluster with no outbound internet access); the frontend degrades to native BRL by design rather than showing a broken price, see `catalog-api` logs for which provider failed and why |
| `POST /api/orders` returns `409` for a game you don't think you own | You (or a prior run through this walkthrough) already have a `Pending` or `Paid` order item for that game — `GET /api/library` and `GET /api/orders/admin` (as admin) show every order across attempts; only a `Failed` order allows a retry (`notes.md` 42, 51, 52) |
