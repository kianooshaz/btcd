# Session 14 — The RPC Server and Your Capstone Project

**Duration:** ~2.5 hours
**Goal:** Understand how the node exposes itself: the RPC surface, JSON/RPC plumbing, websockets (block notifications), and the `rpcclient` library — then prove your learning with a capstone project.

## Part 1 — RPC architecture (1 hour)

1. `rpcserver.go` — **huge file; read structurally.** Find:
   - method registry: the map of `"getblockchaininfo"` → handler; every method's help text lives in `rpcserverhelp.go`
   - `internalRPC`/`unimplemented` patterns, auth (TLS certs from Session 01), and how `btcjson` types shape requests
   - the adapters (`rpcadapters.go`) — narrow interfaces that keep the RPC layer honest about what a chain/mempool/miner actually offers
2. `btcjson/` — request/response structs, the `Cmd` generation style. Skim `btcjson/README.md` and one command file (e.g. `walletsvrwsntfns.go` or `blockchainsvr.go`) to see the pattern.
3. `rpcwebsocket.go` — the notification channel: `notifyblocks` sends every connect/disconnect event to wallet-like clients. This is how `btcwallet` lives happily next to a wallet-less btcd.
4. `rpcclient/` — the *client* library (what `btcctl` and `btcwallet` use). Skim `rpcclient/README.md`, `chain.go`, and `example_test.go`.
5. `cmd/btcctl/` — how a CLI is built on `rpcclient`. You've used it all path; now read it.
6. `docs/json_rpc_api.md` — the full method list; skim once so you know what exists.

## Hands-on

1. Enable websockets and use `rpcclient` in a scratch program:
   ```go
   connCfg := &rpcclient.ConnConfig{ Host: "localhost:18334", ... }
   client, _ := rpcclient.New(connCfg, nil)
   ntfn, _ := client.NotifyBlocks()
   // print every block hash for 2 minutes
   ```
2. Call `getbestblockhash`, `getblockheader <hash>`, `gettxout <txid> <n>` and map each response field to the `btcjson` struct.
3. Find one RPC method that is implemented *only* via `btcctl` convenience (i.e., a client-side combination) — if none, find the help-text generation for `getblock` and trace how `rpcserverhelp.go` is built.

## Part 2 — Capstone (choose one, ~2+ hours)

Pick **one** and actually build it. Each is doable with what you've learned:

**A. Block explorer from the ground up (recommended)**
A CLI/program that: takes a block hash → fetches via `rpcclient` → decodes with `btcutil`/`wire` → prints tx tree with fee estimation from inputs/outputs → for SegWit inputs, re-computes the BIP-143 sighash for one input and verifies it against the witness. Touches: 02, 03, 04, 05, 14.

**B. Mini node-with-a-compass**
Instrument btcd (temporary commits, not for upstream): log every consensus-rule rejection in `blockchain/validate.go` with block hash + rule name; sync testnet from scratch; produce a summary table of which rules reject the most pre-IBD junk. Touches: 06, 08, 12.

**C. Write a `getmempoolancestors` clone**
Add a new RPC method that, given a mempool txid, walks `mempool`'s ancestor set and returns them (compare with Bitcoin Core's method of the same name; btcd may or may not have it — check first, and if it exists, instead write `getmempooldescendants`). Touches: 10, 14; real patch-quality work, including `rpcserverhelp.go` entries and a test.

**D. Fuzz-adjacent hardening study**
Pick 3 `*_test.go` files in `txscript` (e.g. `reference_test.go`) and 3 in `wire`; write one new table-driven test each from Bitcoin Core's test vectors that btcd lacks. Touches: everything, plus testing craft.

## Finishing

- Re-read your notes from Session 00. Every question you wrote down should now have an answer.
- Skim btcd's git log for the last year (`git log --oneline -100`) — you should understand most commit messages now. That comprehension is the actual finish line.

## Where to go next

- Upstream: btcsuite/btcd PRs and issue tracker — read a merged PR end-to-end.
- `btcwallet` (github.com/btcsuite/btcd/wallet if in-tree or separate repo): the wallet layer this node was designed for.
- LN: `btcd` + `btcwallet` is the node stack **lnd** runs on; `lnd` source is a natural next codebase.
- Bitcoin Core: read `validation.cpp` — you'll recognize every check from Session 06, now in C++.
