---
name: beexar-integration
description: Integrate the Beexar casino game platform into a codebase. Covers the official Node, PHP, Go and Python SDKs (@beexar/sdk, beexar/sdk, github.com/beexar-games/beexar-go, beexar on PyPI) plus a manual path for any other language implemented directly against the public OpenAPI contract. Wires up the launcher calls (launch a real or demo game session, list the operator catalogue) and the four inbound seamless-wallet callbacks — POST /balance, /betwin, /rollback, /finish — with X-REQUEST-SIGN HMAC-SHA256 verification over the raw request body, decimal-string money, idempotency on id_provider and rollback tombstones. Use this skill when integrating Beexar, building a seamless wallet or game-aggregator integration, implementing bet/win/rollback/balance callbacks, verifying X-REQUEST-SIGN, debugging a signature mismatch, or working with launch_url and casino_id.
---

# Beexar integration

Integrate the Beexar game platform into this codebase. There are two halves, and
the second one is the work.

- **Launcher** (you → Beexar): start a real-money or demo session, read the game
  catalogue. Three calls, one of them unauthenticated.
- **Seamless wallet** (Beexar → you): four inbound callbacks — `GET`-like
  `POST /balance`, `POST /betwin`, `POST /rollback`, `POST /finish` — that move
  the player's money in the operator's own ledger. Every one is HMAC-signed and
  must be verified, idempotent, and answered inside a fixed time budget.

## Contract this skill was built against

<!-- BEGIN GENERATED: contract-versions -->

| Contract | Version |
|---|---|
| `api/common/schemas.yaml` | `v.2026.06.20` |
| `api/gateway.yaml` | `v.2026.06.09` |
| `api/providers/softswiss/wallet.yaml` | `v.2026.03.03` |
| `api/softswiss/gateway.yaml` | `v.2026.06.20` |

<!-- END GENERATED: contract-versions -->

## Step 1 — analyse the project first

Work out, from the codebase and not from assumption:

1. **Language and runtime** → picks the path.
   - Node/TypeScript, PHP, Go or Python → official SDK. Read
     `references/node.md`, `references/php.md`, `references/go.md` or
     `references/python.md`.
   - Anything else (Ruby, C#, Java, Elixir, …) → no SDK. Read
     `references/manual.md` and implement against the OpenAPI documents.
2. **Which halves apply.**
   - A wallet / ledger / player-account service → the four callbacks.
   - A player-facing site or app → the launcher (`launch_url` in an iframe).
   - A back-office tool → the catalogue.
   - Most integrations need the callbacks *and* the launcher.
3. **Where HTTP routes and configuration live**, and **what the existing ledger
   looks like** — the callbacks have to run inside its transaction, so find it
   before writing anything.

If the project's purpose is unclear, ask rather than guess.

## Step 2 — read what you actually need

- The **language file** from step 1: install, client setup, exact method names,
  a working example.
- `references/wallet-callbacks.md` — **read this before writing any callback
  handler.** Idempotency, tombstones, atomicity, what a replay must answer.
- `references/error-codes.md` — the envelope and the full api_code registry.
- `references/timings.md` — how long you have and what gets retried.
- `references/launcher.md` — only if the project launches games.

## Step 3 — implement

1. Configuration from **environment variables**, never hardcoded:
   `BEEXAR_CASINO_ID` (your operator slug) and `BEEXAR_API_SECRET`. Use
   `BEEXAR_BASE_URL` if the project needs to point at a non-production gateway.
2. **Callbacks**: wire all four routes; the SDK verifies the signature, parses
   and validates before your code runs. Implement the four methods against the
   existing ledger, inside one database transaction per request.
3. **Launcher**: wrap the client in whatever service layer the project already
   uses. Put `launch_url` in an iframe; do not proxy the game.
4. If the project has tests, add coverage for the new code. Do not scaffold a
   test framework that is not already there.
5. If it has a README or docs, record the new environment variables.

## Step 4 — verify

- List the files you changed and why, and the environment variables a human
  must set.
- Confirm it builds and that any tests you added pass.
- Tell the human to run the **Integration Test Game** — 29 scenarios against
  their live callbacks, launched with `game: "testgame"`. It is the only real
  proof the integration is correct. https://docs.beexar.com/guides/testing/
- Surface anything you had to decide for them.

## If the SDK or the platform disagrees with this skill

This skill is generated from the platform's own source, but an installed copy
can still be older than the API. If a method named here does not exist, a
response carries a field not described here, or an api_code shows up that is not
in `references/error-codes.md`, **stop and say so** rather than inventing names:

> The installed Beexar skill does not match the current SDK/API (`<what
> differed>`). It may be out of date — run `npx skills update`, or check
> https://docs.beexar.com/api-reference/ for the live contract — then I'll retry.

You can check without asking: the live specs are public at
`https://docs.beexar.com/api/wallet.yaml`, `/api/softswiss/gateway.yaml`,
`/api/gateway.yaml` and `/api/common/schemas.yaml`. Compare `info.version` with
the table at the top of this file.

## Hard rules

- **Never verify a signature against a re-serialised body.** The HMAC covers the
  exact bytes received. A body parser that decodes and re-encodes the JSON
  destroys them — key order, escaping and number rendering all change — and no
  signature will ever match again. Every SDK binding exists to prevent exactly
  this; read the "raw body" section of the language file.
- **Never do floating-point arithmetic on money.** Amounts and balances are
  decimal strings in the currency's main unit (`"0.90"` is ninety cents, not
  ninety). Keep them in the SDK's `Money` type. If the ledger stores integer
  minor units, convert at the boundary with the provided helper, never with
  `parseFloat` or a division.
- **Never generate your own `id_provider`.** It is the platform's idempotency
  key. Store it, return YOUR transaction id in the response.
- **Never answer HTTP 403 from a callback.** A signature failure is HTTP **400**
  with `meta.api_code` `403`. Only 400 and 500 exist on this contract.
- **Never omit `meta.balance`** on api_code `100`, `105` or `106`. Nothing
  downstream will complain; the player will simply see the wrong number.
- **Never let the SDK own idempotency.** It cannot join your database
  transaction, so the dedupe check, the tombstone check and the balance update
  must all happen in yours.
- **Never hardcode credentials**, and never ship the API secret to a browser.

Full documentation: https://docs.beexar.com
