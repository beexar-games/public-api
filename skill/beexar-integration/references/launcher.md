# Launching a game

Three calls. Two are signed; one is public.

## Start a real-money session

`POST /api/v1/softswiss/launcher/real`, signed with
`X-REQUEST-SIGN: hex(HMAC-SHA256(raw_body, api_secret))`.

```json
{
  "casino_id": "your-operator-slug",
  "game": "dice",
  "account": { "id": "player_123", "currency": "EUR" },
  "locale": "en"
}
```

Answer: `{ "launch_url": "https://games.beexar.com/dice/your-slug/dice-game?token=…" }`.

Put that URL in an iframe. Do not proxy the game through your own server, and do
not cache the URL — the token identifies one session.

`account` may also carry `firstname`, `lastname`, `nickname`, `email`,
`country`, `date_of_birth`, `registered_at` and `tags`. `urls.return_url` and
`urls.deposit_url` control where the game's own navigation sends the player.

## Start a demo session

`POST /api/v1/softswiss/launcher/demo` — same shape, `currency` and `balance`
instead of an account. Demo play makes **no wallet callbacks**; the balance is
virtual and lives on the platform.

There is a browser-only mode where the signature is 64 zero characters. Do not
use it from a server, and do not implement it in a server SDK: it authenticates
nothing, and a habit of disabling HMAC is exactly the habit not to build.

## Read the catalogue

`GET /api/v1/operator/games?operator=your-slug` — **public, no signature**.
Returns the games enabled for you, with RTP, volatility, thumbnail, supported
currencies and the country blacklist.

## Errors

Same envelope as everything else, with one asymmetry worth knowing: a **bad
signature on a launcher call answers HTTP 403**, while a bad signature on a
*wallet callback* must be answered by you with HTTP **400** and api_code `403`.
Branch on `meta.api_code`, and the asymmetry stops mattering.

| api_code | Meaning |
|---|---|
| `403` | invalid request signature |
| `400` | malformed body; `meta.api_message` names the field |
| `404` | `casino_id` unknown or mismatched |
| `405` | game not available to your casino |
| `410` | casino disabled |
| `420` | endpoint not available for your integration type |
| `154` | currency not allowed, needs a feature subscription, or bet limits missing |
| `500` | platform error — retryable |
