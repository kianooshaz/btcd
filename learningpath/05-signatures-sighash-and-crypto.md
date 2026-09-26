# Session 05 — Signatures, SigHash, and the Crypto Layer

**Duration:** ~2.5 hours
**Goal:** Understand what a signature actually commits to (the sighash), how ECDSA/Schnorr verification works in btcd, and where the crypto layer plugs into script execution.

## Core concepts

A signature doesn't sign "a transaction" — it signs a **serialized subset** of it, chosen by the `sighash` type (`SIGHASH_ALL`, `SINGLE`, `NONE`, `ANYONECANPAY`). The digest algorithm differs by era: legacy → preimage of a modified tx serialized twice-hashed; SegWit (BIP-143) → a commitment built from hashes of parts; Taproot (BIP-341) → yet another, with tagged hashes.

Verification: `ECDSA`: recover-or-verify on the secp256k1 curve. `Schnorr` (BIP-340): simpler math, batch-friendly. `Taproot` (BIP-341/342): key-path spend = Schnorr over tweaked key; script-path = tapscript leaves committed in a merkle tree.

## Files to read

1. `txscript/sighash.go` — **the core file.** `CalcSignatureHash` (legacy), `calcSignatureHash` internals, then `CalcSignatureHash`→BIP-143's `CalcWitnessSigHash` and the segwit v1 `CalcTaprootSignatureHash`. Read the comments — they explain *why* each hash commits to each field.
2. `txscript/sigvalidate.go` — the dispatch: legacy ECDSA, SegWit ECDSA, taproot Schnorr. `SigHashType` handling — note how btcd rejects the `bit 5` bug-compatible nonsense.
3. `txscript/hashcache.go` — the `TxSigHashes` cache: why recomputing BIP-143 midstate for every input is wasteful, and how the engine caches per-tx hashes.
4. `txscript/sign.go` — the *signing* side (wallet-land): `SignTxOutput`, script template discovery, how btcd finds which input type a script is and produces sigs for it.
5. `btcec/README.md`, `btcec/privkey.go`, `btcec/pubkey.go` — key types.
6. `btcec/ecdsa/` — `signature.go`: `Verify`, and (enlightening) the `ECDSARecover`-style code; `btcec/ecdsa/signature_test.go` for usage.
7. `btcec/schnorr/` — `signature.go`, BIP-340: note the tagged hash `TaggedHash` and x-only pubkeys.
8. `txscript/taproot.go` — `ComputeTaprootKeyNoScript`, tapscript leaf hashes, control block math. Skim; full taproot deep-dive is optional extra credit.
9. `btcec/field.go` — skim only, to see that field arithmetic is hand-optimized; do *not* try to fully absorb it today.

## Hands-on

1. Reuse your Session 04 engine program: for a real testnet SegWit input, print the `sighash` by calling `txscript.CalcWitnessSigHash` (or the taproot variant) and the actual signature from the witness. Verify `vm.Execute()` passes.
2. Flip one byte of a **non-committing** field (e.g. the nVersion) and recompute — signature stays valid for legacy ECDSA but breaks under BIP-143. That difference *is* SegWit's malleability fix.
3. Create a key pair with `btcec.NewPrivateKey`, derive the address with `btcutil.NewAddressPubKey`/`NewAddressPubKeyHash`, sign a message, verify with the ECDSA package.

## Exercises

1. Map every input of `calcSignatureHash` to "what attack does including this field prevent?" (txid → replay/re-binding; amount → fee theft by miners; scriptPubKey hash → script swapping...)
2. Read `sigcache.go` — where does the sig cache live, what's the key, what's the memory bound?
3. `go test ./txscript/ -run TestCalcTaprootSignatureHash -v` and skim one test vector.

## Check yourself

- Why does BIP-143 commit to the *amount* of the input being spent?
- What is an "x-only pubkey" and what info does it drop? (the even/odd y parity — recovered from the signature)
- Where in `engine.go` does the hashcache get used?

**Next session:** `06-block-validation-pipeline.md`
