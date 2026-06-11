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
| XLM -> USDC (100) | 18.3475000 | 18.3010816 | 18.3475000 | classic | single | 0.00 bps |
| XLM -> USDC (1000) | 183.4750000 | 182.5177106 | 183.4750000 | classic | single | 0.00 bps |
| XLM -> USDC (10000) | 1834.6918420 | 1806.5750346 | 1834.6918420 | classic | single | 0.00 bps |
| USDC -> XLM (5000) | 27192.0129731 | 26419.1944038 | 27192.0129731 | classic | single | 0.00 bps |
| USDC -> EURC (500) | 432.7867255 | 432.9709658 | 432.9709658 | soroban | single | 0.00 bps |
| USDC -> EURC (5000) | 4319.6514116 | 4269.4013044 | 4319.6514116 | classic | multi-hop | 3.31 bps |
| EURC -> USDC (500) | 572.1580273 | 571.8677959 | 572.1580273 | classic | single | 0.00 bps |
| XLM -> EURC (1000) | 159.0050000 | 158.8081036 | 159.0050000 | classic | single | 0.00 bps |
| XLM -> EURC (10000) | 1588.0355021 | 1578.4793970 | 1588.0355021 | classic | multi-hop | 72.23 bps |
| USDC -> AQUA (100) | 271437.8837112 | 39109.7877300 | 271437.8837112 | classic | multi-hop | 5.08 bps |
| USDC -> AQUA (1000) | 2709750.8883385 | 45028.6873996 | 2709750.8883385 | classic | multi-hop | 140.72 bps |
| XLM -> AQUA (10000) | 4980169.2975801 | 24586.1582016 | 4980169.2975801 | classic | single | 0.00 bps |
| AQUA -> EURC (1000) | 0.3155320 | 0.3157155 | 0.3157155 | soroban | multi-hop | N/A (>100x) |
| EURC -> AQUA (100) | 310667.9457267 | 40759.9228288 | 310667.9457267 | classic | multi-hop | 254673.49 bps |
| XLM -> VELO (10000) | 541076.3313858 | — | 541076.3313858 | classic | multi-hop | 90.58 bps |
| XRP -> USDC (500) | 548.7640575 | 0.0005517 | 548.7640575 | classic | multi-hop | 2.82 bps |
| XLM -> USDC (100000) | 18281.9072956 | 16594.9162351 | 18281.9072956 | classic | multi-hop | 0.98 bps |
| VELO -> EURC (600000) | 1716.7297781 | — | 1716.7297781 | classic | multi-hop | N/A (>100x) |
(All "out" values are in destination-token native units. "Baseline" is
the best single-pool output achievable across both venues (Classic
SDEX and Soroban) at the given input amount; `vs Baseline` is `(route - baseline) / baseline`
in basis points.)

## 6. Requirements check

| Requirement | Threshold | Observed | Status |
|---|---|---|---|
| Pairs where route beats best single-pool baseline | ≥ 5 | **10** | ✓ |
| Multi-hop wins | ≥ 2 | **10** | ✓ |
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

**Classic-side data and composite fixes (same date, third correction):**
continued verification against raw Horizon books uncovered three more
bugs in the (private) pathfinder's classic side, all fixed and the
evidence regenerated once more. (1) Bid-amount convention: Horizon
denominates order amounts in the asset the maker gives away, so
bid.amount is the maker's counter budget (our output cap), not input
capacity; the loader treated it as input, understating books with
price < 1 (XLM/USDC appeared ~5.4x too shallow) and inflating books
with price > 1 by roughly the price factor (a phantom near-zero-
slippage XLM->AQUA path). Established empirically by a mirror test:
the same physical order carries the same amount in both book
orientations. (2) Orderbook depth: the loader fetched 50 levels;
large trades walked off the visible book and the remainder was
silently dropped. Now 200 (Horizon's cap). (3) Composite sampling:
two-hop functions sampled their second leg over the query amount in
the wrong units, wasting most of the grid and understating multi-hop
routes; legs are now sampled over their reachable input range.
Net effect on this table: direct quotes on liquid pairs are larger
(real depth visible), the AQUA direct quotes are honest (phantom
inflation removed), and composite wins in the tens of bps appeared
(USDC -> AQUA (100) +55.6 bps; USDC -> AQUA (1000) +16.9 bps) that the previous code could not represent.
Requirements hold (8 beats / 8 multi-hop). Each boundary
now has a live regression probe in the private repo (decimals_probe
for AMM reserves, sdex_probe for orderbook conventions).

**Proxy tokens and expanded case set (same date, fourth update):**
the intermediate-token list was rebuilt from data instead of intuition.
A trade-driven scan (2400 recent trades via Horizon, then a bilateral
order-book probe of the top-traded assets against BOTH XLM and USDC,
with executable-depth, spread and cross-rate sanity gates) confirmed
the existing intermediates AQUA and yXLM and added two new proxy
tokens: VELO and the XRP anchor. The routing graph grew from 20 to 42
classic edges with no latency cost (pair fetches were already
parallel). Three cases were added to the benchmark to cover the new
transit topology — including a 100,000 XLM trade that the optimizer
splits between the direct book and an XLM->XRP->USDC leg, the exact
proxy-transit pattern the feature targets. Numbers in the table above
are a fresh frozen snapshot on the expanded graph; thresholds hold
with margin (9 of 17 beat baseline, 9 multi-hop wins).

**Three-hop routing and a 600,000 VELO showcase (2026-06-11, fifth
update):** the router gained a third hop — src -> via1 -> via2 -> dst
compositions on top of the direct and single-via families, with a
progressive cycle-safe fallback (the full candidate set, then the
better-yielding ordering of each unordered via pair, then two-hop
only). On Stellar's hub-and-spoke liquidity this serves pairs whose
deep books live on DIFFERENT hubs; the benchmark gained a
VELO -> EURC (600,000) case targeting exactly that topology (VELO is
deep against XLM and near-dust against EURC directly, while EURC's
depth sits against USDC). The same work surfaced and fixed a latent
grid-boundary interpolation bug: float64 narrowing of the tick
position could index one slot past the grid and silently drop flow in
rare boundary cases. This fifth snapshot is frozen on the 3-hop
router: 10 of 18 cases beat the
single-pool baseline, 10 multi-hop wins. Two stale
prose figures left behind by the 0ef137c fix are also corrected: the
previous update's closing parenthetical now carries the numbers that
snapshot actually froze (9 of 17, not 7 of 14), and the README query
count matches the case set.

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
