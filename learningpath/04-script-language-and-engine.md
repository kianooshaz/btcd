# Session 04 — Script: Bitcoin's Little Virtual Machine

**Duration:** ~2.5 hours
**Goal:** Understand Bitcoin Script (the language), btcd's tokenizer/opcodes, and how `txscript.Engine` executes scripts to decide "can this output be spent?". This is one of the two hardest sessions — take your time.

## Core concepts

Script = a list of opcodes+data, executed left to right on a stack. "Spendable" = script evaluates to a non-empty stack with `true` on top. Standard templates: P2PKH, P2SH, P2WKH, P2WSH, P2TR (taproot).

## Files to read (in order)

1. `txscript/README.md` and `txscript/doc.go` — package philosophy: `txscript` is consensus code, so it mirrors Bitcoin Core's script behavior exactly, quirks included.
2. `txscript/opcode.go` — the opcode table. Don't memorize; understand the shape: each opcode has a name, a byte, and an execution function. Find `OP_DUP`, `OP_HASH160`, `OP_CHECKSIG`, `OP_CHECKMULTISIG` and read their functions.
3. `txscript/stack.go` — the stack, with all the overflow/underflow guards. Note where consensus limits are enforced (`MaxStackSize`, altstack).
4. `txscript/scriptnum.go` — how numbers are encoded (little-endian, sign bit in the top byte), why `OP_ADD` operands are limited, and the consensus/policy distinction in `MakeScriptNum`.
5. `txscript/tokenizer.go` — how a byte string is split into opcodes/pushes; where `MAX_SCRIPT_SIZE` and push-size limits bite.
6. `txscript/engine.go` — **the main event.** Read `NewEngine`, `Execute`, and step through: scriptSig executes, then scriptPubKey; P2SH redemption is a second pass; SegWit runs the witness program instead. Find `checkSignatureEncoding`, `checkPubkeyEncoding` — policy-vs-consensus strictness.
7. `txscript/script.go` and `scriptbuilder.go` — the *construction* API (what wallets use): `PayToAddrScript`, `NewScriptBuilder`.
8. `txscript/standard.go` — `ExtractPkScriptAddrs`, script classes, `IsPayToScriptHash`, etc. This is the file you'll use most often in application code.

## Hands-on: execute scripts

Write a scratch program:

```go
// P2PKH spend evaluation
scriptSig := ... // a signature + pubkey push (from a real testnet tx)
pkScript := ... // the output script being spent
vm, _ := txscript.NewEngine(pkScript, btcutil.NewTx(msgTx), 0,
    txscript.ScriptFlags(txscript.ScriptBip16|txscript.ScriptVerifyWitness), nil, nil, -1)
err := vm.Execute()
```

Use `txscript.NewScriptBuilder().AddOp(txscript.OP_DUP)...` to build a P2PKH script and print it with `txscript.DisasmString`.

## Exercises

1. `go test ./txscript/ -run TestEngine` — read 3–5 cases in `engine_test.go`; they're written as tables and are the best Script documentation in the repo.
2. Craft these with `NewScriptBuilder` and `DisasmString` them: P2PKH, a `OP_RETURN` data carrier, a 2-of-3 multisig.
3. Find in `engine.go` where `OP_CHECKMULTISIG` pops the extra dummy element — the famous off-by-one quirk of Bitcoin.
4. Grep `ScriptVerify*` flags and map each to a BIP (BIP-62, BIP-147, ...).

## Check yourself

- What's the difference between a consensus failure and a policy failure when validating a script?
- Why does `OP_CHECKMULTISIG` consume n+1 stack items?
- What makes P2SH special: why does `BtcDecode`... sorry, why does *evaluation* of P2SH need a second pass?

**Next session:** `05-signatures-sighash-and-crypto.md`
