# Post-syncToTip Catch-Up Mode

Fixes #116, #112

## Problem

After `syncToTip()` completes its initial waypoint-based sync, the node enters a serial event loop that processes blocks **one at a time**. Each block triggers a full fork choice cycle (~2s on Polygon mainnet). On Polygon's 2-second block time, this leaves zero margin for catch-up when the node falls behind.

### How the death spiral forms

1. A transient delay occurs (span rotation wait ~12s, heavy block execution, GC pause, node restart)
2. The event loop falls a few blocks behind
3. Each block still takes ~2s to process (fork choice overhead) — matching the chain's production rate
4. New milestones and blocks accumulate faster than they can be consumed
5. Memory grows from queued events, GC pressure increases, commits slow down
6. The node falls further behind with no mechanism to recover

### Observed symptoms

From issue #116 — after a restart, each `syncToTip` cycle processes more waypoints but falls further behind:

| Cycle | waypointsLen | Duration | Age behind head |
|-------|-------------|----------|-----------------|
| 1     | 94          | —        | (restart)       |
| 2     | 142         | 5m00s    | 6m08s           |
| 3     | 159         | 6m30s    | 7m34s           |

From issue #112 — the node intermittently falls behind by thousands of blocks, memory spikes to 106 GiB / 108 GiB limit, and recovery takes hours.

### Root cause

The architecture has two sync modes with vastly different throughput:

| Mode | Throughput | When used |
|------|-----------|-----------|
| `syncToTip()` | Many blocks per fork choice cycle (waypoint batching) | Only on startup |
| Event loop | 1 block per fork choice cycle (~2s each) | After startup, forever |

Once the event loop enters, there is **no path back** to the efficient batching mode, even when the node is minutes behind.

## Solution

Wrap `syncToTip()` + event loop in an **outer retry loop**. When the event loop detects the node has fallen too far behind (tip age > 30s), it breaks out and re-enters `syncToTip()`, which uses waypoint-based batching to efficiently close the gap.

### New control flow

```
Run() {
    setup()                              // once
    for {                                // outer catch-up loop
        syncToTip()                      // batch many blocks per FC cycle
        if SSF { return }
        initialiseCcb()
        needsCatchUp := runEventLoop()   // 1 block per FC cycle
        if !needsCatchUp { return }      // only on ctx cancel / SSF
        log("re-entering syncToTip")     // catch-up detected
    }
}
```

### Changes

All in `polygon/sync/sync.go`:

1. **`catchUpAgeThreshold` constant** (30s) — threshold to trigger catch-up
2. **`lastTipAge` field** on `Sync` struct — updated every `commitExecution()`
3. **`commitExecution()`** — stores `time.Since(tip timestamp)` before each fork choice update
4. **`runEventLoop()`** — extracted from `Run()`, returns `(needsCatchUp bool, err error)`. After processing block events (NewBlock, NewBlockBatch, NewBlockHashes), checks `lastTipAge > threshold`. Milestone events are excluded from the check because they don't call `commitExecution()`.
5. **`Run()`** — outer `for` loop around syncToTip + runEventLoop
6. **`applyNewMilestoneOnTip()`** — bugfix: unconditional `s.lastMilestoneEndBlockNum = endBlock` changed to `max()` to prevent a stale at-tip milestone from lowering the value set by an ahead-of-tip milestone

### Why not check age after milestones?

Milestones don't trigger `commitExecution()` and don't update `lastTipAge`. Checking after a milestone would use a stale value that may not reflect the node's current state.

## Safety analysis

| Concern | Status |
|---------|--------|
| Background download goroutines | Safe — context-aware, events queue up and are skipped as `errAlreadyProcessed` on re-entry |
| `blockRequestsCache` | Safe — goroutines self-clean on completion |
| `lastMilestoneEndBlockNum` | Preserved — persists on the `Sync` struct, monotonically increasing |
| Pending `tipEvents` | Safe — stale events are rejected by `checkNewBlockHeader` (already below CCB root) |
| `store` buffered blocks | Safe — `syncToTip` calls `store.Flush` before each commit |
| Infinite ping-pong risk | Safe — `syncToTip` returns immediately if no milestones available; age check only triggers after a block event, so the event loop must make progress first |
| CCB recreation | Safe — `initialiseCcb` loads ≤512 headers from the new root |

## Verification

```bash
go build ./cmd/erigon          # compiles cleanly
go test ./polygon/sync/...     # all tests pass
```

In production, look for:
- `[sync] node is behind, switching to catch-up mode` — event loop detected lag
- `[sync] re-entering syncToTip for catch-up` — outer loop triggered
- Tip age decreasing rapidly during catch-up (waypoint batching)
- Node returning to event loop and staying at tip
