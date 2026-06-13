# Mainnet Transaction Evidence

All transactions executed on **Stellar mainnet** against the deployed
executor contract
`CALCLWVZT6CQDSJV4MP3LOLSGKMYJNOWO5IL4Y54OY2EGZ2TF6RAF7QC`
(2026-06-13). Each is independently verifiable on a public explorer; the
contract id in every transaction matches the contract source in this repo.

| # | Function | Route | In | Out | Date (UTC) | Transaction |
|---|---|---|---|---|---|---|
| 1 | `swap` (split) | XLM → USDC, Soroswap + Phoenix 50/50 | 4 XLM | 7.608517 USDC | 2026-06-13 12:25:17 | `ee3446412436b39dba202a21a225eb3c25d93900bfde3e784481011242a4bb21` |
| 2 | `swap_aqua` | USDC → AQUA (Aquarius) | 0.2 USDC | 527.1959329 AQUA | 2026-06-13 12:42:47 | `3240332ad78dfc873c0ac239041f5503768c7d2b68d8a7fa59a6831384e0914f` |
| 3 | `swap_phoenix` | XLM → USDC (Phoenix) | 2 XLM | 3.799793 USDC | 2026-06-13 12:42:59 | `16b71d2dfa0b346d2d78fe6be1c925c203799ed17de6c181a103616c7a98b810` |
| 4 | `swap_aqua_then_soroswap` | AQUA → USDC → XLM (Aquarius → Soroswap) | 100 AQUA | 0.1970751 XLM | 2026-06-13 12:43:11 | `ec699cba0888d475ef4dcd40c72a49a5894b49c8c0fb8b56c579b28e6882c526` |

Explorer links (stellar.expert):

1. https://stellar.expert/explorer/public/tx/ee3446412436b39dba202a21a225eb3c25d93900bfde3e784481011242a4bb21
2. https://stellar.expert/explorer/public/tx/3240332ad78dfc873c0ac239041f5503768c7d2b68d8a7fa59a6831384e0914f
3. https://stellar.expert/explorer/public/tx/16b71d2dfa0b346d2d78fe6be1c925c203799ed17de6c181a103616c7a98b810
4. https://stellar.expert/explorer/public/tx/ec699cba0888d475ef4dcd40c72a49a5894b49c8c0fb8b56c579b28e6882c526

## Cross-protocol transaction (tx #4) — event highlights

The route AQUA → USDC → XLM executes in one transaction:

- Aquarius leg: `trade` on pool `CA6GAFOJ…` converts the input AQUA to USDC,
  delivered to the executor contract.
- Soroswap leg: `SoroswapPair swap` on pair `CAM7DY53…` converts that USDC
  (read from the contract's actual balance) to native XLM, delivered to the
  executor.
- The executor forwards the final XLM to the user.

Two different protocols, two different pull-authorization patterns, one
atomic `InvokeHostFunction`.

## Executor cleanliness

After the swaps above, the executor's token balances are zero:

```
balance(XLM, contract)  = 0
balance(USDC, contract) = 0
balance(AQUA, contract) = 0
```

queried via each Stellar Asset Contract's `balance()`. No user funds are
retained by the executor between transactions.

## Split plan used in tx #1

The `swap` entry point received this `--plan` (a `Vec<Strand>`): two
strands, `parts` 1:1, one hop each — Soroswap (venue 0) and Phoenix
(venue 2), both XLM→USDC. Unused venue-specific fields carry placeholders.
See [`split-plan.json`](./split-plan.json).


## Pooled-merge executor (`swap_merge`) — live mainnet swap

A second contract,
`CAQPNW2M6G5SV3AWJKUCQPOZ5RKEUUFYGSASU4WRCIVSQRXRGHQ2HRYY`,
carries the same entry points plus `swap_merge`, a pooled fan-in executor.
Where the linear plan (`Vec<Strand>`) runs each leaf path independently —
duplicating any hop several branches share — `swap_merge` executes the route
as topologically-ordered **stages** and splits each token's *pooled* on-chain
balance once across its pools. The result is one swap per graph **edge**
(not per leaf path), which both realises the true fan-in optimum and keeps
heavy multi-branch routes inside the Soroban per-transaction instruction
budget.

| Function | Route | In | Out | Date (UTC) | Transaction |
|---|---|---|---|---|---|
| `swap_merge` | AQUA → yXLM (3 split branches converging on XLM, then a pooled XLM→yXLM split across Aquarius + Soroswap) | 1001.89 AQUA | 1.9889078 yXLM | 2026-06-13 16:38:53 | `822ff1c1ee1d5f27e9c635676c1798bd390ab72b279234a04e9a9eb36d60d9b4` |

Explorer: https://stellar.expert/explorer/public/tx/822ff1c1ee1d5f27e9c635676c1798bd390ab72b279234a04e9a9eb36d60d9b4

**Plan shape (4 stages, 7 edge-swaps):**

- Stage 1 — AQUA: split across three pools (two Aquarius legs + one to a
  shared intermediate), feeding XRP / USDC / XLM.
- Stages 2–3 — XRP, USDC: each converted to XLM (one pool each).
- Stage 4 — XLM: the contract reads its **pooled** XLM balance (the sum of
  every upstream branch) and splits it once across an Aquarius and a Soroswap
  pool into yXLM.

Because the shared XLM→yXLM hop runs **once** on the pooled total rather than
once per upstream branch, the same route that overflows the instruction
budget when linearised fits comfortably as stages. The realised output
(1.9889078 yXLM) is **~0.48% above** the classic SDEX path for the same input,
and the pooled fan-in matches the route the off-chain pathfinder selected.
