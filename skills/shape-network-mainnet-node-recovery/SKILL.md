---
name: shape-network-mainnet-node-recovery
description: Use when operating, recovering, documenting, or sanity-checking a self-hosted Shape Network mainnet node built on op-node plus op-geth. Shape-specific notes, catch-up expectations, doc mismatches, and recovery pitfalls are included.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [shape-network, shape-mainnet, op-node, op-geth, recovery, runbook, docker, rollup]
    related_skills: [shape-ai-agent, hermes-agent]
---

# Shape Network Mainnet Node Recovery

## Overview

This skill is specifically for **Shape Network mainnet**, not generic Ethereum and not generic OP Stack advice.

Use it when a self-hosted Shape mainnet node needs to be:
- checked for health
- restarted carefully
- recovered after outage or partial corruption
- compared against official Shape docs
- documented so future operators do not repeat the same mistakes

This skill reflects a real recovered Shape mainnet stack using:
- `op-geth`
- `op-node`
- Docker
- host networking
- a preserved uploaded datadir mounted from `/root/Upload`

The biggest lesson is simple: on Shape, **live operational truth beats assumptions**. Do not infer too much from generic docs, peer-count expectations, or vaguely named folders.

## When to Use

Use this skill when:
- the user says the Shape mainnet node is down, stalled, or behind
- `op-node` and `op-geth` need health verification
- Docker restart fails or the node does not come back cleanly
- you need to decide whether a folder is safe to delete
- you need to document the working Shape mainnet setup for future reuse
- the node appears alive but you need to tell the difference between real catch-up and fake movement
- a fork-era mismatch like Jovian might be involved

Do not use this skill for:
- Shape Sepolia unless you are deliberately borrowing only the reasoning pattern
- generic Ethereum mainnet geth advice
- Reth-first migration procedures that need their own dedicated workflow

## Known-good Shape-specific runtime reference

This was the final healthy reference point.

### op-geth
- image: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-geth:v1.101603.4`
- network mode: `host`
- restart policy: `unless-stopped`
- critical mount: `/root/Upload:/data`
- JWT mount: `/root/shape-mainnet-geth/jwt.hex:/jwt.txt:ro`

Important flags:
- `--op-network=shape-mainnet`
- `--datadir=/data`
- `--http.port=8545`
- `--ws.port=8546`
- `--authrpc.port=8551`
- `--rollup.sequencerhttp=https://mainnet.shape.network`
- `--rollup.historicalrpc=https://mainnet.shape.network`
- `--rollup.disabletxpoolgossip=true`
- `--syncmode=full`
- `--history.transactions=0`
- `--history.logs=0`
- `--override.granite=1727370000`
- `--override.holocene=1739880000`
- `--override.isthmus=1774530000`
- `--override.jovian=1778157001`

### op-node
- image: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.18.0`
- network mode: `host`
- restart policy: `unless-stopped`
- data mount: `/root/shape-mainnet-geth/opnode-data:/data`
- JWT mount: `/root/shape-mainnet-geth/jwt.hex:/shared/jwt.hex:ro`
- config mount: `/root/.shape-mainnet-geth-config:/config:ro`

Important flags:
- `--network=shape-mainnet`
- `--l2=http://127.0.0.1:8551`
- `--l2.enginekind=geth`
- `--syncmode=consensus-layer`
- `--rpc.port=9545`
- `--metrics.port=7300`
- explicit Shape `--p2p.bootnodes=...`
- `--override.granite=1727370000`
- `--override.holocene=1739880000`
- `--override.isthmus=1774530000`
- `--override.jovian=1778157001`

## Non-negotiable Shape-specific facts

1. **`/root/Upload` may be the live geth datadir.**
   Never delete it before verifying mounts.

2. **Shape mainnet execution peer count can be zero and still be normal.**
   Do not treat `net_peerCount = 0` as automatic failure on current Shape mainnet.

