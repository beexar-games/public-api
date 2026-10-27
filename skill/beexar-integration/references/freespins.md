# Freespins

Optional. Only for an operator that gives players free rounds. The API is the
SoftSwiss Freespins API one to one: three calls you make, one callback you
answer. Nothing changes in the launch or in the four round callbacks — free
rounds never reach `/betwin`.

## The flow

1. **`POST /api/v1/softswiss/bet/levels`** — the bet levels each game offers,
   per currency. A level is the **total** bet of one round. Filters `games[]`
   and `currencies[]` are optional; unknown values are ignored, not refused.
2. **Record the issue in your own ledger** — `issue_id`, the player's account
   id and currency — *before* the next call. The result names only `issue_id`.
3. **`POST /api/v1/softswiss/freespins/issue`** → `{}` —
   `freespins_quantity` rounds at `bet_level` in `games[]`, playable until
   `valid_until`. The quantity is shared by all the listed games, not given in
   each.
4. The player launches one of the games the usual way (Launcher/Real, nothing
   new in it). The game opens with the freespins, its bet locked.
5. **`POST <freespins finish URL>`** — Beexar → you, once, when the issue is
   played out or expires with spins played: `{issue_id, amount}`. Credit
   `amount`, answer `{balance}`.
6. **`POST /api/v1/softswiss/freespins/cancel`** → `{}` — any time, even
   mid-play. A canceled issue never reaches step 5; what it won is forfeited.

Both outbound calls are signed exactly like the launcher:
`X-REQUEST-SIGN: hex(HMAC-SHA256(raw_body, api_secret))`, `casino_id` in the
body.

```json
{
  "casino_id": "your-operator-slug",
  "issue_id": "fs-123",
  "games": ["dice"],
  "freespins_quantity": 10,
  "bet_level": 1,
  "valid_until": "2026-12-01T00:00:00Z",
  "account": { "id": "player_123", "currency": "EUR" }
}
```

`valid_until` is UTC with a `Z`, at most one month ahead.

## Retrying Issue

A repeated Issue with the same `issue_id` **and the same terms** answers like the
first, so retry a timeout or a 5xx with the **same body**. Recomputing
`valid_until` ("now + 7 days") on the retry makes it other terms, and other terms
under one `issue_id` are refused with api_code `610`.

## The callback: `/freespins/finish`

```json
{ "issue_id": "fs-123", "amount": "12.50" }
```

- **Sent once per issue**, and retried with the same body byte for byte — after
  1, 3 and 5 minutes, then every 5 — until you answer `200`. Each attempt has
  15 s (`OperatorAttemptTimeout` in `timings.md`); a slower answer counts as
  no answer. Refusals are retried too, so do not refuse what you can credit.
- **Idempotent on `issue_id`**, the way `/betwin` is on `id_provider`: the first
  one credits and stores the balance it answered; a repeat answers that stored
  balance and credits nothing. Same database transaction as the credit.
- **`amount` may be zero** (`"0.00"`, or `"0"` in a currency without
  decimals) — the freespins won nothing. Accept it and close the
  issue; unlike a `/betwin` amount, zero is valid here. It is in the issue's
  currency, at that currency's scale.
- **Unknown `issue_id`** → api_code `610` (invalid freespins issue).
  **An issue you canceled** → api_code `605`. Beexar never sends a result for an
  issue whose cancel it accepted.
- Mark an issue canceled only when Beexar **accepted** your cancel. A cancel
  refused with `620` means the issue is already played out — its result is on
  its way.

## Configuration

The callback goes to a **fifth callback URL**, the operator's *freespins finish
URL* in the backoffice, next to the four round URLs. With the SDKs it ends in
`/freespins/finish` (e.g. `https://your-host/wallet/freespins/finish`): they
route on the path suffix, longest route first, so it can share a prefix with
`/finish`.

Leave it empty when the operator does not offer freespins: Issue then answers
`406` and nothing else changes.

## In the SDKs

The callback is an **optional** fifth method. A handler without it keeps
working; the route then answers like an unknown one (api_code `404`). See the
language file for the exact names.

## Errors from the calls

Same envelope as the launcher; branch on `meta.api_code`.

| api_code | Call | Meaning |
|---|---|---|
| `403` | all | invalid request signature (HTTP 403) |
| `400` | all | malformed body; `freespins_quantity` above 10000 |
| `404` / `410` / `420` | all | `casino_id` unknown / disabled / wrong integration type |
| `406` | Issue | this feature is not enabled for your casino: no freespins finish URL configured |
| `154` | Issue | currency not allowed |
| `405` | Issue | a game your casino does not have (name games by slug, no provider prefix) |
| `608` | Issue | a game that plays no freespins |
| `601` | Issue | a game has no such `bet_level` in the currency |
| `602` | Issue | no game |
| `603` | Issue | bad `valid_until` |
| `605` | Issue | this `issue_id` was canceled before it was issued |
| `610` | Issue | this `issue_id` was issued with other terms |
| `611` | Cancel | the issue has already expired |
| `620` | Cancel | the issue is played out and cannot be canceled |
| `500` | all | platform error — retryable |
