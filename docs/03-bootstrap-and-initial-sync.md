# Bootstrap and Initial Sync

This file describes how to think about a fresh or semi-fresh bring-up, even though the documented incident recovery used an existing uploaded datadir rather than a cold sync.

## Two valid starting points

### Path A: existing preserved datadir

This is the path used in the documented recovery.

Use this when:
- a valid geth datadir already exists
- the folder has been uploaded or preserved externally
- rebuilding from scratch would be slower or riskier

In our case, disk limits on the VPS meant the snapshot workflow had to be done in stages:
- download the snapshot to a local PC first
- unpack it on the PC instead of on the VPS
- upload the unpacked datadir to the VPS afterward

That was slower operationally, but it avoided blowing up the VPS disk during download and extraction.

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

## Catch-up expectations and rough timeframes

These are operator expectations, not protocol guarantees.

### If restoring from an already-useful uploaded datadir

This was the path used in the documented recovery.

Expected phases:

1. **Immediate startup phase: first few minutes**
   - geth container becomes runnable
   - RPC on `8545` starts answering again
   - `eth_syncing` may still be active
   - op-node may still be behind even after geth answers

2. **Visible forward-progress phase: next several minutes**
   - local execution head should start moving upward, not stay frozen
   - geth logs should show fresh chain import activity
   - op-node logs should stop looking stuck and start processing payloads again

3. **Convergence phase: tens of minutes to a few hours depending on backlog**
   - local block number should move toward public Shape RPC
   - `unsafe_l2` should approach the current execution head
   - `safe_l2` and `finalized_l2` can remain behind for a while even when the node is fundamentally healthy

4. **Steady-state healthy phase**
   - local execution head matches or nearly matches public Shape RPC
   - `eth_syncing = false`
   - ongoing logs show normal incremental processing rather than startup failure loops

### What we actually observed in the successful recovery

After the broken geth container was recreated and the stack was back up:
- the node resumed serving RPC first
- block progression became visible within about a minute of sampling
- sampled head movement over roughly one minute went from `28,521,198` to `28,521,248`, then `28,521,322`, then `28,521,592`, then `28,521,661`

That matters because it gives a practical benchmark: after a successful restart from a good uploaded datadir, you should expect to see obvious movement fairly quickly. If the head stays flat for a long period while public Shape RPC continues moving, that is not normal catch-up behavior.

### If doing a true fresh build instead of restoring prepared state

Expect this to take much longer and be much more sensitive to:
- disk headroom
- IOPS
- upstream L1 quality
- snapshot availability or lack of it

For this reason, the runbook strongly prefers restoring useful prepared state when available instead of pretending a cold rebuild is equally cheap.

### Important interpretation rule

Do not confuse these two states:
- **healthy but still converging**
- **stuck and pretending to be alive**

Healthy but still converging usually looks like:
- block numbers keep rising
- logs keep changing in a meaningful way
- gap to public RPC shrinks over time

Stuck usually looks like:
- same head over repeated checks
- repetitive logs with no real advancement
- op-node churn without meaningful reduction in lag
- `eth_syncing` never resolves and the public gap does not improve

## What to treat as abnormal early

- `no space left on device`
- `exec: "geth": executable file not found in $PATH`
- repeated auth issues between op-node and Engine API
- no head movement while public Shape RPC continues advancing

## Hardfork note: Jovian

One of the important lessons from this setup was that Jovian needed to be treated explicitly.

The working setup pinned:
- `--override.jovian=1778157001`

Operationally, there was no comfortable sense that the network would simply advertise this loudly enough at the exact moment an operator needed it. In practice, we treated Jovian as something that needed to be pinned and documented deliberately rather than assumed to be obvious from ambient Shape docs or defaults.
