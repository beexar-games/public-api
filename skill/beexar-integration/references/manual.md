# Integrating without an SDK

Any language. The whole contract is four inbound endpoints and three outbound
calls, and the OpenAPI documents are public:

- `https://docs.beexar.com/api/wallet.yaml` — the four callbacks **you** implement
- `https://docs.beexar.com/api/softswiss/gateway.yaml` — the launcher
- `https://docs.beexar.com/api/gateway.yaml` — the catalogue
- `https://docs.beexar.com/api/common/schemas.yaml` — shared schemas

Generate a client and server stubs from them if your ecosystem has a generator.
Everything below is what a generator will not give you.

## Signing and verifying

```
signature = lowercase_hex(HMAC_SHA256(key = api_secret, message = raw_request_body))
header    = X-REQUEST-SIGN
```

Two rules, and the integration lives or dies on them:

1. **Sign the exact bytes you send.** Serialise the body once into a buffer,
   sign that buffer, send that buffer. Serialising twice — once to sign, once to
   send — is how signatures silently stop matching.
2. **Verify the exact bytes you received.** Capture the raw body *before* any
   JSON middleware touches it. A parsed-then-re-encoded body is a different byte
   string, and the original is unrecoverable.

Compare in constant time (`hmac.compare_digest`, `hash_equals`, `crypto.timingSafeEqual`).

## What to validate before your handler runs

- `currency` matches `^[A-Z][A-Z0-9_]{2,31}$`. Reject lowercase; do not upcase it.
- `amount` matches `^\d+(\.\d{1,16})?$`, is at most 40 characters, and is
  greater than zero. **Reject the exponent form.** `"1E2000000000"` parses
  instantly in every bignum library and then allocates gigabytes on the first
  rescale — the guard has to run on the shape, before any arithmetic.
- Request bodies over **64 KiB** are refused. Enforce it on bytes actually read,
  not on `Content-Length`.
- Unknown JSON fields are **tolerated on purpose** — the contract is
  `additionalProperties: true` and the platform probes operators with an extra
  field. Do not reject them.

## Answering

Success is `200` with the response body from the spec. Errors are always this
envelope:

```json
{
  "code": "invalid_argument",
  "msg": "insufficient funds",
  "meta": { "api_code": "100", "api_message": "insufficient funds", "balance": "50.00" }
}
```

`code` is `invalid_argument` (HTTP 400) or `internal` (HTTP 500) and nothing
else. See `error-codes.md` for the api_code registry, `wallet-callbacks.md` for
idempotency and tombstones, and `timings.md` for the budgets.

## Cross-checking your implementation

The conformance fixtures the official SDKs run are in the public repository at
`api/providers/softswiss/conformance/`: request bytes, the signature over them,
the ledger before and after, and the expected response. They are
language-neutral — point your own test suite at them and you get the same
coverage the four SDKs have.

Then run the Integration Test Game — 29 scenarios against your live endpoints.
https://docs.beexar.com/guides/testing/