3. **Jovian must be treated explicitly.**
   In practice there was not enough reassuring operator-facing indication that Jovian would simply be obvious at the exact timestamp that mattered.

4. **A moving `unsafe_l2` is not enough.**
   The real question is whether execution `eth_blockNumber` advances and whether lag to trusted public RPC actually improves.

5. **The uploaded datadir was expensive to prepare.**
   Because of VPS disk limits, the snapshot workflow was:
   - download snapshot to local PC
   - unpack on the PC
   - upload unpacked data to VPS

   That means the uploaded state is not disposable scratch data.

## Fast health workflow

Run checks in this order.

### 1. Container state
```bash
docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}' | grep 'shape-mainnet-op-'
```

What you want:
- both containers up

High-signal failure:
- `shape-mainnet-op-geth` shows `Exited (127)`

### 2. Disk first
```bash
df -h /
docker system df
```

Interpretation:
- if root is effectively full, stop there and treat storage as the primary blocker
- do not spam restarts into a full disk

### 3. Confirm live datadir mount
```bash
docker inspect shape-mainnet-op-geth --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Interpretation:
- if `/root/Upload -> /data` appears, `/root/Upload` is live production state

### 4. Local execution head
```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://127.0.0.1:8545
```

Convert to decimal if reporting to the user.

### 5. Public comparison
```bash
LOCAL=$(curl -s -H 'Content-Type: application/json' -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' http://127.0.0.1:8545 | python3 -c 'import sys,json; print(int(json.load(sys.stdin)["result"],16))')
PUBLIC=$(curl -s -H 'Content-Type: application/json' -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' https://mainnet.shape.network | python3 -c 'import sys,json; print(int(json.load(sys.stdin)["result"],16))')
echo "local=$LOCAL public=$PUBLIC lag=$((PUBLIC-LOCAL))"
```

### 6. Sync state
```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' \
  http://127.0.0.1:8545
```

### 7. Rollup sync status
```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"optimism_syncStatus","params":[],"id":1}' \
  http://127.0.0.1:9545
```

### 8. Logs
```bash
docker logs --tail 100 shape-mainnet-op-geth
docker logs --tail 100 shape-mainnet-op-node
```

## Catch-up expectations for Shape specifically

These are practical operator expectations, not guarantees.

### If recovering from a good uploaded Shape datadir

Expected phases:

1. **First few minutes**
- geth becomes runnable
- RPC on `8545` answers again
- `eth_syncing` may still be active
- op-node may still lag even after execution RPC is back

2. **Next several minutes**
- local execution head should visibly move upward
- geth logs should show fresh import activity
- op-node should stop looking wedged and start processing payloads again

3. **Tens of minutes to a few hours depending on backlog**
- local head should move toward public Shape RPC
- `unsafe_l2` should approach current head
- `safe_l2` and `finalized_l2` can still trail without implying failure

4. **Healthy steady state**
- local execution head matches or nearly matches public Shape RPC
- `eth_syncing = false`
- logs show normal incremental processing rather than startup loops

### Real observed benchmark from the successful recovery

Sampled head movement over roughly one minute:
- `28,521,198`
- `28,521,248`
- `28,521,322`
- `28,521,592`
- `28,521,661`

Interpretation:
- once the fix was real, obvious movement appeared quickly
- if the head stays flat for a long time while public Shape RPC keeps moving, that is not normal catch-up

### Healthy convergence vs fake movement

Healthy but still converging:
- block numbers keep rising
- logs keep changing meaningfully
- lag to public RPC shrinks across repeated samples

Stuck but pretending to be alive:
- same execution head over repeated checks
- repetitive logs with no real advancement
- op-node churns but lag does not improve
- `unsafe_l2` moves while execution stays pinned

## Recovery procedure that actually worked

### Step 1: inspect before touching anything
- confirm container states
- confirm disk usage
- confirm mounts
- confirm whether `/root/Upload` is live

### Step 2: identify only safe cleanup targets

Safe means:
- not mounted into the current live stack
- not referenced by container config
- not needed for rollback

In the real recovery, these old Reth experiment dirs were safe to remove:
- `/root/shape-mainnet-reth-fresh-data`
- `/root/shape-mainnet-reth-fresh-download`

### Step 3: back up inspect output before destructive container work
```bash
mkdir -p /root/service-backups
docker inspect shape-mainnet-op-geth > /root/service-backups/shape-mainnet-op-geth.$(date +%Y%m%d-%H%M%S).inspect.json
```

Why:
- preserves exact image, args, mounts, network mode, restart policy

### Step 4: free disk safely if needed
- never delete `/root/Upload` casually
- never remove JWT or live op-node data

### Step 5: retry and reassess the failure mode

If geth still fails with:
- `exec: "geth": executable file not found in $PATH`
- `Exited (127)`

then disk was not the only problem and the container itself may be broken.

### Step 6: recreate only the broken geth container
High-level rule:
- delete the broken container metadata
- preserve the mounted datadir
- recreate using the same known-good image, args, mounts, network mode, restart policy

### Step 7: restart in the correct order
1. geth first
2. verify Engine API availability
3. op-node second

### Step 8: verify with repeated sampling
Do not trust one sample.

Track:
- local execution head in decimal
- public Shape execution head in decimal
- lag
- `eth_syncing`
- `unsafe_l2`
- `safe_l2`
- `finalized_l2`
- recent log changes

Repeat every 2 to 5 minutes at least 3 times.

## Shape docs vs operational reality

Important differences to remember:

### 1. Official docs are not enough for incident response
Shape docs are useful, but they do not tell you how to recover a node from:
- full root disk
- Docker metadata write failure
- a misleading datadir folder name
- a broken geth container exiting `127`

### 2. Sequencer and historical RPCs in the recovered stack used `mainnet.shape.network`
Even though the Superchain registry exposes a distinct sequencer RPC, the recovered working geth used:
- `--rollup.sequencerhttp=https://mainnet.shape.network`
- `--rollup.historicalrpc=https://mainnet.shape.network`

### 3. Explicit fork overrides mattered
Do not assume embedded defaults are enough.

### 4. Jovian needed to be documented deliberately
There was no comfortable operator-facing sense that Jovian timing would be obvious exactly when needed, so explicit override documentation mattered.

### 5. Storage workflow was nontrivial
Because of VPS disk limits, the snapshot was downloaded locally, unpacked locally, then uploaded unpacked to the VPS.

## Common pitfalls

1. **Deleting `/root/Upload` because the name looks temporary**
   This is the worst mistake in this workflow.

2. **Treating zero peers as the main bug**
   On current Shape mainnet, that can be expected.

3. **Calling the node healthy because `unsafe_l2` moves**
   Execution head movement matters more.

4. **Restarting blindly before checking disk**
   If disk is full, restarts mostly create noise.

5. **Assuming full convergence is instant**
   RPC can return before the node is fully caught up.

6. **Waiting forever on a flat head**
   If execution is not moving across repeated samples, do not call that normal catch-up.

7. **Assuming docs always mention active hardfork timing clearly enough**
   Jovian was the counterexample.

## Verification checklist

- [ ] Confirmed whether `/root/Upload` is mounted as `/data`
- [ ] Checked root disk before restart work
- [ ] Checked both container states
- [ ] Checked local execution head
- [ ] Compared local head vs public Shape RPC
- [ ] Checked `eth_syncing`
- [ ] Checked `optimism_syncStatus`
- [ ] Read recent `op-geth` and `op-node` logs
- [ ] Interpreted results over repeated samples, not one snapshot
- [ ] Reported Shape block numbers in decimal
- [ ] Did not equate zero peers with failure automatically
- [ ] Treated this as **Shape Network specific** rather than generic node advice
