# Session 11 — Peer-to-Peer Networking: peer, connmgr, addrmgr

**Duration:** ~2.5 hours
**Goal:** Understand how btcd finds peers, manages connections, drives the message state machine, and relays inventory. After this you can read any P2P conversation in the logs.

## Core concepts

- **addrmgr**: the address database — known addresses, tried/good/bad scores, DNS-seed bootstrap, random selection with anti-eclipse bias.
- **connmgr**: dial/listen state machine, connection limits, ban scores, tor support.
- **peer**: one goroutine-driven state machine per connection: handshake (`version`/`verack`), message loops (in/out), ping/pong keepalive, inventory announcer queue.
- **v2transport**: BIP-324 encrypted transport (newer btcd) — negotiates before `version`.

## Files to read

1. `peer/README.md` and `peer/doc.go` — the best short description of the peer state machine.
2. `peer/peer.go` — **big file (~3000 lines) — read structurally, not linearly:**
   - `Config` and `newPeer`/`peer` struct — every field
   - handshake: `negotiateVersion`/inbound vs outbound paths, services negotiation, `OnVersion` callbacks
   - `inHandler` — the receive loop: read message → dispatch to `MessageListeners` (this is where `server.go` hooks in via callbacks)
   - `outHandler` + `outputInvChan` — the send loop and the inventory announcer (`queueInvMsg`) with batching
   - `shouldHandlePing`/pong round-trip time, stall detection (`stallHandler` — read the deadline table; it's subtle and great Go)
3. `connmgr/connmanager.go` — `New`, `Connect`, `Dial`, the target-conn loop, retry with backoff.
4. `connmgr/dynamicbanscore.go` — decaying ban score. Short, clever.
5. `addrmgr/addrmanager.go` — buckets (`addrIndexBucketSize`), `NeedMoreAddresses`, DNS seeds in `connmgr/seed.go`.
6. `server.go` — reconnect: `OnVersion`/`OnBlock`/`OnTx`/`OnInv`/`OnGetData` handlers registered by the server (grep `peer.NewConfig` / `SetupMessageListeners`). This is where Sessions 02/10/12 meet.

## Hands-on

1. Run your testnet node with `--debuglevel=PEER=trace,CONN=debug`. Watch a full handshake and an `inv`→`getdata`→`block` exchange. Match each log line to a function in `peer.go`.
2. `go test ./peer/ -run TestPeerHandshakes -v` — in-memory pipe-based peer tests; read one test to see how to fake a remote peer with `io.Pipe`.
3. Write a scratch Go program: use `peer` + `connmgr` to connect to your local node, complete the handshake, send a `getaddr`, print the `addr` response. (Look at `peer/example_test.go` — it's nearly complete.)
4. Find where `MsgReject` is sent and what triggers bans vs polite disconnects.

## Exercises

1. Map a `block` message's path: `peer.inHandler` → listener → `server.OnBlock` → `netsync`. Which goroutine boundaries does it cross, and via which channels?
2. Explain the inventory-announcement batching in `queueInv`/`sendInventory`: why wait/flush on a ticker instead of sending per-tx?
3. Find the fee-filter / compact-block related optional messages (`sendheaders`, `feefilter`, `sendcmpct`) in the handshake code and note which are advertised under what conditions.

## Check yourself

- What's the difference between ban score from `dynamicbanscore` and a hard `--banpeer`-style rule?
- Why does each peer get its own stall deadline table per message type? (think: slowloris-style DoS on block download)
- Where does the address manager decide a peer is "good"?

**Next session:** `12-initial-block-download-netsync.md`
