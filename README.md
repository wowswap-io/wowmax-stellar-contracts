# wowmax-stellar-contracts

Soroban smart contracts powering the WOWMAX Stellar DEX aggregator: a router
contract and protocol adapters for Soroswap, Phoenix, Aquarius, and CometDEX.

Built and deployed as part of the WOWMAX Stellar Community Fund (SCF) Build
Award — Integration Track.

## Architecture

- **router** — main entry point. Receives swap requests with a distribution
  across protocols and dispatches each leg to the appropriate adapter.
- **adapters/soroswap** — adapter for Soroswap (UniswapV2-style AMM).
- **adapters/phoenix** — adapter for Phoenix multihop router.
- **adapters/aqua** — adapter for Aquarius liquidity pool router.
- **adapters/comet** — adapter for CometDEX (BalancerV1-style multi-asset pools).
- **deployer** — helper contract for deploying and initializing adapters.

## License

GPL-3.0. See LICENSE.

## Attribution

This project is a derivative work of soroswap/aggregator
(https://github.com/soroswap/aggregator), licensed under GPL-3.0. See NOTICE
for details.

## Building

```bash
cd contracts
make build
```

Produces optimized WASM in `contracts/target/wasm32-unknown-unknown/release/`.

## Deployed addresses

- **Mainnet**: see `public/mainnet.contracts.json` (populated after deploy)
- **Testnet**: see `public/testnet.contracts.json` (populated after deploy)
