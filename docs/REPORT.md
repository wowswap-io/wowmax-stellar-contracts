# WOWMAX Stellar — On-Chain Execution Layer

**Project:** WOWMAX Stellar DEX Aggregator (SCF Build, Integration Track)
**Component:** On-chain swap-plan executor (Soroban)
**Network:** Stellar mainnet (live pools, no fixtures)
**Contract:** `CALCLWVZT6CQDSJV4MP3LOLSGKMYJNOWO5IL4Y54OY2EGZ2TF6RAF7QC`

---

## 1. Scope

This component is the on-chain half of the WOWMAX Stellar aggregator: a
Soroban contract that **executes a pre-computed swap plan** atomically in a
single transaction. The plan — which pools to use, how to split the input,
and the hop sequence — is produced off-chain by the WOWMAX pathfinder and
passed to the contract explicitly. The contract executes exactly that plan
and enforces end-to-end slippage.

The contract deliberately holds **no routing logic**. It is a thin,
predictable executor: pull input from the user, run the plan's legs against
the named pools, enforce `amount_out_min` on the summed output, forward
proceeds. This keeps the on-chain surface small and auditable.

## 2. What it does beyond a per-protocol aggregator

The common Soroban aggregator dispatches each split leg to a single
protocol (one protocol per path) and lets that protocol's router choose
pools. This contract adds:

1. **Cross-protocol multi-hop in one call.** A single hop sequence can mix
   protocols — e.g. AQUA→USDC on Aquarius, then USDC→XLM on Soroswap — in
   one atomic `InvokeHostFunction`. The contract holds the intermediate
   token between hops and feeds it forward by its actual balance.
2. **Exact-pool targeting.** Each leg names its precise pool (Aquarius
   `pool_index`, Soroswap pair, Phoenix pool), rather than delegating pool
   selection to a protocol router.
3. **Atomic slippage on the summed output.** `amount_out_min` is checked on
   the total across all split branches; any failing branch reverts the
   whole transaction.

## 3. Per-protocol authorization

Each Soroban DEX pulls the input token differently. The contract sets the
correct `authorize_as_current_contract` sub-invocation per venue,
established empirically from mainnet simulation:

| Venue | Entry call | Token pulled to |
|---|---|---|
| Soroswap | `swap_exact_tokens_for_tokens` | the **pool** |
| Aquarius | `swap_chained` | the **router** |
| Phoenix  | `swap` (7-arg) | the **pool** |

For cross-protocol and split plans, the contract holds funds between legs
and acts as the swapper itself (it is the `to`/`sender`/`user` arg of each
protocol call), forwarding the final output to the end user.

## 4. Plan model

```
plan: Vec<Strand>
Strand { parts: u32, hops: Vec<Hop> }
Hop    { venue: u32, pool, token_in, token_out, <venue-specific fields> }
```

- `venue`: 0 = Soroswap, 1 = Aquarius, 2 = Phoenix.
- Split: `strand_in = floor(amount_in * parts / total_parts)`; the last
  strand takes the remainder so the split sums to `amount_in` exactly.
- Multi-hop: a strand's hops run sequentially, each consuming the previous
  hop's output (read from the contract's actual balance).
- Slippage: enforced once, on the sum of all strand outputs.

## 5. Mainnet validation

All entry points were exercised on Stellar mainnet against live pools on
the deployed contract
`CALCLWVZT6CQDSJV4MP3LOLSGKMYJNOWO5IL4Y54OY2EGZ2TF6RAF7QC`. Every
transaction is verifiable on a public explorer.

