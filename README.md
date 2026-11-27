# liquid-plugin

`assetchan` is the plugin half of Lightning asset channels on Liquid: channels
whose funds are a Liquid issued asset instead of L-BTC. The
[Core Lightning fork](https://github.com/MarkCherepovskyi/lightning) knows a
channel's asset and builds and signs its transactions. This plugin does the
rest, with an `elementsd` wallet that holds the assets.

> **Status: MVP, not production ready.** This is open-source code for
> testnets. It has not been audited, it has known open issues, and it must not
> be used with funds that matter.

## What it does

- **Turns asset channels on.** Core only takes them when a plugin offers their
  feature bit, and the plugin decides which assets (`assetchan-asset`).
- **Opens channels.** `assetchan-open` funds a channel from the `elementsd`
  wallet: the asset to the channel's 2-of-2, and an L-BTC fee reserve next to
  it.
- **Polices incoming offers.** The `openchannel` hook checks the asset and the
  fee reserve, and makes our side of a channel close to an address of the
  `elementsd` wallet.
- **Keeps a payment in its asset.** `assetchan-invoice` ties an invoice to an
  asset. The `htlc_accepted` hook refuses payments that arrive in another
  asset and forwards that would leave in another one. `assetchan-pay` pays
  only over channels in the invoice's asset.
- **Asset gossip.** Asset channels are never announced in BOLT 7 gossip, so
  plugins tell each other about them in a custom message.

After a unilateral close, core sweeps the asset outputs itself and pays the
fees with L-BTC from CLN's own wallet, so keep some L-BTC there.

## Requirements

- The Core Lightning fork linked above
- `elementsd` with a wallet, reachable over JSON-RPC
- Python 3 with `pyln-client`

## Run

The plugin must run as an important plugin and refuses to start otherwise:

```
lightningd --network=liquid-testnet \
    --important-plugin=/path/to/plugin/assetchan/assetchan.py \
    --assetchan-asset=<asset id>
```

`plugin/assetchan/nodes.sh` starts and stops a set of test nodes on Liquid
testnet or Liquid regtest; its header lists the commands and the environment
it reads.

| Option | Default | Meaning |
|---|---|---|
| `assetchan-asset` | none | Liquid asset id we open and accept asset channels in. Repeatable. |
| `assetchan-min-fee-reserve` | 1000 | Smallest L-BTC fee reserve (sat) we accept an asset channel with. |
| `assetchan-max-fee-reserve` | 10000 | Largest L-BTC fee reserve (sat) we accept an asset channel with. |
| `assetchan-default-fee-reserve` | 2000 | L-BTC fee reserve (sat) `assetchan-open` puts up by default. |
| `assetchan-gossip-interval` | 300 | Seconds between two rounds of asset gossip. |
| `assetchan-close-to` | `elements` | Where our asset goes when a channel closes: a fresh address of the `elementsd` wallet (`elements`), or whatever core picks (`none`). |
| `assetchan-elements-rpc` | from the `bitcoin-rpc*` settings | `elementsd`'s JSON-RPC as `http://user:password@host:port`. |
| `assetchan-elements-wallet` | default wallet | The `elementsd` wallet to use. |

Amounts are in the asset's base units; on the wire one unit is 1000 msat.

## RPC

| Method | What it does |
|---|---|
| `assetchan-open` | Opens an asset channel to a connected peer, funded by the `elementsd` wallet |
| `assetchan-close` | Mutually closes a channel; our asset goes to the close address |
| `assetchan-list` | Lists our asset channels |
| `assetchan-funds` | Shows what the `elementsd` wallet holds |
| `assetchan-send` | Sends units of an asset from the `elementsd` wallet to an address |
| `assetchan-newaddr` | Returns a fresh unconfidential address of the `elementsd` wallet |
| `assetchan-invoice` | Creates an invoice that must be paid in a given asset |
| `assetchan-pay` | Pays an invoice in a given asset |
| `assetchan-layer` | Returns the askrene layer that keeps a payment in one asset |
| `assetchan-listchannels` | Lists gossiped channels with the asset each is in |
| `assetchan-listgossip` | Lists the asset channels known from asset gossip |

## Documentation

- `plugin/assetchan/DESIGN.md`: the design of asset channels, core and plugin
- `plugin/assetchan/TESTNET.md`: the test cases run by hand on Liquid testnet
- `plugin/assetchan/ETH.md`: the earlier plan for channels in ETH; the implementation is in evm-plugin
- `docs/Liquid/`: the channel lifecycle, the transactions and the plugin in detail

Some of these documents still use the paths the code had inside the CLN tree
(`contrib/assetchan`).

## Tests

`plugin/assetchan/tests/test_assetchan.py` uses Core Lightning's
`pyln-testing` fixtures on Liquid regtest and needs a build of the CLN fork.
`plugin/assetchan/run-tests.sh` runs it in Docker; it was written for the
layout inside a CLN checkout. The developer edition runs the suite with
`./dev test liquid`.

## Related repositories

- [Core Lightning fork](https://github.com/MarkCherepovskyi/lightning) with the asset-channel core changes
- [swap-plugin](https://github.com/liquid-lightning/swap-plugin) and [pricer-plugin](https://github.com/liquid-lightning/pricer-plugin), for cross-asset payments
- [evm-plugin](https://github.com/liquid-lightning/evm-plugin) and [utxo-contract](https://github.com/liquid-lightning/utxo-contract), for channels on EVM chains
- [dashboard](https://github.com/liquid-lightning/dashboard), a web page for the nodes

## Licence

MIT, see [LICENSE](LICENSE).
