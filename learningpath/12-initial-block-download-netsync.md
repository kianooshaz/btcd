# Session 12 — Initial Block Download: the netsync Manager

**Duration:** ~2 hours
**Goal:** Understand how a new node goes from genesis to tip efficiently: peer selection, header-first download, parallel fetch pipelines, and how progress is reported.

## Core concepts

Modern sync is **header-first**: fetch all headers cheaply (tiny messages) to learn the best chain, then download blocks with bounded parallelism, validating each against its already-known header.

`netsync` implements the sync-manager role: it talks to the peer package through an interface (`SyncManager` callbacks registered by `server.go`), owns the download state machine, and decides *which peer to ask for which block*.

## Files to read

1. `netsync/README.md` and `netsync/doc.go` — the concurrency model: one manager, message-driven, `peerState` map.
2. `netsync/interface.go` — the narrow interface between `netsync`, `peer`, and `blockchain` (`ProcessBlock` result types: `statusExtendChain`, `statusSideChain`...).
3. `netsync/manager.go` — **the session's main course.** Read in this order:
   - `New`, `SyncManager` interface methods (`ProcessBlock`, `QueueBlock`, ...`HandleDonePeerMsg`)
   - `startSync`: choosing sync peer (highest protocol version / services — find `selectPeer`-style logic), sending `getheaders`
   - header handling: `handleNewPeerMsg`→`headersSynced` transition; where `getblocks`/checkpoint logic kicks in (`blockchain.LatestCheckpoint`)
   - block pipeline: `fetchSet`, in-flight maps, `maxRequestedBlocks` and per-peer limits, `handleTxDoneMsg`-style flows
   - `handleDonePeerMsg`: re-requesting missing blocks from other peers (the failure path is where real systems earn their keep)
   - `blocklogger.go` — progress reporting
4. `server.go` — the `OnHeaders`/`OnBlock`/`OnInv` server callbacks and how they forward into `netsync`. Also `NewPeer`/`DonePeer` wiring.
5. Skim `blockchain/checkpoints.go` again in this context: checkpoints gate header acceptance during IBD only.

## Hands-on

1. Fresh data dir + testnet + `--debuglevel=SYNC=debug`: watch the whole IBD. Pause at each phase — locate headers → getblocks/getheaders switch → parallel `getdata` storm → "caught up" log — in `manager.go`.
2. Kill the node mid-sync (Ctrl-C), restart it: find the resume logic (what's persisted? — recall Session 08/09: block index + flat files survive; only the pipeline state is ephemeral).
3. `go test ./netsync/ -short` — note how little is unit-tested here (it's integration-tested); skim `integration/` to see the `rpctest` harness for a full multi-node test.

## Exercises

1. From code: what stops a malicious peer from feeding blocks for a *different* chain? (headers must extend your known best header chain — find the check)
2. Find the in-flight block cap per peer and total; explain what happens when a peer stalls past its deadline (recall Session 11 stall handler).
3. Write down the exact sequence of P2P messages for: (a) initial header sync, (b) one new block at tip, (c) a 2-block reorg announced by `inv`.

## Check yourself

- Why header-first instead of the old "download block 1, then 2, ..."? (DoS resistance + knowing the chain shape before paying for blocks)
- What is `checkpointDoS`-style protection protecting against?
- Where does IBD end and "normal operation" begin in code? (look for `Synced()`/`IsCurrent` gates that re-enable mempool/relay behavior)

**Next session:** `13-mining-and-fee-estimation.md`
