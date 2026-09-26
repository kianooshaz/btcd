# Session 02 — The Wire Protocol: How Bitcoin Speaks

**Duration:** ~2 hours
**Goal:** Understand Bitcoin's P2P message format and btcd's serialization machinery. After this session you can decode raw P2P traffic by hand and know exactly which code does it.

## Core concepts

Every P2P message = 24-byte header + payload:

```
| magic (4) | command (12, zero-padded) | length (4) | checksum (4) | payload... |
```

The checksum is the first 4 bytes of `double-SHA256(payload)`. The magic bytes differ per network (testnet vs mainnet) — that's how a node rejects traffic from the wrong chain.

## Files to read (in this order)

1. `wire/README.md` and `wire/doc.go` — the package philosophy: every message is a struct implementing `Message` with `BtcDecode`/`BtcEncode`.
2. `wire/protocol.go` — protocol constants and service flags (`SFNodeNetwork`, ...), `ProtocolVersion` history. Read the version comments: they're a BIP history lesson.
3. `wire/common.go` — `readVarInt`/`writeVarInt`, `VarBytes`, `readElement`. These little functions are used *everywhere*; make sure you could write `readVarInt` yourself.
4. `wire/message.go` — `MakeEmptyMessage`, the message registry: how a command string like `"version"` becomes a `MsgVersion` struct.
5. `wire/msgversion.go` — the first message of every connection. Read `NewMsgVersion` and its `BtcDecode`.
6. `wire/msgaddr.go` + `wire/msgaddrv2.go` — address gossip; note the v2 version from BIP-155.
7. `wire/invvect.go` — inventory vectors (`InvTypeTx`, `InvTypeBlock`) — how peers announce data without sending it.

## The key abstraction to internalize

`wire` is *consensus-neutral and stateless*. It knows nothing about validity, chains, or peers — it only knows how bytes become structs and back. Compare `wire/msgblock.go` `BtcDecode` (with `WitnessEncoding` vs `BaseEncoding`) to see how SegWit is just "a different wire encoding".

## Exercise: decode by hand

Take this header (mainnet `version` message start):
```
f9 be b4 d9 76 65 72 73 69 6f 6e 00 00 00 00 00 65 00 00 00 35 8d 49 32
```
Decode: magic, command, payload length. Verify the command bytes spell `version`. (`echo` the command field through `xxd -r -p`.)

## Coding exercises

1. Write a small program in a scratch module (or `wire/example_test.go` style) that:
   - builds a `MsgVersion` with `wire.NewMsgVersion`,
   - serializes it with `wire.WriteMessageWithEncodingN` into a bytes.Buffer,
   - hex-dumps it, and decodes it back with `wire.ReadMessageWithEncodingN`.
2. Modify your dump to use the wrong network magic and see `ReadMessage` reject it.
3. Find where *btcd itself* calls `wire.ReadMessageN` (grep in `peer/peer.go`) — that's the bridge between sessions.

## Check yourself

- Why does the header carry a checksum at all when TCP already has checksums? (silent corruption detection is not the goal... think: framing, plus why `inv` messages with garbage must not crash consensus-independent code)
- What is a `varint` and how many bytes does the value 70000 take? (5 — check with the code table in `common.go`)
- Where is the payload length cap enforced, and what is it?

**Next session:** `03-blocks-transactions-merkle.md`
