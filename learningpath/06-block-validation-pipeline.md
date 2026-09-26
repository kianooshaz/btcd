# Session 06 — The Block Validation Pipeline

**Duration:** ~3 hours
**Goal:** Follow a block from "bytes off the wire" to "part of the best chain", through every consensus check. This is the heart of the node and the longest session.

## Core concepts

Two kinds of rules:
- **Context-free**: valid for any block, alone (merkle root matches, proof-of-work target met, coinbase shape, script limits).
- **Contextual**: depend on where the block lands in the chain (difficulty retarget, median-time-past locktimes, BIP-30/34/113, soft-fork activation, coinbase maturity).

And two outcomes for a valid block: **extends best chain** vs **becomes a side-chain fork** (stored, waiting to possibly win later).

## Files to read (this is a guided path, follow it)

1. `blockchain/README.md` and `blockchain/doc.go` — the overview: `Chain`, `BlockChain`, chain state, the notification system.
2. `blockchain/chain.go` — `New`, `BestChain()`, `ProcessBlock` entry, `connectBestChain`, `reorganizeChain`. Read `ProcessBlock` fully: orphan handling → already-have → PoW check (yes, twice: cheap early check, then context) → `checkBlockContext` → `connectBestChain`.
3. `blockchain/process.go` — the dispatcher: `maybeAcceptBlock` and friends; where "block already known" and "parent missing" are decided.
4. `blockchain/validate.go` — **the rulebook. ~2000 lines, but every function is one rule.** Read in order:
   - `checkBlockHeaderContext` — difficulty (`CalcNextRequiredDifficulty`), PoW (`checkProofOfWork`), header sanity
   - `CheckBlockSanity` — context-free checks: merkle root, coinbase position, tx count vs merkle count
   - `checkBlockContext` — timestamp median-time rule, coinbase height in scriptSig (BIP-34), witness commitment (BIP-141)
   - `CheckTransactionInputs` — the UTXO walkthrough: double spends *within the block*, coinbase maturity (`CoinbaseMaturity`), output sums vs input sums, `CheckSerialiseScripts` boundary
   - `checkConnectBlock` — the big one: runs every tx's scripts against the UTXO view, enforces BIP-16, BIP-30, BIP-34,SigOps limits, weight limits
5. `blockchain/difficulty.go` — retarget every 2016 blocks, the 4x clamp, testnet's special 20-minute rule.
6. `blockchain/mediantime.go` — BIP-113's median-of-11 timestamp rule. Tiny file, real consensus rule.
7. `blockchain/accept.go` — `maybeAcceptBlock`: checkpoint logic, non-standard-tx policy vs consensus.

## Trace exercise (the most valuable 45 minutes of the path)

Take one *real* block and annotate `ProcessBlock` with which check each line corresponds to:

1. `./btcctl --testnet getblockhash 100` → grab an early block.
2. In a scratch program, deserialize it (`btcutil.NewBlockFromBlockAndBytes`) and call the exported checks: `blockchain.CheckBlockSanity`, `blockchain.CheckTransactionInputs` (needs a `UtxoViewpoint` — see how tests in `validate_test.go` fake the view), `blockchain.CheckBlockHeaderContext` (needs `BestState` — again see tests).
3. Then verify how `blockchain_test.go`/`fullblocks_test.go` drive the *whole* pipeline with test data (`testdata/`), and run: `go test ./blockchain/ -short`.

## Exercises

1. Find where a block with a bad merkle root fails, and write down the exact error chain (`RuleError` types in `blockchain/error.go`).
2. Change one byte of a coinbase tx in a downloaded block and re-run `CheckBlockSanity` — which check catches it first?
3. Locate the SigOp counting (grep `CountSigOps`) and the block weight check (grep `BlockWeight`, see `blockchain/weight.go`) in `checkConnectBlock`.
4. Run `go test ./blockchain/fullblocks_test.go -short` style tests and skim one `fullblocktests` scenario — these are "evil block" batteries.

## Check yourself

- Why is PoW checked before contextual checks? (cheap rejection of spam before expensive work)
- What's the difference between an orphan block and a side-chain block in btcd's code?
- Where does a *reorg* actually swap the tip, and what happens to the UTXO set? (`disconnectTransactions`/`connectTransactions`)

**Next session:** `07-utxo-set-and-views.md`
