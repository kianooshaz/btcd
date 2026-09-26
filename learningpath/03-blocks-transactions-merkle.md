# Session 03 — Blocks, Transactions, and Merkle Trees

**Duration:** ~2 hours
**Goal:** Know the exact structure of a Bitcoin transaction and block, how SegWit changed the layout, and how Merkle roots commit to block contents.

## Core concepts

A transaction:
```
version (4) | marker+flag (2, SegWit only) | txins | txouts | witness | locktime (4)
```
Each `txin`: `prevout{hash, index} | scriptSig | sequence`
Each `txout`: `value (8) | pkScript`
Coinbase = first input of a block, `prevout` hash all-zeros.

A block:
```
80-byte header | varint tx count | txs...
```
Header: `version | prevHash | merkleRoot | timestamp | difficultyBits | nonce`

## Files to read

1. `wire/msgtx.go` — **the longest read of the path, take it slow.** `MsgTx`, `AddTxIn`, `AddTxOut`, `BtcDecode` (watch the witness marker/flag branch), `TxHash()` vs `WitnessHash()` — the SegWit txid/wtxid distinction lives right here.
2. `wire/msgblock.go` — `MsgBlock`, `Transactions`, witness encoding, `Deserialize`.
3. `wire/blockheader.go` — 80 bytes, `BlockSha`.
4. `btcutil/tx.go` and `btcutil/block.go` — the *higher-level* wrappers the rest of btcd actually passes around (with cached hashes, height, etc.). Note the pattern: `wire` structs are the wire format; `btcutil` wrappers are the working format.
5. `blockchain/merkle.go` — `CalcMerkleRoot`: why leaves are duplicated for odd counts, and why it uses `TxHash` (witness-stripped) while a separate commitment (`witnessMerkleRoot` concept) handles witnesses.
6. `btcutil/amount.go` — 8 decimals as `int64`. Small file, big lesson about fixed-point money.

## Exercise: dissect a real transaction

1. Grab a raw tx: `./btcctl --testnet getrawtransaction <txid>`
2. Write a scratch Go program: decode it with `btcutil.NewTxFromHex`, print version, inputs (prevout hash/index, sequence), outputs (value in BTC, script length), locktime, and `msgTx.TxHash()` vs `msgTx.WitnessHash()`.
3. Repeat with a SegWit tx (one with witnesses) — see which fields the classic `TxHash` ignores.
4. Same flow for a block: `getblock <hash> 0` then `btcutil.NewBlockFromBlockAndBytes`; verify `block.Hash()` and that `MsgBlock` tx count matches.

## Coding exercise: Merkle root check

Using `blockchain.CalcMerkleRoot` (or writing your own 20-line version with `chainhash.DoubleHashB`), compute a block's merkle root from its txids and compare with the header. Get a real block via RPC.

## Check yourself

- Why do SegWit txids exclude witness data? (hint: malleability — what could a non-signer change without invalidating the tx?)
- What is `sequence` used for today? (RBF signalling — check `mempool/policy.go` later)
- Why is the coinbase input's prevout all-zeros and what happens if it isn't? (consensus rule — you'll meet the check in `blockchain/validate.go`)

**Next session:** `04-script-language-and-engine.md`
