# Session 01 — Build It, Run It, Break It

**Duration:** ~1.5 hours
**Goal:** Run a real node on testnet, use `btcctl` to talk to it, and learn your way around the entry-point code and the debugging workflow you'll use for the rest of the path.

## Prerequisites

- Session 00 completed
- Go installed (check `go version` against the version in `go.mod`)

## Part 1 — Build and run (30 min)

```bash
go build -o btcd .
go build -o btcctl ./cmd/btcctl
./btcd --testnet --rpcuser=k --rpcpass=k
```

In a second terminal:

```bash
./btcctl --testnet --rpcuser=k --rpcpass=k getblockchaininfo
./btcctl --testnet --rpcuser=k --rpcpass=k getpeerinfo
./btcctl --testnet --rpcuser=k --rpcpass=k getbestblockhash
./btcctl --testnet --rpcuser=k --rpcpass=k getrawmempool
```

While it syncs (testnet3 is small enough to finish), watch the log. Find in the logs: peer handshakes, "Processed X blocks", mempool additions.

Also useful:
```bash
./btcd --testnet --debuglevel=debug          # much more logging
./btcd --testnet --connect=YOUR_IP           # talk to one peer only
```

## Part 2 — Read the startup path (45 min)

Follow the code the way the process actually executes:

1. `btcd.go` — `main()`. Just parses config, sets up logging, calls `btcdMain`.
2. `config.go` — huge but mechanical: every flag/env/config-file option. Skim the `loadConfig` flow: what order are things validated, where does the data directory come from?
3. `log.go` — btcd's log routing (each package has its own subsystem logger).
4. `server.go` — **the heart of the glue layer**. `newServer` constructs every subsystem. Read it once slowly and list every component it creates: address manager, conn manager, sync manager, mempool, mining controller, RPC server...
5. `params.go` — how `--testnet`/`--regtest` select `chaincfg` parameters. Open `chaincfg/params.go` and look at `TestNet3Params`: genesis hash, checkpoints, DNS seeds.

Notice the pattern used everywhere in btcd: components are constructed with references to each other and communicate via channels and small interfaces (`mempool/interface.go`, `netsync/interface.go`).

## Part 3 — First code modification (15 min)

Make a trivial change and verify it end-to-end:

- In `btcd.go` or `server.go`, add a log line at startup: e.g. log the selected network's genesis hash (`cfg.ActiveNetParams.GenesisHash`).
- Rebuild, run, see your line in the output. Congratulations — you've built consensus-adjacent code.

## Exercises

1. Find which log subsystem prints "New valid peer" (grep `peer.go` and `log.go`).
2. In `btcctl`, call `getblock <hash>` on the genesis block (`--testnet` and mainnet have *different* genesis hashes — why? Check `chaincfg`).
3. Run `go vet ./...` and `go test ./blockchain/ -short` — make sure the project's tooling works on your machine now, not later when you need it.

## Check yourself

- What is the difference between `--testnet`, `--regtest`, and `--simnet`?
- Where does btcd store its data on your OS? (`btcutil/appdata.go`)
- What happens in `newServer` if the RPC cert generation fails?

**Next session:** `02-wire-protocol-messages.md`
