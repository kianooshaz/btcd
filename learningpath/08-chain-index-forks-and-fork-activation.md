# Session 08 — Block Index, Fork Choice, and Soft-Fork Activation

**Duration:** ~2 hours
**Goal:** Understand how btcd remembers every block header it has ever seen, how it picks the "best" chain among forks, and how consensus rules change over time via version bits.

## Core concepts

- The **block index** = an in-memory tree of *all* known headers (validated or not), each with status flags (data stored? fully validated? valid fork candidate?).
- **Fork choice rule** = most cumulative work wins, not longest chain. Side chains that lose are kept — they can win later.
- **Soft forks** activate through **version bits** (BIP-9), **threshold states** (`defined → started → locked in → active`), per-rule and per-window.

## Files to read

1. `blockchain/blockindex.go` — `blockNode` (note: btcd calls index entries *nodes*), `addBlockNode`, `modFlags` (`statusDataStored`, `statusValid`, `statusValidateFailed`...), ancestor/locate helpers, `serialize/deserializeBlockIndex` (the on-disk format — note it's big-endian unlike everything else, with a comment explaining the mistake and backward compat).
2. `blockchain/chainview.go` — the fork view: an efficient "current best chain" slice supporting branch points; used by everyone who asks "what's on the active chain at height X".
3. `blockchain/chain.go` — revisit with new eyes: `bestChain`, `reorganizeChain` (disconnect side-chain? no — the reverse: disconnect best, connect fork), `IsCurrent`, `locateInventory`.
4. `blockchain/difficulty.go` — `CalcNextRequiredDifficulty` from a `blockNode`: the 2016-window math and the *testnet* special case.
5. `blockchain/thresholdstate.go` and `versionbits.go` — the BIP-9 state machine: `ThresholdState`, window math (`MinerConfirmationWindow`, `RuleChangeActivationThreshold`), and where `Chain.Params` names deployments (`chaincfg/params.go` → `Deployments` map; find the SegWit and CSV deployments).
6. `blockchain/checkpoints.go` — embedded checkpoints: what they do and *don't* enforce (they're a DoS mitigation, not trust).
7. `blockchain/notifications.go` — the `NotificationType` list: what the rest of the node (mempool, mining, RPC) learns when blocks connect/disconnect.

## Hands-on

1. `go test ./blockchain/ -run TestChainView -v` — fork/reorg scenarios in miniature.
2. `go test ./blockchain/ -run TestThresholdState -v` — watch a fake deployment march through states block by block.
3. In `chaincfg/params.go`, list every key in `TestNet3Params.Deployments` and read one deployment's `Start`/`End`/`RuleChangeActivationThreshold` numbers.
4. Sketch the `blockNode` status-flag lifecycle from first header receipt to fully-connected block.

## Exercises

1. From code: what happens when a *lower-work* block arrives? (indexed, maybe fetched later, stored — name the functions) What when a *higher-work* side chain finally beats the tip? (`reorganizeChain`)
2. Find where the *buried* deployments (BIP-34/65/66 — height-activated) are checked vs BIP-9 ones. Two different mechanisms for the same idea.
3. Grep `MedianTime` and explain why median time, not node time, drives BIP-113.

## Check yourself

- Why keep losing forks at all? (a late block can create a winning fork; also orphan-race handling for miners)
- What's the difference between `statusValid` and `statusValidFork`?
- If a BIP-9 deployment expires (`Failed`), can it come back? What does the code say? (`ThresholdStateFailed`, `invalidateCircle`-style reset in new windows)

**Next session:** `09-database-layer.md`
