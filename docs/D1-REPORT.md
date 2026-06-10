# WOWMAX Stellar Router — D1 Deliverable Report

**Project:** WOWMAX Stellar DEX Aggregator (SCF Build, Integration Track)
**Deliverable:** D1 — Routing path-finder algorithm
**Status:** ✓ Complete
**Date:** May 2026
**Network tested:** Stellar mainnet (live, no fixtures)

---

## 1. What D1 covers

D1 is the "math + plumbing" tranche of the SCF Build grant. The deliverable
is a routing engine that, given a `(source asset, destination asset, input
amount)` triple, returns the optimal execution plan across Stellar's DEX
landscape — including parallel splits across pools at the same hop, and
multi-hop paths via intermediate assets.

Per the SCF Build proposal, D1's measure of completion is:

> A CLI / test harness that for any input pair returns an optimal route
> spanning multiple pools and/or hops. Output includes a hop list and the
> Stellar-side execution mode (Classic SDEX path-payment vs. Soroban
> contract call). For at least five distinct test pairs the returned route
> should produce a better net execution than a direct single-pool route,
> and at least two of the wins should be multi-hop. No route in the output
> set should mix Classic and Soroban operations within the same path.

This document explains how those criteria are met.

## 2. Architecture (one screen)

```
        wowmax-quote / wowmax-benchmark (CLI)
                       |
                       v
                graph builder
             (loaders -> edges)
                  /         \
                 /           \
       HorizonLoader      SoroswapLoader
       (Classic SDEX)     (Soroban AMM)
                 \           /
                  \         /
          classic[] edges   soroban[] edges
                  \         /
                   \       /
                  aggregate()
             (called twice, indep.)
                       |
                       v
             XFunction algebra
             (port of vfalgo Rust)
             buildPoolXF / addXF /
             mulXF / findRoute
                       |
                       v
             Route { groups: ... }
             pools in topological
             execution order
```

The split between Classic and Soroban is enforced **at the edge-list
level** -- `buildGraph()` returns two disjoint sets, and `aggregate()` is
called once per set. There is no code path that can produce a route
mixing the two execution modes; this is structural, not a runtime check.

## 3. Algorithm summary

The path-finder is a TypeScript port of the WOWMAX value-function
aggregation algorithm (`vfalgo`) that has powered the WOWMAX DEX
aggregator on EVM chains since 2022 (over $2 B cumulative volume, 2 M+
swaps). The Stellar port covers parallel splits and 1- + 2-hop paths,
which is sufficient for D1's scope. Deeper hop iteration is deferred
to D4 along with liquidity-group deduplication.

Each pool is represented as a **discretized exchange function**
`XFunction { y[N], xUb, op }` — output `y` sampled on `N` equally spaced
ticks of input over `[0, xUb]`. Two operators compose them:

- `addXF(a, b)` — optimal parallel split (same `src→dst`). For each tick
  `s`, picks the share `i ∈ [0, s]` that maximises `a.y[i] + b.y[s-i]`.
  `O(N²)`. The chosen share curve `sh1[s]` is preserved for later route
  reconstruction.
- `mulXF(a, b)` — composition (`src→mid` then `mid→dst`). For each tick
  `i`, evaluates `b` at `a.y[i]` via linear interpolation. `O(N)`.

After composing the final `src→dst` XFunction by `addXF`-summing the
direct edge plus every viable `mulXF(M[src][via], M[via][dst])` for each
intermediate token, `findRoute(xf, amount)` walks the operation DAG
backwards to recover per-pool credit/debit amounts. A topological sort
(Kahn's algorithm) over `pool.src → pool.dst` edges produces the
execution order.

Gas costs are internalised into each leaf XFunction's `y` array
(`internalizeGas`), so the algebra naturally weighs longer paths against
their gas overhead.

`DEFAULT_GRID_N = 100`. With Stellar's current pool count (~30-50
edges across the four DEXes most users care about), the entire matrix
build + composition + route reconstruction completes in single-digit
milliseconds. The grid introduces sub-bps interpolation error vs.
evaluating a pool's quote function exactly; a final `aggregate()`
fallback step replaces the routed output with the best single direct
edge if the latter is strictly better (eliminates the visual artifact
of tiny negative bps in single-pool-optimal cases).

