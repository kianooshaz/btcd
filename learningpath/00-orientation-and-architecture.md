# Session 00 — Orientation: What btcd Is and How It Fits Together

**Duration:** ~1 hour reading, no coding
**Goal:** Understand what a full node does, what btcd implements, and get a mental map of the whole codebase before touching any of it.

## What btcd is

btcd is a full Bitcoin node written in Go. Like Bitcoin Core (C++), it:

- speaks the Bitcoin peer-to-peer protocol (`wire`, `peer`, `connmgr`)
- downloads, validates, and stores the blockchain (`netsync`, `blockchain`, `database`)
- enforces consensus rules — the same rules Bitcoin Core enforces (`blockchain`, `txscript`)
- maintains a mempool of unconfirmed transactions (`mempool`)
- optionally mines (`mining`)
- exposes an RPC API compatible with Bitcoin Core (`rpcserver.go`, `btcjson`)

One important thing it deliberately does **not** do: wallet functionality. btcd is a node only; wallets (like `btcwallet`) connect to it over RPC.

## The big picture: message flow

```
                  +---------------------------------------------------+
                  |                      btcd                          |
  peers <----P2P--+--> peer --> connmgr --> server.go                  |
                  |     (encryption)         |                        |
                  |                          v                        |
                  |                     netsync                       |
                  |                          |                        |
                  |                          v                        |
                  |                     blockchain                    |
                  |         (consensus rules, UTXO set, block index)  |
                  |                       |        |                  |
                  |                       v        v                  |
                  |                   database   mempool               |
                  |                    (ffldb)       ^                 |
                  |                                  |                 |
  wallet/CLI <-RPC+-- rpcserver.go ---- cpuminer/mining                |
                  +---------------------------------------------------+
```

A transaction arrives over P2P → goes to the mempool (policy checks) → when a miner (or another peer) includes it in a block → the block arrives → `blockchain` validates every rule from genesis to this block → the accepted state is persisted in `database`.

## The layered design

btcd is a set of **independent, reusable packages**, glued together by the top-level files. This is the single most important architectural fact:

| Layer | Package | Reads like |
|---|---|---|
| Serialization | `wire/` | pure structs + read/write bytes |
| Crypto/keys | `btcec/`, `btcutil/` | math + address formats |
| Script | `txscript/` | a small virtual machine |
| Consensus | `blockchain/` | the rulebook |
| Storage | `database/` | a KV abstraction over flat files |
| Policy (non-consensus) | `mempool/`, `mining/` | "what we relay/mine" |
| Networking | `peer/`, `connmgr/`, `addrmgr/`, `netsync/`, `v2transport/` | state machines |
| Glue | `btcd.go`, `server.go`, `config.go`, `rpcserver.go` | wiring |

Read the top-level README.md and each package's `README.md` and `doc.go` — the maintainers wrote short, excellent overviews of every package.

## Files to read today (just skim, don't deep-read)

- `README.md` — project overview and install
- `btcd.go` — 60 lines. The real `main()`. See how little it does.
- `doc.go` (top level) — package documentation
- `docs/README.md` — docs index
- `blockchain/README.md`, `txscript/README.md`, `wire/README.md` — read all package READMEs
- `sample-btcd.conf` — every config option, commented. Skimming this tells you what a node can be configured to do.

## Exercises

1. `go build` at the repo root, then `go run . --help`. You now have a node binary.
2. Start it against **testnet3** (`go run . --testnet`) in one terminal, watch the logs for 5 minutes. Note every log line you don't understand — the learning path will answer them.
3. Draw (on paper) the dependency arrows between: `wire`, `blockchain`, `database`, `mempool`, `peer`, `netsync`, `server.go`. Which ones depend on which?

## Check yourself

- What are the two "roles" of a full node? (hint: validate + relay)
- Why is the wallet separate from btcd?
- Which package would you reuse if you wanted to write a *parser* for .blk block files without running a node? (answer: `wire` + `btcutil`)

**Next session:** `01-build-and-run-your-node.md`
