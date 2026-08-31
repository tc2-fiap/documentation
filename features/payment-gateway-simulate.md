# Multi-gateway payment integration

**Status: implemented, live sandbox verification pending.** `payments-api` now supports `AbacatePayGateway` and `MercadoPagoGateway` alongside `SimulatedPaymentGateway`, composed into an ordered fallback chain (`PaymentGatewayChain`) selected by `PaymentGateway:Providers` (a comma-separated, ordered ConfigMap value — e.g. `abacatepay,mercadopago,simulated`). `simulated` remains the default and stays the guaranteed last resort in any real chain. See [`../spec/notes.md`](../spec/notes.md) 38 for the full decision record — this document now describes what was built, and stays as the place to look before touching this code again.

**What's built and unit-tested:** the chain itself (Strategy + Chain of Responsibility, falls through only on `PaymentGatewayUnavailableException`), both gateways' real outbound `HttpClient` calls (create-charge, dev/sandbox approval trigger, status check), the `Processing` payment sub-state, the polling `BackgroundService` that confirms real charges, and a webhook receiver + signature verification that's wired but dormant (see below).

**What's not done yet:** live end-to-end verification against real AbacatePay/Mercado Pago sandbox accounts — real API keys, watching a poll actually observe a status transition, exercising the webhook path behind a real public Ingress. All gateway tests mock the HTTP layer; nothing in this codebase has made a real call to either provider.

## Why polling instead of the webhook design this doc originally proposed

The original version of this document (and `instructions.md` §5.2) proposed a webhook receiver as the confirmation mechanism — architecturally correct, and still the plan for a real deployment. But this project runs local-only (`instructions.md` §11): there's no stable public URL for either provider to call back to, and a free tunnel's rotating URL was already the objection `notes.md` 4 raised against making a real gateway the default. So the **active** path is `PaymentStatusPollingService` — an exponential-backoff `BackgroundService` that calls each provider's own GET status-check endpoint until a charge resolves (or times out into a forced `Rejected`, so an order can never hang forever).

The webhook receiver (`IPaymentWebhookHandler`, `POST /api/payments/webhooks/{provider}`, HMAC verification) is still fully built and wired — it's just unreachable in this topology. Both paths converge on the same idempotent `PaymentFinalizationService.FinalizeAsync`, so activating the webhook later (once a real Ingress exists) needs no further code changes.

## Provider reference: AbacatePay (Pix QR Code API, dev mode)

Routes verified against current AbacatePay docs — the original version of this section had reconstructed the create-charge URL from a truncated source; this has since been checked.

**Create a charge:**

```
POST https://api.abacatepay.com/v1/pixQrCode/create
Authorization: Bearer <API_KEY>
Content-Type: application/json
```

```json
{
  "amount": 1500,
  "expiresIn": 3600,
  "description": "FIAP Games order <orderId>",
  "metadata": {
    "externalId": "<orderId>"
  }
}
```

Response: `data.id` (e.g. `pix_char_...`) is the id every later call keys on — `AbacatePayGateway` stores it as `Payment.ExternalReference`.

**Triggering dev-mode approval — API call, not a browser click:**

```
POST https://api.abacatepay.com/v1/pixQrCode/simulate-payment?id=<pixQrCodeId>
Authorization: Bearer <API_KEY>
```

`AbacatePayGateway.ChargeAsync` calls this immediately after create, so the cascade stays autonomous — nobody opens the checkout page and clicks "Simular Pagamento" by hand (same no-human-in-the-loop rationale as `notes.md` 7). If this call itself fails, the charge still stands; it settles via the poller's timeout instead.

**Checking status (the active polling target):**

```
GET https://api.abacatepay.com/v1/pixQrCode/check?id=<pixQrCodeId>
Authorization: Bearer <API_KEY>
```

`data.status` ∈ `PENDING, EXPIRED, CANCELLED, PAID, REFUNDED` — mapped to the local `PaymentAttemptStatus` in `AbacatePayGateway.CheckStatusAsync` (`PAID` → `Approved`; `EXPIRED`/`CANCELLED`/`REFUNDED` → `Rejected`; `PENDING` → keep polling).

## Provider reference: Mercado Pago (classic Payments API only)

**This project uses only the classic Payments API** (`payment_method_id: pix`), never the newer Checkout API/Orders product — that has an incompatible status vocabulary (`created`/`processed`/`action_required`/...) and isn't used anywhere here.

**Create a charge:**

```
POST https://api.mercadopago.com/v1/payments
Authorization: Bearer <ACCESS_TOKEN>
X-Idempotency-Key: <fresh GUID per attempt>
Content-Type: application/json
```

```json
{
  "transaction_amount": 15.00,
  "description": "FIAP Games order <orderId>",
  "payment_method_id": "pix",
  "payer": {
    "email": "comprador_teste@email.com"
  },
  "metadata": {
    "external_id": "<orderId>"
  }
}
```

`X-Idempotency-Key` is a fresh GUID per attempt, not the `OrderId` — Mercado Pago's idempotency semantics are its own concept, distinct from this system's `OrderId`-keyed idempotency (`notes.md` 8). Response `id` (numeric) is stored as `Payment.ExternalReference`.

**Triggering sandbox approval — called immediately after create, same autonomy rationale as AbacatePay:**

```
POST https://api.mercadopago.com/v1/payments/<PAYMENT_ID>/sandbox/simulate
Authorization: Bearer <ACCESS_TOKEN>
```

Body `{}` or `{"status": "approved"}`.

**Checking status (the active polling target):**

```
GET https://api.mercadopago.com/v1/payments/<PAYMENT_ID>
Authorization: Bearer <ACCESS_TOKEN>
```

`status` ∈ `pending, approved, in_process, rejected, cancelled, refunded, in_mediation` — mapped in `MercadoPagoGateway.CheckStatusAsync` (`approved` → `Approved`; `rejected`/`cancelled`/`refunded`/`in_mediation` → `Rejected`; `pending`/`in_process` → keep polling).

**Still unverified:** Mercado Pago's real `x-signature` webhook header is a composite `ts=...,v1=...` manifest, not a bare HMAC of the raw request body. `MercadoPagoGateway.VerifyAndParseAsync` currently computes the same HMAC-SHA256-over-raw-body shape `AbacatePayGateway` uses, as a structural placeholder — this needs replacing with the real manifest format before the (currently dormant) webhook path is ever put behind a live Ingress.

## If this gets extended further

- A new gateway needs one new class implementing `IPaymentGateway` (+ `IPaymentStatusChecker` if it can return `Processing`) and its `ProviderName` constant appended to `PaymentGateway:Providers` — no chain, consumer, or poller code changes (see `notes.md` 38 for why this stays string-keyed rather than an enum).
- Sandbox/dev-mode credentials only — never production credentials, and never present a sandbox result as a real transaction (`notes.md` 5).
- Keep `SimulatedPaymentGateway` as the CI/test/fallback path and the config default; don't remove it.
- When live sandbox verification actually happens, record it as its own `notes.md` entry rather than folding it silently into 38.
