# Go

```bash
go get github.com/beexar-games/beexar-go
```

Zero dependencies — standard library only, in the SDK and in its tests. Go 1.26+.
Source: https://github.com/beexar-games/beexar-go

## Launcher

```go
client, err := beexar.NewClient(os.Getenv("BEEXAR_CASINO_ID"), os.Getenv("BEEXAR_API_SECRET"))

res, err := client.LaunchReal(ctx, beexar.LaunchRealRequest{
    Game:    "dice",
    Account: beexar.LaunchAccount{ID: "player_123", Currency: "EUR"},
    Locale:  "en",
})
// res.LaunchURL
```

Also `client.LaunchDemo(ctx, …)` and `client.ListGames(ctx, beexar.ListGamesQuery{})`.

## Wallet callbacks

Implement `beexar.WalletHandler`; the SDK does signature, parsing, validation
and the error envelope.

```go
type Wallet struct{ db *sql.DB }

func (w *Wallet) BetWin(ctx context.Context, req beexar.BetWinRequest, _ beexar.RequestContext) (beexar.BetWinResult, error) {
    // one DB transaction — see wallet-callbacks.md
}
// Balance, Rollback, Finish

server, err := beexar.NewWalletServer(&Wallet{db}, os.Getenv("BEEXAR_API_SECRET"))
mux.Handle("/wallet/", server.NewHandler())
```

Struct fields are Go-cased (`req.AccountID`, `t.IDProvider`,
`req.RoundIDProvider`) while the wire stays snake_case. Amounts arrive as
`beexar.Money`.

## The raw body — the one thing to get right

`server.NewHandler()` reads the body once and hands the same bytes to
`Dispatch`. Wiring it yourself:

```go
body, _ := io.ReadAll(io.LimitReader(r.Body, beexar.MaxBodyBytes+1))
out := server.Dispatch(r.Context(), beexar.RouteBetWin, body, r.Header.Get(beexar.SignatureHeader), nil)
```

Never decode and re-encode the JSON before verifying: the bytes the signature
covers are then gone. `beexar.WithWarning(fn)` makes the SDK tell you when it
can see that this happened.

## Errors

```go
return beexar.ErrInsufficientFunds(currentBalance) // 400 / api_code 100
return beexar.ErrAlreadyRolledBack()               // 400 / api_code 409
return beexar.ErrInvalidPlayer("")                 // 400 / api_code 101
```

`ErrInsufficientFunds`, `ErrBetLimitReached` and `ErrMaxBetExceeded` take the
balance as a **required** parameter. Any other error becomes an opaque 500.

## Money

```go
beexar.MustParseMoney("100.00").Sub(beexar.MustParseMoney("0.30")).String() // "99.70"
beexar.MoneyFromMinorUnits(big.NewInt(9970), 2)                            // from an integer ledger
```

`Money` marshals to a JSON string and has no float constructor.

## Freespins (optional)

Read `freespins.md` first.

```go
levels, err := client.BetLevels(ctx, beexar.BetLevelsRequest{Games: []string{"dice"}}) // []beexar.BetLevelData
err = client.IssueFreespins(ctx, beexar.FreespinsIssueRequest{
    IssueID: "fs-123", Games: []string{"dice"}, FreespinsQuantity: 10, BetLevel: 1,
    ValidUntil: time.Now().UTC().Add(7 * 24 * time.Hour).Format(time.RFC3339),
    Account:    beexar.LaunchAccount{ID: "player_123", Currency: "EUR"},
})
err = client.CancelFreespins(ctx, beexar.FreespinsCancelRequest{IssueID: "fs-123"})
```

The callback is the optional `beexar.FreespinsFinishHandler` — a
`FreespinsFinish(ctx, req, rc)` method on the same value you pass to
`NewWalletServer`; `req.IssueID` and `req.Amount` (a `beexar.Money`, possibly
zero). Refuse with `beexar.ErrInvalidFreespinsIssue("")` (610) or
`beexar.ErrFreespinsStateTransition("")` (605). `NewHandler` routes
`/freespins/finish`; without the method it answers api_code 404.

## Reference implementation

`examples/inmemory` in the repository — a complete, correct wallet, freespins
result included. `examples/server` is a runnable server.

## Build check

`go build ./... && go test -race ./...`