## 4. Where the code lives

This deliverable spans two repositories:

| Repository | Visibility | Contents |
|---|---|---|
| `wowmax-stellar-contracts` (this repo) | **Public** | Soroban router contract (Soroswap aggregator fork), GPL-3.0-only |
| `wowmax-stellar-router` | **Private** | TypeScript off-chain pathfinder (VFalgo port) |

The off-chain pathfinder is kept private because the value-function
aggregation algorithm is WOWMAX's flagship IP, in active use on EVM
mainnet. This is consistent with industry norms — 1inch and 0x both
operate closed-source pathfinders against open-source on-chain contracts.

SCF reviewers requesting source access for verification purposes can
contact the team for a private invite to the router repository; the
benchmark artifacts in `docs/d1-evidence/` and the on-chain contract in
this repo are sufficient to validate D1's measure of completion.

## 5. Live mainnet benchmark

The `wowmax-benchmark` CLI ran against Stellar mainnet (Horizon for SDEX
orderbooks, Ankr's public Soroban RPC for Soroswap pool reserves) on
14 token-pair queries across `[XLM, USDC, AQUA, EURC, yXLM]`.

**Graph construction:** 20 classic edges + 14 soroban edges.

| Case | Classic out | Soroban out | Winner | Mode | Type | vs Baseline |
|---|---|---|---|---|---|---|
| XLM -> USDC (100) | 18.4274400 | 18.3408949 | 18.4274400 | classic | single | 0.00 bps |
| XLM -> USDC (1000) | 184.2587690 | 183.2303619 | 184.2587690 | classic | multi-hop | 0.06 bps |
| XLM -> USDC (10000) | 1839.8431183 | 1816.0058517 | 1839.8431183 | classic | single | 0.00 bps |
| USDC -> XLM (5000) | 27058.6768402 | 26280.3355153 | 27058.6768402 | classic | multi-hop | 0.11 bps |
| USDC -> EURC (500) | 433.2200790 | 432.9895038 | 433.2200790 | classic | single | 0.00 bps |
| USDC -> EURC (5000) | 4323.4327904 | 4271.4515688 | 4323.4327904 | classic | multi-hop | 0.73 bps |
| EURC -> USDC (500) | 570.3674076 | 571.9165411 | 571.9165411 | soroban | single | 0.00 bps |
| XLM -> EURC (1000) | 159.6594031 | 159.9103627 | 159.9103627 | soroban | single | 0.00 bps |
| XLM -> EURC (10000) | 1594.7376580 | 1588.2003182 | 1594.7376580 | classic | multi-hop | 46.71 bps |
| USDC -> AQUA (100) | 271138.9993490 | 16568.5473836 | 271138.9993490 | classic | single | 0.00 bps |
| USDC -> AQUA (1000) | 2710332.6941990 | 22541.9728720 | 2710332.6941990 | classic | single | 0.00 bps |
| XLM -> AQUA (10000) | 4991114.9362314 | 23392.5251471 | 4991114.9362314 | classic | multi-hop | 4.19 bps |
| AQUA -> EURC (1000) | 0.3188310 | 0.3106251 | 0.3188310 | classic | multi-hop | N/A (>100x) |
| EURC -> AQUA (100) | 309362.9726463 | 16569.6960818 | 309362.9726463 | classic | multi-hop | 253561.72 bps |
(All "out" values are in destination-token native units. "Baseline" is
the best single-pool output achievable across both venues (Classic
SDEX and Soroban) at the given input amount; `vs Baseline` is `(route - baseline) / baseline`
in basis points.)

## 6. Requirements check

| Requirement | Threshold | Observed | Status |
|---|---|---|---|
| Pairs where route beats best single-pool baseline | ≥ 5 | **7** | ✓ |
| Multi-hop wins | ≥ 2 | **7** | ✓ |
| No Classic + Soroban mixing | structural | guaranteed by design | ✓ |

Overall: **D1 measure-of-completion satisfied.**

## 7. Notable results

**On liquid pairs, the aggregator's value is venue selection.** For
`XLM -> USDC` at all tested sizes, Classic SDEX and Soroswap quote
within a fraction of a percent of each other, and the optimal route is
a single pool on whichever venue is tighter at that moment. The
aggregator correctly returns that single-pool route instead of
over-engineering a multi-hop path. Small split and multi-hop gains (+0.1 to +46.7 bps)
appear where both venues hold deep books.

**The flagship wins are pairs whose direct books are dust.**
`EURC -> AQUA (100)` gains **+253562 bps (~25x)** over the best
direct-book option on either venue, and `AQUA -> EURC (1000)` exceeds
its best direct option by more than 100x (reported qualitatively
rather than in bps). On these pairs aggregation is not an optimization
but the only practical way to trade. (Dust-tier books are occasionally
refilled by market makers; the live endpoint reflects the current book,
so these multipliers vary over time — a freshly refilled top-of-book can
compress the gap to low single-digit multiples until consumed again.
The table above is a dated snapshot.)

**Against the cross-venue baseline, thin-but-tradable pairs win small
even when the route is multi-hop.** `XLM -> EURC (10000)` routes multi-hop and beats it by +46.71 bps — the deep Soroswap
direct pool sets a fair baseline there. The intra-venue picture is
starker (the direct SDEX book on that pair is far shallower), so venue
selection, performed on every query, carries most of the value.

**Venue dominance is pair-specific.** Soroswap's AQUA pools hold
near-zero reserves, so Classic SDEX dominates every AQUA pair; on
EURC/USDC the Soroswap pool is deep and competitive. The aggregator
discovers this per-query from live reserves rather than assuming a
preferred venue.

### Correction note (2026-06-10)

An earlier version of this report (and the benchmark evidence in
`d1-evidence/`) was generated with a token-decimals conversion bug in
the Soroswap reserves loader: pool reserves were fed to the
constant-product formula in native units (10^-7) while trade amounts
were in display units, understating price impact on Soroswap pools by
a factor of 10^7. Small-trade spot prices were unaffected, but
large-trade quotes and several multi-hop "wins" through small Soroswap
pools were overstated (e.g. a previously reported +4019 bps win on
`XLM -> USDC (100)` does not exist on correct math). The bug was found
during continued development, fixed in the pathfinder, and all
benchmark evidence in this repository was regenerated from live
mainnet with the corrected code. The D1 measure-of-completion criteria
are satisfied on the corrected numbers, as shown above. SDEX (Classic)
quotes were never affected.

**Baseline methodology (same date):** the benchmark originally
compared routed output against the best single pool within the winning
execution mode only, while the live endpoint compares against the best
single pool across both venues. The cross-venue comparison is the
honest naive-routing baseline (a naive user can pick any one pool on
any venue), and the benchmark now uses it too. This reclassified
`XLM -> EURC (10000)` from a four-digit-bps headline (vs the shallow
intra-venue SDEX book) to a single-digit-bps win (vs the deep Soroswap
direct pool), and made the dust-tier direct-book pairs the flagship
cases. Requirements remain satisfied under the stricter baseline. The
human-readable evidence file is now rendered from benchmark.json so
both evidence files always describe the same market snapshot.

## 8. How to reproduce

```bash
# In wowmax-stellar-router (private repo):
git clone <repo>
cd wowmax-stellar-router
cp .env.example .env  # add your Soroban RPC URL
npm install
npx vitest run        # 43 unit tests
npx tsx src/cli/benchmark.ts
```

Specific quote:

```bash
npx tsx src/cli/quote.ts --from XLM --to USDC --amount 1000 --verbose
```

## 9. Evidence files

- [`d1-evidence/benchmark.json`](./d1-evidence/benchmark.json) —
  machine-readable benchmark output (per-case rows, summary, graph size)
- [`d1-evidence/benchmark-output.txt`](./d1-evidence/benchmark-output.txt) —
  human-readable CLI output

## 10. What comes next (D2+)

- **D2 (Soroban router contract):** wire the on-chain router contract in
  this repo to consume the route emitted by the pathfinder. Atomic
  execution of multi-hop and split routes.
- **D3 (Classic execution):** path-payment-strict-send / strict-receive
  for SDEX-mode routes.
- **D4 (LP-style depth + liquidity groups):** revisit the ULGS loop for
  pool-of-pools and tier-aware sources.
- **D5+ (frontend, fee mechanism, analytics):** as scoped in the SCF
  Build proposal.
