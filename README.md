# wowmax-stellar-contracts

> Soroban smart contracts powering the [WOWMAX](https://wowmax.exchange)
> Stellar DEX aggregator.

Part of the WOWMAX deliverable for the [Stellar Community Fund (SCF) Build
Award](https://communityfund.stellar.org/), Integration Track.

**Status:** D1 (path-finder algorithm) complete. See
[`docs/D1-REPORT.md`](./docs/D1-REPORT.md) for the deliverable report and
[`docs/d1-evidence/`](./docs/d1-evidence/) for live-mainnet benchmark
artifacts.

---

## What this is

WOWMAX is a DEX aggregator that has executed over $2 billion in cumulative
swap volume across 20+ EVM chains since 2022. This repository contains the
**on-chain** half of the WOWMAX deployment on Stellar:

- a Soroban **router contract** that atomically executes split + multi-hop
  routes across multiple Stellar DEX protocols
- **protocol adapters** for Soroswap, Phoenix, Aquarius, and CometDEX

The **off-chain** half -- the routing algorithm (a TypeScript port of the
WOWMAX value-function aggregator that powers WOWMAX on EVM) -- lives in a
separate, private repository for IP reasons. This is consistent with how
1inch and 0x operate: open on-chain contracts, closed-source pathfinders.
SCF reviewers can request access to the router repo for verification
purposes; the public benchmark evidence in `docs/d1-evidence/` reproduces
the deliverable criteria end-to-end against live Stellar mainnet.

## Architecture

```
+------------------+      +-------------------+
|   off-chain      |      |   on-chain        |
|   pathfinder     | ---> |   router contract |
| (private repo)   |      |   (this repo)     |
+------------------+      +---------+---------+
                                    |
                          +---------+---------+
                          |                   |
                  +-------v------+   +--------v-------+
                  |  Soroswap    |   |   Phoenix      |
                  |  adapter     |   |   adapter      |
                  +--------------+   +----------------+
                          |                   |
                  +-------v------+   +--------v-------+
                  |   Aquarius   |   |   CometDEX     |
                  |   adapter    |   |   adapter      |
                  +--------------+   +----------------+
```

- **router** -- main entry point. Receives swap requests with a distribution
  across protocols and dispatches each leg to the appropriate adapter.
- **adapters/soroswap** -- adapter for Soroswap (UniswapV2-style AMM).
- **adapters/phoenix** -- adapter for Phoenix multihop router.
- **adapters/aqua** -- adapter for Aquarius liquidity pool router.
- **adapters/comet** -- adapter for CometDEX (BalancerV1-style multi-asset
  pools).
- **deployer** -- helper contract for deploying and initializing adapters.

The off-chain pathfinder produces split + multi-hop routes by composing a
discretized **value-function algebra** over pool quote functions. See
[`docs/D1-REPORT.md`](./docs/D1-REPORT.md) section 3 for the algorithm
summary. The router contract simply executes the route emitted by the
pathfinder -- splitting / multi-hopping is decided off-chain and executed
atomically on-chain.

## Deliverable roadmap

| Tranche | Scope | Status |
|---|---|---|
| **D1** | Routing path-finder algorithm | **complete** |
| D2 | Soroban router contract: atomic split + multi-hop execution | in progress |
| D3 | Classic SDEX path-payment execution | scheduled |
| D4 | Liquidity-group dedup, deeper hop iteration | scheduled |
| D5 | Frontend integration (app.wowmax.exchange) | scheduled |
| D6+ | Fee mechanism, analytics, observability | scheduled |

## D1 deliverable evidence

- [D1-REPORT.md](./docs/D1-REPORT.md) -- full deliverable report
- [benchmark.json](./docs/d1-evidence/benchmark.json) -- machine-readable
  benchmark output from live Stellar mainnet
- [benchmark-output.txt](./docs/d1-evidence/benchmark-output.txt) --
  human-readable CLI output

**Headline result:** 14 token-pair test queries against mainnet, 6 beat
single-pool baseline (D1 threshold: >= 5), 7 multi-hop wins (D1 threshold:
>= 2), zero routes mix Classic and Soroban execution modes (enforced
structurally). Best case `XLM -> USDC (100)` at +4019 bps via multi-hop.
Full table in the D1 report.

## Building
Produces optimized WASM artifacts in
`contracts/target/wasm32-unknown-unknown/release/`.

## Deployed addresses

- **Mainnet:** see [`public/mainnet.contracts.json`](./public/mainnet.contracts.json)
  (populated after deploy)
- **Testnet:** see [`public/testnet.contracts.json`](./public/testnet.contracts.json)
  (populated after deploy)

## About WOWMAX

[WOWMAX](https://wowmax.exchange) is a non-custodial DEX aggregator and
copy-trading platform. As of mid-2026:

- $2 B+ cumulative swap volume
- 336 K+ unique traders
- 2 M+ swaps executed
- 20+ EVM chains supported
- Stellar integration adds the first non-EVM chain to the WOWMAX network

## License

GPL-3.0. See [LICENSE](./LICENSE).

## Attribution

This project is a derivative work of
[soroswap/aggregator](https://github.com/soroswap/aggregator), licensed
under GPL-3.0. See [NOTICE](./NOTICE) for details. Significant modifications
were made to fit the WOWMAX routing model; the value-function algorithm
itself originates from WOWMAX's EVM aggregator and is not inherited from
the Soroswap codebase.
