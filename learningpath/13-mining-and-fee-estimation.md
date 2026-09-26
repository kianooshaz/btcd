# Session 13 — Mining: Building Block Templates

**Duration:** ~2 hours
**Goal:** Understand how a block template is built from mempool contents: coinbase construction, tx selection policy, weight/sigop accounting, merkle assembly — and how `cpuminer` actually grinds hashes.

## Core concepts

A **block template** = "a block that would be valid *right now*": selected txs + coinbase + header with a claimed target. Mining = validating the template as a block *under your own rules*, then racing to find a nonce satisfying PoW.

Even if you never mine, this package is where **policy becomes concrete**: the template builder decides which txs make it into the next block — fees, weight, sigops, and package limits in action.

## Files to read

1. `mining/README.md` — the split: `mining.go` (template logic) vs `cpuminer/` (a toy miner for solo/experiment use; real pools talk Stratum, which btcd does *not* implement).
2. `mining/mining.go` — `BlkTmplFetcher`-era interfaces and `TxDesc`: the mempool view used for mining.
3. `mining/cpuminer/miner.go` — read the whole flow:
   - `generateBlocks`: loop — template → header assembly → **nonce loop** (`UpdateBlockTime`, `solveBlock`) → submit via `submitBlock` callback into the server's normal `ProcessBlock` path
   - the 4-way nonce/unrolL and per-template share work distribution
4. `mining/policy.go` — template policy: `GetMinRelayFee`-style checks in selection, max block weight/reserved space (`cfg.BlockMaxWeight`), priority scoring.
5. `server.go` — `submitBlock` (find it): template submission *re-enters* the exact validation pipeline from Session 06 — mining is not exempt from consensus.

## The coinbase, concretely

Find in the template code where the coinbase tx is assembled: input (witness-reserved-value for SegWit — recall BIP-141's commitment rule), outputs (fees = inputs − outputs across selected txs), and the **witness commitment** appended as the last output of the coinbase. Cross-reference with `blockchain/validate.go`'s `checkWitnessCommitment` (or similarly named function): miner builds it, node re-computes it.

## Hands-on

1. Regtest is your friend:
   ```bash
   ./btcd --regtest --rpcuser=k --rpcpass=k --miningaddr=<your regtest addr>
   ./btcctl --regtest --rpcuser=k --rpcpass=k generate 5
   ```
   `generate` drives `cpuminer` synchronously. Then `getblock <hash> 1` and verify: coinbase amount, witness commitment output, tx ordering vs fee.
2. `go test ./mining/... -short` and skim one template test to see how a fake chain feeds the builder.
3. In `miner.go`, add a debug log printing hashes/sec — then delete it. Feel the nonce loop.

## Exercises

1. Compute by hand: for a template with txs summing inputs 100,000 sats and outputs 99,500 sats, what's the coinbase value for a 50-BTC-subsidy regtest block? Where in code is the subsidy looked up? (grep `CalcBlockSubsidy`)
2. Find where the template builder stops adding txs: weight limit? sigops limit? both? Which check fires first for a mempool full of multisig?
3. Explain what `UpdateBlockTime` must respect on regtest vs testnet (recall `mediantime.go` and the testnet 20-minute rule).

## Check yourself

- Why must mining submit through the *same* validation path as peers? (a miner that accepts its own invalid block wastes everything)
- What happens if two cpuminer goroutines find a solution simultaneously?
- Where does the template get its "parent block" (previous hash + height + difficulty) from? (`chain.BestSnapshot`-style call — trace it)

**Next session:** `14-rpc-api-and-capstone.md`
