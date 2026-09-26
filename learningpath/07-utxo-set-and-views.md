# Session 07 — The UTXO Set: Bitcoin's Actual Database of Money

**Duration:** ~2 hours
**Goal:** Understand what the UTXO set is, how btcd represents it in memory and on disk, and how `UtxoViewpoint` stages changes during validation.

## Core concepts

Bitcoin has no accounts. The spendable money = the set of **unspent transaction outputs** (UTXOs). Spending tx N deletes outputs and creates new ones. Balance = sum of outputs locked to your keys.

btcd's modern UTXO design:
- **On disk**: a flat UTXO set snapshot per tip (see `blockchain/upgrade.go` history notes and `chainio.go`) — built once via migration, then updated incrementally.
- **In RAM**: `utxocache.go` — an LRU write-back cache in front of the DB, flushed periodically (this is the big performance feature; read the comments on flush intervals).
- **Per-operation**: `utxoviewpoint.go` — a *view* = the working set of outputs touched by the txs currently being validated. Reads miss to the cache/DB, writes are staged, committed atomically on block accept, rolled back on disconnect.

## Files to read

1. `blockchain/interfaces.go` — `UtxoEntry` and `UtxoViewpoint` interfaces plus `SpendableInput`/`OutputDetail`: the *contracts* everything is written against. Read `FetchUtxoEntry`, `FetchUtxos`, `SpendUtxo` semantics.
2. `blockchain/utxoviewpoint.go` — the concrete view: `AddTxOuts`, `commit`, `disconnectTransactions` (how a reorg *un-spends* things — spentness flags, `IsFullySpent`).
3. `blockchain/utxocache.go` — `UtxoCache`, `addTxOuts`, `addTxOut`, `fetchUtxos`, `Flush`, eviction. Note the `spentTxOuts` handling for disconnects and how height of each entry is tracked (needed for coinbase maturity and locktime).
4. `blockchain/chainio.go` — `dbFetchUtxoEntry`, `dbPutUtxoView`, how entries are compressed on disk.
5. `blockchain/compress.go` — **clever and fun**: amount compression (exponent+mantissa) and script compression (type byte + known-template stripping). This is why the UTXO set is smaller than you'd guess.
6. `blockchain/validate.go` — revisit `CheckTransactionInputs` now with the view model clear: `view.FetchUtxos` → maturity checks → `view.SpendUtxo` staging.

## Hands-on

1. `go test ./blockchain/ -run TestUtxoCache -v` and skim `utxocache_test.go`: watch a view spend outputs, commit, and flush.
2. Write a scratch program: build a tiny fake chain in memory (copy the setup from `blockchain/example_test.go` — it exists and is excellent), mine a coinbase, then spend it in block 2 and inspect the view.
3. In `utxocache.go`, add a temporary log line in `Flush` printing entries written; run the short tests; remove it.

## Exercises

1. Answer from code: where is `CoinbaseMaturity` (=100) enforced, and which entry field does it need? (`HasExpiry`/height)
2. Trace what happens to a UTXO during: (a) spend in a block on the best chain, (b) its block gets reorged away. Name the functions.
3. Grep `unspent()` — why is a spent-but-cached entry kept until commit?
4. Read `blockchain/example_test.go` end-to-end. It is the single best "API tour" of the package.

## Check yourself

- Why can't btcd just store "balances per address"? (spend authorization is per-output, not per-account; plus privacy/indexing reasons)
- What does `UtxoEntry.SpentBy`-style state look like for a coinbase output vs a normal one?
- What happens if the process crashes between UTXO cache flush and block-index update? (find the ordering in `connectBestChain`)

**Next session:** `08-chain-index-forks-and-fork-activation.md`
