# Bootstrap and Initial Sync

This file describes how to think about a fresh or semi-fresh bring-up, even though the documented incident recovery used an existing uploaded datadir rather than a cold sync.

## Two valid starting points

### Path A: existing preserved datadir

This is the path used in the documented recovery.

Use this when:
- a valid geth datadir already exists
- the folder has been uploaded or preserved externally
- rebuilding from scratch would be slower or riskier

Critical rule:
- verify the existing datadir mount before you do anything destructive

### Path B: true fresh build

Use this only when:
- you intentionally want a fresh node
- you do not need the current uploaded state
- you have the time, bandwidth, and disk to rebuild safely

## Bootstrap checklist

1. confirm host disk size and headroom
2. confirm Docker runtime health
3. confirm paths for JWT, geth datadir, and op-node datadir
4. confirm L1 execution provider
5. confirm L1 beacon provider
6. pin exact client versions
7. pin explicit Shape overrides
8. bring up geth first if using an existing execution state
9. bring up op-node after Engine API is ready
10. compare local block height to a trusted public Shape RPC

## Expected early behavior

During a healthy startup you should expect:
- geth RPC to come alive before the node is fully caught up
- op-node to need healthy L1 access and healthy Engine API access
- `eth_syncing` to return an object before later returning `false`
- `safe_l2` and `finalized_l2` to lag `unsafe_l2`

## What to treat as abnormal early

- `no space left on device`
- `exec: "geth": executable file not found in $PATH`
- repeated auth issues between op-node and Engine API
- no head movement while public Shape RPC continues advancing
