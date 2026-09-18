# Python

```bash
pip install beexar
```

Zero runtime dependencies — standard library only. Python 3.9+.
Source: https://github.com/beexar-games/beexar-python

## Launcher

```python
from beexar import Client

beexar = Client(os.environ["BEEXAR_CASINO_ID"], os.environ["BEEXAR_API_SECRET"])

launch_url = beexar.launch_real({
    "game": "dice",
    "account": {"id": "player_123", "currency": "EUR"},
    "locale": "en",
})
```

Also `beexar.launch_demo({...})` and `beexar.list_games()`.

## Wallet callbacks

Implement four methods; the SDK does signature, parsing, validation and the
error envelope. Any object with the right method names works — there is no base
class to inherit.

```python
from beexar import BetWinResult, BetWinTransactionResult, WalletError, WalletServer

class Wallet:
    def balance(self, request, context): ...
    def bet_win(self, request, context): ...   # one DB transaction — see wallet-callbacks.md
    def rollback(self, request, context): ...
    def finish(self, request, context): ...

server = WalletServer(Wallet(), os.environ["BEEXAR_API_SECRET"])
```

Request objects are frozen dataclasses with the wire field names
(`request.account_id`, `t.id_provider`, `request.round_id_provider`). Amounts
arrive as `Money`.

## Mounting

```python
from beexar.asgi import fastapi_router, flask_blueprint, asgi_app

app.include_router(fastapi_router(server, prefix="/wallet"))         # FastAPI
app.register_blueprint(flask_blueprint(server, url_prefix="/wallet"))  # Flask
application = asgi_app(server)                                        # bare ASGI
```

## The raw body — the one thing to get right

Declaring a Pydantic model on a FastAPI route makes Starlette parse the body and
you never see the bytes; `request.get_data(as_text=True)` in Flask re-decodes
them. The bindings above avoid both. `dispatch()` refuses a `str` outright — a
decode is not always reversible.

## Errors

```python
raise WalletError.insufficient_funds(current_balance)  # 400 / api_code 100
raise WalletError.already_rolled_back()                # 400 / api_code 409
raise WalletError.invalid_player()                     # 400 / api_code 101
```

`insufficient_funds`, `bet_limit_reached` and `max_bet_exceeded` take the
balance as a **required** argument. Anything else you raise becomes an opaque 500.

## Money

```python
str(Money.parse("100.00").sub(Money.parse("0.30")))   # "99.70"
Money.from_minor_units(9970, 2)                       # from an integer ledger
```

`Money` raises on `float()`, on `+` and on `-`, so a number cannot leak in.

## Reference implementation

`examples/inmemory_wallet.py` in the repository. `examples/server.py` is a
runnable framework-free server.

## Build check

`python -m pytest`
