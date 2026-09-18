# Beexar public API

Everything you need to integrate the Beexar game platform: the OpenAPI
contract, four official SDKs, an agent skill, and the conformance fixtures the
SDKs are tested against.

**Documentation: https://docs.beexar.com**

---

## Integrate with an AI agent

```bash
npx skills add beexar-games/public-api
```

Installs the `beexar-integration` skill into Claude Code, Cursor, Codex, OpenCode
and ~80 other agents. It knows the contract, picks the right SDK for your
codebase, and — unlike a hand-written guide — it is **generated from the
platform's own source**, so its api_code table and timing budgets cannot drift
from what the platform actually does.

Prefer to paste a prompt? See https://docs.beexar.com/tools/prompts/

## Integrate with an SDK

| Language | Install | Repository |
|---|---|---|
| Node / TypeScript | `npm install @beexar/sdk` | [beexar-node](https://github.com/beexar-games/beexar-node) |
| PHP | `composer require beexar/sdk` | [beexar-php](https://github.com/beexar-games/beexar-php) |
| Go | `go get github.com/beexar-games/beexar-go` | [beexar-go](https://github.com/beexar-games/beexar-go) |
| Python | `pip install beexar` | [beexar-python](https://github.com/beexar-games/beexar-python) |

Each one gives you the same two halves: a signed client for launching games, and
a wallet server where you implement four methods against your own ledger and the
SDK handles signature verification, parsing, validation, decimal money and the
error envelope. All four ship a complete reference wallet you can read in one
sitting.

Every SDK has **zero runtime dependencies**.

## Integrate without an SDK

The contract is small and the specs are here:

| Spec | What it describes |
|---|---|
| [`openapi/providers/softswiss/wallet.yaml`](openapi/providers/softswiss/wallet.yaml) | the four callbacks **you** implement |
| [`openapi/softswiss/gateway.yaml`](openapi/softswiss/gateway.yaml) | launching a real or demo session |
| [`openapi/gateway.yaml`](openapi/gateway.yaml) | the operator game catalogue |
| [`openapi/common/schemas.yaml`](openapi/common/schemas.yaml) | shared schemas |

Generate a client with your ecosystem's OpenAPI generator, then read
[`skill/beexar-integration/references/manual.md`](skill/beexar-integration/references/manual.md)
for the parts a generator will not give you — raw-body signing, idempotency,
tombstones and the money rules.

Rendered reference, with a try-it console: https://docs.beexar.com/api-reference/

## Check your implementation

`conformance/` holds the fixtures all four SDKs run: the exact request bytes,
the signature over them, the ledger before and after, and the expected response.
They are language-neutral — point your own tests at them and you get the same
coverage the official SDKs have.

Then run the **Integration Test Game**: 29 scenarios against your live
callbacks, launched with `game: "testgame"`.
https://docs.beexar.com/guides/testing/

## Versioning

One version number covers all four SDKs — the same number always means the same
contract snapshot. `CONTRACT_VERSIONS.json` records which version of each
OpenAPI document a release was built from, and each release lists the SHA-256 of
all four specs.

## License

MIT
