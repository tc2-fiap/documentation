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

Only needed if you also want to run a single service standalone instead of the whole cluster: **.NET 10 SDK**, **Node 22+**, **Docker Compose** (see that repo's own `README.md`).

## 1. Clone the repos

This project is split across eight independent GitHub repos under [`github.com/tc2-fiap`](https://github.com/tc2-fiap) — see [`../README.en-US.md`](../README.en-US.md) for the full picture and what each one owns. Running the system itself only needs seven: the five backend services, `frontend`, and `orchestration` (`documentation` — this repo — and the separate `base-project` reference monolith aren't part of the running system).

`orchestration`'s Helm chart expects the other six as **sibling directories** on disk — its `Chart.yaml` dependencies are literal relative paths (`file://../users-api/k8s`, and so on for each service), not a registry lookup. Clone all seven into one empty parent directory, keeping the default folder names `git clone` gives you:

```bash
mkdir fiap-games && cd fiap-games
for repo in users-api catalog-api orders-api payments-api notifications-api frontend orchestration; do
  git clone https://github.com/tc2-fiap/$repo.git
done
```

Every command from here on runs from this parent directory (the one now containing all seven as siblings), unless a step says otherwise.

## 2. Create the cluster

```bash
kind create cluster --config orchestration/kind/cluster-config.yaml
```

This creates a one-node cluster named `fiap-games` with host ports 80/443 mapped in and the `ingress-ready` node label set, so an ingress controller can bind those ports directly — no `kubectl port-forward` needed for anything reached through the Ingress.

Install nginx-ingress into it:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

## 3. Install the system

```bash
cd orchestration
helm dependency update
helm install fiap-games .
```

`helm dependency update` is required after cloning (or after any subchart change) — it's what actually resolves the six `file://../*/k8s` dependencies in `Chart.yaml` into `charts/*.tgz` for Helm to install.

This brings up 8 pods: Postgres, RabbitMQ, and the five backend services plus the frontend, all in the `fiap-games` namespace, all wired to one Ingress at `http://localhost`.

## 4. Verify

```bash
kubectl get pods -n fiap-games
```

Expect 8 pods, all `Running`, all `RESTARTS` at `0`. A restart here almost always means a service started before Postgres or RabbitMQ was ready — every DB-backed service carries a `wait-for-postgres` init container specifically to prevent that, so a restart is a real signal worth investigating, not a transient to retry past.

```bash
kubectl wait --namespace fiap-games --for=condition=ready pod --all --timeout=180s
```

Schema isolation, confirmed directly against Postgres — this should be **refused**:

```bash
kubectl exec -n fiap-games deploy/postgres -- \
  psql -U orders -d fiap_games -c "SELECT * FROM users.\"Users\";"
# ERROR: permission denied for schema users
```

(Role password is `orders-dev-password` per `values.yaml`; `psql` inside the pod uses the local socket, so no password prompt.)

## 5. Demo walkthrough

Everything below goes through the one Ingress base URL — no port-forwarding, no per-service hostnames.

```bash
BASE=http://localhost
```

### Register and log in

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

Watch `kubectl logs -n fiap-games deploy/notifications-api -f` in another terminal — registering should produce a welcome-email log line within a second or two (`UserCreatedEvent` round-tripping through RabbitMQ).

### Browse the catalog and buy a game

`catalog-api` seeds itself with 8 real games (real Steam cover art, realistic BRL prices) the first time it starts against an empty database, so there's already something to browse without creating anything by hand. In the browser this goes through a cart and a checkout confirmation step (see [The same flow in a browser](#the-same-flow-in-a-browser) below); against the API directly, one call places the order:

```bash
curl -s $BASE/api/games -H "Authorization: Bearer $TOKEN" | jq
GAME_ID=$(curl -s $BASE/api/games -H "Authorization: Bearer $TOKEN" | jq -r '.items[0].id')

ORDER=$(curl -s -X POST $BASE/api/orders -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{\"gameIds\": [\"$GAME_ID\"]}")
echo $ORDER | jq
ORDER_ID=$(echo $ORDER | jq -r '.id')
```

Note the request body carries only `gameIds` — a list, since a single checkout can place one order for several games (a cart, in the browser) — and never a price; each item's price is read from `catalog-api` and snapshotted onto the order (`instructions.md` §6). Try the same `POST /api/orders` call again with the same `$GAME_ID` — it now returns `409 Conflict` ("You already own or have a pending order for: `$GAME_ID`"), since a user can't own the same game twice. That's enforced two ways: an application-level check for a friendly error message, and — the actual, race-proof guarantee — a partial unique index on Postgres itself (`notes.md` 51, 52). Either way it only unblocks again if that order later settles `Failed`.

### Watch `Pending` become `Paid`

```bash
watch -n1 "curl -s $BASE/api/orders/$ORDER_ID -H \"Authorization: Bearer $TOKEN\" | jq '.status'"
```

The simulated payment gateway decides deterministically by price, not randomly (`notes.md` 6) — a game priced at `999.00` or under, not ending in `.13`, is `Approved`; the order flips to `Paid` within `PAYMENT_PROCESSING_DELAY_SECONDS`. Once `Paid`, it appears in the library:

```bash
curl -s $BASE/api/library -H "Authorization: Bearer $TOKEN" | jq
```

To see a rejected purchase instead, buy a game priced above `999.00`, or one whose price ends in `.13` — the order settles to `Failed` and never appears in the library.

### USD quotation and your own checkout

```bash
curl -s $BASE/api/quotations/usd-brl -H "Authorization: Bearer $TOKEN" | jq
```

Returns the current USD→BRL rate (Frankfurter, falling back to ExchangeRate-API if it's down), cached server-side for an hour — call it twice and the second response is near-instant. This is what the frontend uses to show a USD-equivalent price when the language toggle is set to English; every backend price stays a plain BRL `decimal` regardless (`notes.md` 39).

```bash
curl -s $BASE/api/payments/checkout/$ORDER_ID -H "Authorization: Bearer $TOKEN" | jq
```

Unlike the admin-only `/api/payments/{orderId}` below, this route is for the order's own owner — it returns the payment's status, gateway, price, and (only when a real PIX gateway produced one) a QR code and copy-paste code, never the full raw gateway payload. With the default `simulated` gateway, `pixCopyPasteCode`/`pixQrCodeBase64` are both `null` — there's nothing to scan, the order just settles on its own (`notes.md` 40).

### Admin login and the cross-service audit trail

The seeded admin account (`values.yaml`'s `admin.email`/`admin.password` — `admin@fiapgames.local` / `admin-dev-password-change-me` by default) can see every user's orders and the full lifecycle of any one of them:

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

The payments and notifications responses include the actual request/response payloads exchanged with the gateway and email provider — real JSON, not a summary, even for the simulated gateway (`notes.md`'s audit-trail entry). Confirm the boundary holds — the same four calls with `$TOKEN` (a non-admin) instead of `$ADMIN_TOKEN` should all return `403`.

The four calls above are scoped to one order. To browse every event/message across the whole system instead — every `UserCreatedEvent` ever published, every purchase-flow event, every payment, every notification, not just one order's — use the system-wide "list all" endpoints (`notes.md` 43):

```bash
curl -s "$BASE/api/users/admin/events" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s "$BASE/api/orders/admin/events" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s "$BASE/api/payments/admin" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
curl -s "$BASE/api/notifications/admin" -H "Authorization: Bearer $ADMIN_TOKEN" | jq
```

Each is paginated (`page`/`pageSize`, capped at 100) and accepts optional filters — a `from`/`to` UTC date range on all four, plus `eventType` (users/orders), `status` (payments), and `type`/`status` (notifications). `catalog-api` has no equivalent endpoint — it publishes and consumes nothing, so there's nothing to list. Same `403` boundary check applies here too.

### The same flow in a browser

```bash
open http://localhost   # or just navigate there
```

Register or log in (a Google sign-in button appears automatically only if `Google:ClientId` is configured — see `notes.md` 28). Add one or more games to your cart from the catalog and review them on `/cart`, or use `Buy Now` on a single game — either way you land on the same `/checkout` confirmation page before anything is actually ordered: an itemized review (cover image, title, genre/platform per game) and the total in both BRL and USD. Confirming lands on the order page: the same line items, an order-and-payment-status box, the total, and — if a real PIX gateway is configured — a QR code to scan. The `Pending` → `Paid`/`Failed` transition appears the moment it happens, pushed over a Server-Sent Events connection rather than polled — unlike the `watch`-based `$ORDER_ID` loop above, which is just a convenient way to observe the same transition from the command line (`notes.md` 53). Logging in as the seeded admin surfaces two nav links: the all-orders view (and its per-order detail page, composing the same four order-scoped endpoints above into one screen) and "System Events" — the `/admin/events` page from the curl calls above, with dropdown filters for source, kind, and type plus a date range, and a click-to-expand raw JSON payload on every row.

The header carries an EN/PT toggle, visible even before logging in. Toggling to Portuguese always shows native BRL (e.g. `R$ 29,99`); toggling to English converts every catalog price to its USD equivalent using the live quotation, falling back to BRL if the rate lookup is ever unavailable — never a blank or broken price. The cart, checkout, and order pages always show both currencies together regardless of the toggle. The language choice itself persists across a reload (`notes.md` 35, 36, 39).

## 6. Tear down

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
| `helm dependency update` can't resolve a dependency (`../users-api/k8s` not found, etc.) | The six sibling repos need to be cloned next to `orchestration/`, with their default folder names — see [step 1](#1-clone-the-repos) |
| `curl $BASE/...` connection refused | The ingress controller isn't ready yet, or the kind cluster wasn't created with the port mappings in `kind/cluster-config.yaml` |
| Google button never appears | Expected with no `Google:ClientId` configured — `GET /api/users/config` reports `googleSignInEnabled: false` and the frontend hides it deliberately, rather than showing a button guaranteed to fail |
| No email arrives despite `EMAIL_PROVIDER=resend` | Check `notifications-api` logs and the `resend-credentials` Secret — a missing/invalid `RESEND_API_KEY` fails the send and is recorded on the `Notification` row itself (visible via the admin notifications endpoint), not silently swallowed |
| Catalog prices show in BRL even with the toggle set to English | `GET /api/quotations/usd-brl` returned `409` — both Frankfurter and ExchangeRate-API are unreachable (usually a cluster with no outbound internet access); the frontend degrades to native BRL by design rather than showing a broken price, see `catalog-api` logs for which provider failed and why |
| `POST /api/orders` returns `409` for a game you don't think you own | You (or a prior run through this walkthrough) already have a `Pending` or `Paid` order item for that game — `GET /api/library` and `GET /api/orders/admin` (as admin) show every order across attempts; only a `Failed` order allows a retry (`notes.md` 42, 51, 52) |
