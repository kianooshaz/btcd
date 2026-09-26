# Session 10 — The Mempool: What a Node Relays and Why

**Duration:** ~2 hours
**Goal:** Understand the mempool as *policy*, not consensus: which unconfirmed transactions a node accepts, relays, evicts, and how fees are estimated.

## Core concepts

Crucial distinction for the whole path:

- **Consensus**: what makes a block valid. Break it → block rejected.
- **Policy / standardness**: what *this node* is willing to relay and mine before it's confirmed. Stricter, configurable, changes without forks.

The mempool is an in-memory index of unconfirmed txs, including unconfirmed ancestors of mempool entries. Its jobs: relay txs, feed miners, provide `getrawmempool` data.

## Files to read

1. `mempool/README.md` and `mempool/doc.go` — overview and the policy list.
2. `mempool/interface.go` — the `Policy` and `BlockChain`-side interfaces the mempool needs; note how this keeps `mempool` decoupled from `blockchain` internals.
3. `mempool/mempool.go` — the main object: `TxPool`, `MaybeAcceptTransaction`, `ProcessTransaction` (the full pipeline: sanity → policy → context against chain + mempool → insert), `removeTransaction`/`removeDoubleSpends`, orphan handling (`maybeAcceptOrphans`, orphan eviction — orphans *are* stored, with caps), and the notification hooks.
4. `mempool/policy.go` — **the standardness rules** — read every `CheckTransactionStandard`-family check: dust (`calcMinRequiredTxRelayFee`), bare multisig, script length, `IsUnspendable` `OP_RETURN` handling, sequence+RBF (`CheckTransactionStandard` sigop/datacarrier limits), and `CalcPriority`.
5. `mempool/estimatefee.go` — fee estimation from observed blocks (the Q-learning-ish estimator): read `EstimateFee` and the decay math.
6. `mempool/error.go` — `RuleError` vs `TxRuleError`: how policy failures are reported to peers (this drives ban decisions upstream in `peer`/`server`).
7. `mining/policy.go` — the *mining* side policy (block template rules) — a sibling view of the same standardness ideas; skim now, deep-dive next-next session.

## Hands-on

1. `go test ./mempool/ -short` then `-run TestCalcMinRequiredTxRelayFee -v`.
2. Read `mempool/mocks.go` — a fake blockchain for unit tests; the interface seams make this possible. Imitate it in your own Go code someday.
3. On your running testnet node: `./btcctl --testnet getrawmempool verbose` — map each JSON field (`size`, `fee`, `ancestorcount`, `depends`) back to the struct it comes from (`mempool.go`'s descriptors).
4. Trace: a tx with a too-low fee arrives on the wire. Follow it: `server.go` on-Tx → `TxPool.ProcessTransaction` → `checkTransactionStandard` → rejection → what does the peer do? (find the relay/announcer side too)

## Exercises

1. List every way a tx can be rejected from the mempool *without* being invalid as a consensus matter. Categorize: size, fee, script type, input availability, malleability-related.
2. Find how a mempool tx is **evicted** when a confirmed tx spends the same input, and when the pool hits `MaxOrphanTxs`/size limits.
3. Set `--limitfreerelay`/`--minrelaysyncfee`-era flags in `sample-btcd.conf`... wait, find the *current* equivalent flags in `config.go` and note defaults.

## Check yourself

- Why is dust defined via *relay fee* rather than a fixed amount?
- Why keep orphans at all, and why cap them? (children arriving before parents)
- What tells the miner a mempool tx is safe to include? (validated + policy-passed + inputs still available)

**Next session:** `11-peer-to-peer-networking.md`
