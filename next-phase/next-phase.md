# Next phase

Pending work — planned, not built. Everything else in this repo documents the system as it actually runs today (`kind`, `http://localhost`, dev-only secrets, a single `Admin` role); this file is the opposite: what's decided-but-not-started for whatever comes after it. English only, same as `../features/` and `../spec/` — this is engineering/planning material, not reader-facing narrative, so it doesn't get a `.pt-BR` pair.

When a piece of this actually ships, it moves the same way `../features/` already works: this file gets a `**Status:**` banner updated in place for that section, and a new dated entry goes into [`../spec/notes.md`](../spec/notes.md) with what was actually decided at build time (which may differ from what's sketched here) — this file is a starting point for that work, not a spec frozen in advance of it.

## Priority order

1. **Payments and email: go live** — both already built, just not turned on for real.
2. **Production deployment** — the payments/email webhook path depends on this one's TLS/domain work.
3. **Manager/Admin role split** — planned, not sequenced against the two above yet.
4. **Everything else** — see the scratch list at the end.

---

## 1. Payments and email: go live

**Status: planned, not started. Top priority of this file.**

Both payment webhooks and email delivery are already built — this isn't about writing new integrations, it's about actually flipping them on in a real environment instead of leaving them dormant (webhooks) or optional-and-unused (email).

### Email — the cheap win

Resend email is **fully implemented already**: `ResendEmailSender.cs` in `notifications-api`, selected via the `Email:Provider` config switch (`console` default, `resend` opt-in), already wired into the Helm chart's Secret template (`Resend__ApiKey` sourced from `resend-credentials.RESEND_API_KEY`). Going live needs:

1. A real Resend API key, set as the Secret's real value (currently defaults to an empty string).
2. `Email:Provider` flipped from `console` to `resend`.

That's it — zero code changes. This is the fastest, lowest-risk thing to flip live in this whole file, and doesn't depend on production deployment at all — it works the same whether the cluster is local or real, since it's an outbound call, not an inbound one.

### Webhooks — needs a real public endpoint first

`AbacatePayGateway` and `MercadoPagoGateway` are fully built and unit-tested, with HMAC signature verification, wired to `POST /api/payments/webhooks/{provider}` in `payments-api`. They're dormant today for exactly one reason: there's no public HTTPS endpoint for either provider to call back to — the active confirmation path in the current local topology is `PaymentStatusPollingService` polling each provider's own status endpoint instead.

Going live needs, in order:

1. **A real public HTTPS endpoint and domain** — see §2 below; this picks up once that exists.
2. **Register the callback URL** in each provider's own dashboard (AbacatePay, Mercado Pago) — pointing at `https://<real-domain>/api/payments/webhooks/{provider}`.
3. **Switch `PaymentGateway:Providers`** from `simulated` to include the real provider(s) first, e.g. `abacatepay,mercadopago,simulated` — `simulated` stays last as the guaranteed fallback, same ordered-chain design already built.
4. **Populate real gateway API keys** as Secret values (currently empty-string defaults).
5. **Verify Mercado Pago's real `x-signature` header format** against live docs before trusting it in production — this is still an unverified placeholder (flagged in `../spec/notes.md` 38), the one piece of this whole path that isn't already confirmed correct.

Once the webhook path is live, whether to keep `PaymentStatusPollingService` running alongside it (as a safety net for a missed/failed webhook delivery) or retire it is worth deciding at that point, not now.

**Sequencing**: email first — no dependencies, no code, a single config flip. Webhooks after §2's TLS/domain work lands, since they have a hard prerequisite that email doesn't.

---

## 2. Production deployment

**Status: planned, not started — only if/when actually requested.**

Everything built so far runs exclusively on a local `kind` cluster (`instructions.md` §11 already states cloud deployment is out of scope for the current spec — this section is that future phase, kept separate rather than folded back into `instructions.md`). This is a strategy, not a decision: it doesn't commit to a specific cloud provider, since going to production hasn't been asked for yet — it exists so that when it is, the cheapest viable path is already mapped out instead of improvised under time pressure.

### Current state — the gap this has to close

- **CI already builds and pushes every image to GHCR** on push to `main` (`docker-build-and-push` job, tags = full commit SHA + `latest`) in 7 of the 8 code repos — `orchestration` itself has no CI workflow yet. A production deploy can reuse these images unchanged.
- **Zero resource requests/limits anywhere** — every Deployment across every subchart (Postgres, RabbitMQ, all six backends, frontend) has no `resources:` block at all. Not a baseline to preserve; a blank slate that needs real numbers before scheduling on a small, cheap node.
- **Postgres**: single instance, `postgres:16-alpine`, one 2Gi PVC, no backup strategy.
- **RabbitMQ**: single instance, **no PVC at all** — queued/unrouted messages don't survive a pod restart today. A real gap, not just a local-dev shortcut, since it'd lose real events in production.
- **Ingress**: no TLS, no cert-manager, no real hostname anywhere. nginx-ingress isn't even a Helm dependency — it's installed via a raw `kubectl apply` of a `kind`-specific manifest (per `GETTING_STARTED.md`), which doesn't generalize to a real cluster.
- **Secrets**: every sensitive value defaults to something like `dev-only-secret-change-me-please-32chars-min` or an empty string, with a `values.yaml` comment that real values are meant to be injected at install time — but no actual external-secrets/sealed-secrets pattern exists yet, just placeholders.

### Recommended cheapest path

A single small VPS running a lightweight single-node Kubernetes distro — **k3s** or **k0s** — with the existing `orchestration` Helm chart installed nearly unchanged. This is the cheap option specifically because it's the one requiring the *least new work*: it reuses 100% of the current Helm manifests and the images CI is already building, rather than re-platforming onto a PaaS (which would mean rewriting every subchart as a different deployment shape — Cloud Run services, ECS task definitions, whatever the target expects). It is not necessarily the "best" architecture in the abstract, just the one that gets a real, working production deployment for the least additional engineering.

### What has to be added before that's viable for real traffic

- **Resource requests/limits** — every Deployment needs real `resources.requests`/`resources.limits`; currently none exist anywhere, so a small node has no basis to schedule sanely without them.
- **TLS** — a real domain, `cert-manager`, and Let's Encrypt. nginx-ingress needs to move from the current raw, `kind`-specific `kubectl apply` to a proper Helm dependency the umbrella chart manages, so it's reproducible on a real cluster the same way everything else already is.
- **Real Secret values** — every currently-placeholder value (JWT signing key, every Postgres role password, gateway API keys, the Resend API key) needs a real value injected at install time, never checked into the chart's defaults.
- **RabbitMQ persistence** — add a PVC before production; this is the one outright correctness gap in the current setup, not just a hardening step.
- **Postgres backups** — at minimum, a scheduled `pg_dump` job. A managed Postgres instance is a reasonable paid upgrade path if budget allows, since it's the one component holding the only source of truth in the system — but that's an optional step up, not a requirement to go live.

### Alternatives, briefly — not decided here

- **A small managed Kubernetes control plane** (a hosted control plane, self-managed nodes): more expensive than a bare VPS, buys managed upgrades/HA for the control plane specifically, which this system's scale doesn't obviously need yet.
- **Per-service PaaS hosting** (Render/Railway-style, one deployment per repo): removes the need to run Kubernetes at all, but means abandoning every existing Helm chart and CI deploy step — a materially bigger rewrite than the recommended path, for a system that was deliberately built Kubernetes-first.

Neither is ruled out — both are just more expensive in money, engineering time, or both, than the recommended path, for a system this size.

### CI/CD path

Extend the existing `docker-build-and-push` GHCR job (already present in 7 of 8 repos) with a deploy step — e.g. `helm upgrade` over SSH to the VPS, or a lightweight GitOps agent watching the same registry — rather than inventing new tooling. `orchestration` needs its first CI workflow at all for this, since it currently has none.

---

## 3. Split `Admin` into `Manager` and `Admin`

**Status: planned, not started.**

### Motivation

Today's single `Admin` role conflates two different concerns: visibility into people and their purchases (who bought what, who needs to be deactivated, who needs an email) and visibility into the system itself (cross-service event audit, cluster health). One role doing both means anyone who needs either capability gets both, whether or not that's appropriate. Split it into two peer roles:

- **Manager** — user visibility and management (list, search, edit, deactivate, promote), plus visibility into every user's purchases (orders, payments, notifications).
- **Admin** — system-level view (cross-service event audit, `platform-api`'s pod/health data).
- **Both** keep full catalog CRUD — cataloging isn't a people concern or a system concern, it's product data either role should be able to manage.

### Current state (why this is a reassignment, not a rewrite)

Every admin-gated route today uses the same idiom: `RequireAuthorization(p => p.RequireRole(...))` — `users-api` via `nameof(Domain.UserRole.Admin)`, every other service via the string literal `"Admin"` (the JWT role claim is just a string; there's no enum shared across service boundaries). A second role extends this exact pattern — no new auth mechanism needed.

Current `Admin`-only surface, service by service:

| Service | Route(s) | What it does |
|---|---|---|
| `users-api` | `GET /{id}`, `GET /` (paged list), `GET /admin/search`, `PUT /{id}`, `DELETE /{id}`, `PUT /{id}/role` | User management |
| `users-api` | `GET /admin/events` | Cross-service audit feed (user domain events) |
| `catalog-api` | `POST`, `PUT /{id}`, `DELETE /{id}` | Catalog CRUD (`GET` routes are already open to any authenticated user) |
| `orders-api` | `GET /admin` (all orders, filterable), `GET /{id}/events` | Purchase visibility |
| `orders-api` | `GET /admin/events` | Cross-service audit feed |
| `payments-api` | `GET /admin`, `GET /{id}` (whole group is Admin-gated; note `GET /{id}` has no ownership check, unlike the player-facing `GET /checkout/{id}`) | Purchase/payment visibility |
| `notifications-api` | `GET ?orderId=`, `GET /admin` | Purchase/notification visibility |
| `platform-api` | `GET /admin/pods` | Cluster/system health — the one purely-infra endpoint in the system |
| every service | `GET /version` (or `GET /api/<prefix>/version`) | Build info — not sensitive either way |

Frontend (`App.tsx`, all currently under `<RequireAdmin>`): `/admin/orders` + `/admin/orders/:id` (purchases), `/admin/events` (cross-service audit, already titled "System Events" in the nav), `/admin/games` + `/new` + `/:id/edit` (catalog), `/admin/system` (pods + every service's `/version`).

### New role model

`UserRole` (`users-api`'s `Domain/UserRole.cs`) gains a third value, `Manager`, alongside `Player`/`Admin` — a peer role with its own scope, not a superset or subset of `Admin`. One role per user, same as today: no multi-role model, JWT role claim stays a single string. `RequireRole("Admin", "Manager")` covers routes both should reach.

### Reassignment

| Route(s) | Goes to |
|---|---|
| `users-api`: list/search/get/update/delete/role-change | `Manager` |
| `users-api`, `orders-api`, `payments-api`, `notifications-api`: every `admin/events`-shaped audit route (feeds the same "System Events" page) | `Admin` |
| `catalog-api`: all CRUD routes | `Manager` **and** `Admin` |
| `orders-api`: `GET /admin`, `GET /{id}/events` | `Manager` |
| `payments-api`: `GET /admin`, `GET /{id}` | `Manager` |
| `notifications-api`: `GET ?orderId=`, `GET /admin` | `Manager` |
| `platform-api`: `GET /admin/pods` | `Admin` only |
| every `/version` route | `Manager` **and** `Admin` |

Frontend pages follow the same split: `/admin/orders`, `/admin/orders/:id` → Manager. `/admin/events`, `/admin/system` → Admin. `/admin/games*` → both.

### New capabilities — these don't exist yet, and are real work, not reassignment

**Activate/deactivate a user.** No `IsActive`/`Disabled`/`Suspended` concept exists anywhere in the `User` domain entity today. Needs:
- `User.IsActive` (bool, default `true`) + an EF Core migration.
- A new Manager-only endpoint, e.g. `PUT /api/users/{id}/status`.
- `POST /api/users/login` checks `IsActive` and rejects a deactivated user's login attempt immediately.
- **Tradeoff to accept, not solve**: a token already issued before deactivation stays valid until it expires (≤60 minutes today) — nothing else in this system revokes a token early either (there's no session store, no blocklist), so adding a per-request DB check just for this would be a new pattern solely for this feature. Recommend living with the ≤60-minute window rather than adding one.

**Send an email to a user.** No admin-triggered email path exists today — every email is event-driven (`UserCreatedEvent`/`PaymentProcessedEvent` → `notifications-api`). Needs a new Manager-only endpoint, e.g. `POST /api/notifications/admin/send` (recipient `userId`, subject, body), reusing the existing `IEmailSender`/`Email:Provider` abstraction already built for welcome/purchase emails — not a new delivery pipeline, not a new provider integration.

### Frontend impact

- `AuthContext` needs an `isManager` flag alongside the existing `isAdmin`.
- `RouteGuards.tsx` needs a `RequireManager` (or a generalized "requires one of these roles" guard, since `/admin/games*` needs to accept either).
- `NavBar.tsx`'s admin nav links split across the two flags per the reassignment table above.
- Two new Manager-only pages: a user list/management screen (list, search, deactivate, promote) and a send-email screen. Neither has an existing analog to extend — both are new.

### Seeding

Keep the existing seeded `admin`/`player` `values.yaml` keys (`orchestration/values.yaml`), add a third seeded `manager` account the same idempotent way `users-api`'s `SeedUserIfConfiguredAsync` already seeds the other two — so a demo/grading walkthrough always has one of each role ready without registering by hand.

### Open question, not resolved here

Who can promote a user to `Manager` or `Admin`? Today any `Admin` can promote anyone to anything via `PUT /{id}/role`. Recommendation: `Manager` keeps that power, since it's bundled with user management generally — but this is a real call worth confirming before implementation, not something to infer from this file alone.

---

## 4. Everything else

A catch-all for ideas smaller or less-formed than the three sections above. Add a one-line bullet here as ideas come up; a bullet graduates to its own numbered section once it's actually being planned in detail.

- (nothing yet)