| Function | Route | Input | Output | Tx |
|---|---|---|---|---|
| `swap` (split) | XLM → USDC via Soroswap + Phoenix (50/50) | 4 XLM | 7.608517 USDC | [`ee344641…`](https://stellar.expert/explorer/public/tx/ee3446412436b39dba202a21a225eb3c25d93900bfde3e784481011242a4bb21) |
| `swap_aqua` | USDC → AQUA (Aquarius) | 0.2 USDC | 527.1959329 AQUA | [`3240332a…`](https://stellar.expert/explorer/public/tx/3240332ad78dfc873c0ac239041f5503768c7d2b68d8a7fa59a6831384e0914f) |
| `swap_phoenix` | XLM → USDC (Phoenix) | 2 XLM | 3.799793 USDC | [`16b71d2d…`](https://stellar.expert/explorer/public/tx/16b71d2dfa0b346d2d78fe6be1c925c203799ed17de6c181a103616c7a98b810) |
| `swap_aqua_then_soroswap` | AQUA → USDC → XLM (Aquarius → Soroswap) | 100 AQUA | 0.1970751 XLM | [`ec699cba…`](https://stellar.expert/explorer/public/tx/ec699cba0888d475ef4dcd40c72a49a5894b49c8c0fb8b56c579b28e6882c526) |

The cross-protocol transaction (`ec699cba…`) is the headline case: its
event log shows an Aquarius `trade` (AQUA→USDC) followed by a Soroswap
`swap` (USDC→XLM) within one `InvokeHostFunction` — a route a one-
protocol-per-path aggregator cannot express.

**No funds retained.** After all swaps, the contract's balances of XLM,
USDC, and AQUA are each zero (verified via each SAC's `balance(contract)`),
confirming every leg forwards proceeds to the user and nothing is stranded
in the executor.

## 6. How to reproduce

Build and deploy:

```bash
cd contracts
stellar contract build
stellar contract deploy \
  --wasm target/wasm32v1-none/release/wowmax_stellar_router.wasm \
  --source <your-key> --network mainnet
```

Invoke a single Phoenix swap (XLM→USDC) as an example:

```bash
stellar contract invoke --id <DEPLOYED_C> \
  --source <your-key> --network mainnet --inclusion-fee 10000000 \
  -- swap_phoenix \
  --user <your-key-public> \
  --pool CBHCRSVX3ZZ7EGTSYMKPEFGZNWRVCSESQR3UABET4MIW52N4EVU6BIZX \
  --token_in CAS3J7GYLGXMF6TDJBBYYSE3HQ6BBSMLNUQ34T6TZMYMW2EVH34XOWMA \
  --token_out CCW67TSZV3SSS2HXMBQ5JFGCKJNXKZM7UQUWUZPUTHXSTZLEO7SJMI75 \
  --amount_in 20000000 --amount_out_min 1
```

The `swap` entry point takes a `--plan` argument (a JSON array of strands);
see [`evidence/mainnet-tx.md`](./evidence/mainnet-tx.md) for the exact
split-plan used in the validation above.

## 7. Reference addresses (mainnet)

| Item | Address |
|---|---|
| Executor contract | `CALCLWVZT6CQDSJV4MP3LOLSGKMYJNOWO5IL4Y54OY2EGZ2TF6RAF7QC` |
| Soroswap router | `CAG5LRYQ5JVEUI5TEID72EYOVX44TTUJT5BQR2J6J77FH65PCCFAJDDH` |
| Soroswap XLM/USDC pair | `CAM7DY53G63XA4AJRS24Z6VFYAFSSF76C3RZ45BE5YU3FQS5255OOABP` |
| Aquarius router | `CBQDHNBFBZYE4MKPWBSJOPIYLW4SFSXAXUTSXJN76GNKYVYPCKWC6QUK` |
| Aquarius USDC/AQUA pool | `CA6GAFOJCW4MGQQBUCQUSA3CLIH25G4SNKB2JHYKZCVWZTNW5VXMSC4O` |
| Phoenix XLM/USDC pool | `CBHCRSVX3ZZ7EGTSYMKPEFGZNWRVCSESQR3UABET4MIW52N4EVU6BIZX` |
| XLM (native SAC) | `CAS3J7GYLGXMF6TDJBBYYSE3HQ6BBSMLNUQ34T6TZMYMW2EVH34XOWMA` |
| USDC SAC | `CCW67TSZV3SSS2HXMBQ5JFGCKJNXKZM7UQUWUZPUTHXSTZLEO7SJMI75` |
| AQUA SAC | `CAUIKL3IYGMERDRUN6YSCLWVAKIFG5Q4YJHUKM4S4NJZQIA3BAS6OJPK` |
