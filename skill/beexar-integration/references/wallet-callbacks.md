# The four callbacks

Read this before writing a handler. Everything here is about money, and all of
it is testable.

## The endpoints

| Route | What it does | Body carries |
|---|---|---|
| `POST /balance` | read the player's balance | `account_id`, `currency`, `game_id` |
| `POST /betwin` | debit and/or credit, atomically | `round_id`, `transactions[]` of `{id_provider, type: bet\|win, amount}` |
| `POST /rollback` | reverse earlier transactions | `round_id_provider`, `transactions[]` of `{id_provider, original_id_provider}` |
| `POST /finish` | close the round | `round_id` — no `game_id` |

Two naming traps, both worth a test:

- rollback uses **`round_id_provider`**; betwin and finish use **`round_id`**.
- `/finish` has **no `game_id`** at all.

## Money

Every amount and balance is a **decimal string in the currency's main unit**.
`"0.90"` is ninety cents. Not cents-as-integer, not a JSON number.

- Keep them in the SDK's `Money` type end to end.
- If your ledger stores integer minor units, convert at the boundary with
  `fromMinorUnits` / `from_minor_units` / `MoneyFromMinorUnits`.
- `bonus_amount` is **required in every `/betwin` response transaction**. Send
  `"0.00"` when no bonus balance moved; the SDKs default it for you.

## Idempotency — the rule everything else rests on

`id_provider` is the platform's transaction id and its idempotency key. The
platform retries, so the same `id_provider` will arrive twice, and the second
time must be invisible.

**A repeat returns what you stored the first time** — your transaction id and
the balance you reported then. Not today's balance. Not a fresh id.

```
lookup(id_provider)  ──found──▶  return stored id + stored balance, change nothing
      │
   not found
      ▼
tombstone(id_provider)?  ──yes──▶  api_code 409, change nothing
      │
      no
      ▼
apply, record {id_provider → your id, balance after}
```

The lookup and the balance update **must be in one database transaction**, with
`id_provider` under a unique index. Two transactions is a race that looks
correct in development and double-credits in production. This is why no Beexar
SDK offers to do idempotency for you: it cannot join your transaction, so it
could only pretend to.

## Atomicity

`/betwin` can carry several transactions. Apply them **in the order given**, and
**all or none**. If the third cannot be paid, the first two must not have
happened — and the `insufficient funds` answer reports the balance as it was
*before* the batch, because nothing moved.

## Tombstones and out-of-order delivery

A rollback can arrive **before** the transaction it reverses. That is normal,
not an error.

- On `/rollback`, record a tombstone for `original_id_provider` **whether or not
  you have ever seen it**. If you have, reverse it too.
- On `/betwin`, check the tombstone after the dedupe check. A transaction whose
  `id_provider` is tombstoned must be **refused with api_code `409`**, never
  applied.
- Reversing a win can only take back what is there — clamp the result at zero
  rather than letting a player go negative on your bookkeeping.
- Answer `200` on a rollback even when there was nothing to reverse, with an
  empty `id` for that transaction. Any other answer makes the platform retry
  forever.

## `finished`

`/betwin` may carry `finished: true`, which closes the round then and there.
When it does not, `/finish` arrives separately. Both must be idempotent;
`/finish` simply answers the current balance.

## Errors

See `error-codes.md`. The short version: two HTTP statuses (400 and 500), two
`code` values, and all meaning in `meta.api_code`. Codes `100`, `105` and `106`
must carry `meta.balance`.

## Time

See `timings.md`. The short version: answer fast, expect 5xx and timeouts to be
retried with the same `id_provider`, and know that insufficient funds and bad
signatures are never retried.

## Proving it

Launch the Integration Test Game against your live callbacks with
`game: "testgame"` and get 29 of 29. Every rule on this page is one of those
tests. https://docs.beexar.com/guides/testing/
