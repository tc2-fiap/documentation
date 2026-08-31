# Future feature: USD-equivalent price display

**Status: implemented — see `../spec/notes.md` 39.** Every price is still BRL end-to-end exactly as this document originally scoped (`catalog-api`/`orders-api`/`payments-api` keep a plain currency-agnostic `decimal`, no DTO field, no settlement change). What shipped stayed even narrower than sketched below: one `catalog-api` endpoint (`GET /api/quotations/usd-brl`) proxies the rate, and the frontend converts for display when the language toggle is English. Two corrections worth keeping as reference: Frankfurter's real API doesn't match this doc's example below (`v2/rate/.../amount=` returns `422`; the working shape is `v1/latest?base=USD&symbols=BRL`, the `FrankfurterQuotationProvider` recommendation below still held), and ExchangeRate-API is used as the automatic fallback, not an either/or choice.

## Scope, if built

This would be a **read-only conversion for display only**. `Order.Price` stays the single source of truth in BRL; the payment gateway's deterministic price rule (`notes.md` 6) keys off the same BRL decimal it does today. Nothing about purchase, settlement, or the payment rule changes — a quoted USD figure would be advisory only (e.g. a small "(~US$ 5.53)" next to a catalog card's `R$ 29,99`), never a value the client can act on or the backend needs to trust.

This is the concrete case `notes.md` 36's revisit clause anticipates ("if a second currency is ever introduced") — except scoped narrowly enough that it wouldn't actually reopen that decision: no DTO needs a currency field, because BRL remains the only currency anything transacts in.

## Candidate APIs (no key required)

### Frankfurter

Supports passing the amount directly, so no client-side arithmetic is needed.

```bash
# USD -> BRL
curl "https://api.frankfurter.dev/v2/rate/USD/BRL?amount=100"

# BRL -> USD
curl "https://api.frankfurter.dev/v2/rate/BRL/USD?amount=100"
```

Example response:

```json
{
  "date": "2026-08-28",
  "base": "USD",
  "quote": "BRL",
  "rate": 5.42,
  "amount": 100,
  "converted": 542
}
```

### ExchangeRate-API (Open Access)

Does not accept an `amount` parameter — the conversion has to be computed client-side from the returned rate.

```bash
# USD -> BRL
curl -s https://open.er-api.com/v6/latest/USD | jq '.rates.BRL * 100'

# BRL -> USD
curl -s https://open.er-api.com/v6/latest/BRL | jq '.rates.USD * 100'
```

### Recommendation

**Frankfurter** is the simpler integration for this project specifically, since it takes the amount directly rather than requiring the caller to multiply by a raw rate.

## Where this would plug in, if built

- Likely lives in `catalog-api` (it already owns `Game.Price`) as a thin proxy/cache in front of Frankfurter, rather than having the frontend call a third-party API directly from the browser — keeps the API key/rate-limit surface (if the provider ever requires one) server-side, and gives one place to cache a daily rate instead of one fetch per page load.
- Frontend: a small addition next to `formatPrice()` in `src/utils/currency.ts` (`repos/frontend/`), rendered only if the conversion is available — degrading gracefully to R$-only if the rate lookup fails or is slow, the same "off means off" pattern already used for the Google sign-in button and Resend email (`notes.md` 28, 29).
- If this gets built, record it in `../spec/notes.md` with the usual shape: the decision, why now, what was rejected, and what would reopen it.
