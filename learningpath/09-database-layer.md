# Session 09 — The Database Layer: ffldb and the Block-Storage Abstraction

**Duration:** ~1.5 hours
**Goal:** Understand btcd's storage design: a small KV interface, a flat-file block store, and why blocks and the block index live separately from the UTXO state.

## Core concepts

btcd's storage is two ideas stacked:

1. `database` package = a **key/value interface** (`database.DB`) with typed buckets, cursors, ACID transactions — implemented by `ffldb` ("flat-file + leveldb-style metadata").
2. On top of that abstraction, `blockchain` lays out *its own* schema: buckets for block index, UTXO state, best state, thresholds, upgrades. Consensus data is **content-addressed**: blocks are stored by hash in flat .ffl files, indexed by metadata in a bucket.

The design goal: `database` knows nothing about Bitcoin, so a different backend could be swapped in (there was a BoltDB driver once; the interface is what makes that possible).

## Files to read

1. `database/README.md` and `database/doc.go` — interface tour: `DB`, `Tx` (read/write), `Bucket`, `Cursor`, drivers, the `BlockStore`-ish helpers (`FetchBlockByHash` etc.).
2. `database/interface.go` — read the doc comments carefully; they define the contract (repeatability of read txs, write-tx exclusivity).
3. `database/driver.go` — driver registration pattern (`DriverDB` registry) — a nice Go idiom used across btcd.
4. `database/ffldb/` — the implementation, skim in this order:
   - `db.go` — `openDB`, the metadata bucket layout, `commitTx` (two-phase: flat files then metadata, crash-safety discussion in comments)
   - `blockio.go` — `fetchBlockByHash`, the flat-file format: `(fileNum, offset, length)` location records
   - `reconcile.go` (if present) — bulk-integrity checks
   - `interface_test.go`/`db_test.go` — how the contract is tested
5. `blockchain/chainio.go` — the *schema* btcd puts on top: bucket names (`blkhdr`... check actual names), `dbFetchBlockByNode`, `dbPutBestState`, spend-journal-ish structures and the UTXO serialization you met in Session 07.
6. `blockchain/upgrade.go` — versioned migrations: how btcd upgrades old data dirs in place (real-world lesson in schema evolution).

## Hands-on

1. Run `go test ./database/ -short` and `go test ./database/ffldb/ -short`.
2. Write a scratch program: open an `ffldb` DB in a temp dir, create buckets, put/get/iterate with a cursor, use `Update` and `View`. (Model it on `database/example_test.go`.)
3. Point it at your real node: open the btcd **data dir** read-only and list the buckets in the metadata (careful: take a copy first; opening live DBs of a running node is a no). `cmd/addblock` and `cmd/findcheckpoint` show safe real-world usage of the API — skim `cmd/addblock/main.go`.

## Exercises

1. Draw the on-disk layout: what lives in flat files vs metadata buckets? Why are blocks *not* in the KV store itself?
2. Find how `database.DB` prevents two processes from opening the same data dir (file locking in ffldb).
3. Trace one `FetchBlockByHash` call: interface → driver → ffldb → flat file read. Note every error path.

## Check yourself

- Why is the write path "write flat files first, commit metadata last"?
- What does `database.Tx` give you that a plain mutex wouldn't? (MVCC-ish snapshot reads, rollback)
- Where does the block index persist, and what happens on a crash mid-write? (replay/upgrade logic in `blockchain/upgrade.go`)

**Next session:** `10-mempool-and-policy.md`
