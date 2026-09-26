# btcd Learning Path

A hands-on, code-first path to learning Bitcoin implementation by reading and working inside the **btcd** full-node codebase (Go). One file = one working session (~1.5–3 hours). Sessions build on each other — do them in order.

## The path

| # | Session | Focus |
|---|---------|-------|
| 00 | [Orientation & architecture](00-orientation-and-architecture.md) | What a full node is, the codebase map |
| 01 | [Build & run your node](01-build-and-run-your-node.md) | btcd + btcctl, startup code path |
| 02 | [Wire protocol](02-wire-protocol-messages.md) | Message framing, serialization |
| 03 | [Blocks, txs, merkle](03-blocks-transactions-merkle.md) | Data structures, SegWit layout |
| 04 | [Script & the engine](04-script-language-and-engine.md) | Bitcoin's virtual machine |
| 05 | [Signatures & sighash](05-signatures-sighash-and-crypto.md) | ECDSA, Schnorr, what sigs commit to |
| 06 | [Block validation pipeline](06-block-validation-pipeline.md) | Every consensus check, wire-to-chain |
| 07 | [UTXO set & views](07-utxo-set-and-views.md) | How money state is stored and updated |
| 08 | [Block index, forks, activation](08-chain-index-forks-and-fork-activation.md) | Fork choice, reorgs, BIP-9 |
| 09 | [Database layer](09-database-layer.md) | ffldb, flat files, schema on top |
| 10 | [Mempool & policy](10-mempool-and-policy.md) | Consensus vs standardness |
| 11 | [P2P networking](11-peer-to-peer-networking.md) | peer, connmgr, addrmgr state machines |
| 12 | [Initial block download](12-initial-block-download-netsync.md) | Header-first sync, pipelines |
| 13 | [Mining](13-mining-and-fee-estimation.md) | Block templates, cpuminer |
| 14 | [RPC & capstone](14-rpc-api-and-capstone.md) | RPC surface + choose a capstone project |

## How to use it

- **Prerequisites**: comfortable Go (goroutines, channels, interfaces), and a basic idea of what Bitcoin is — sessions teach the *implementation*, not Bitcoin-from-scratch. A quick skim of the Bitcoin developer guide (developer.bitcoin.org) before Session 04 helps.
- **Always use testnet or regtest** for running nodes; never mainnet with a fresh wallet setup.
- Every session has: reading list (real files in this repo), hands-on exercises, and self-check questions. Do the exercises — reading alone won't stick.
- Sessions 04–06 are the hardest; the rest depend on them most.
- Keep notes per session; Session 14's capstone assumes them.
