# Incident Timeline

This is the most literal narrative of what happened during the successful Shape mainnet geth recovery.

## Before the outage work

- The Shape mainnet stack was Docker-based.
- The user had uploaded a large preserved geth datadir.
- Because VPS disk limits were awkward, the snapshot workflow was not a trivial one-shot on the server.
- The practical flow was:
  1. download snapshot to local PC
  2. unpack snapshot on the local PC
  3. upload the unpacked datadir to the VPS
- That uploaded state later appeared on the VPS as `/root/Upload`.
- The folder name looked disposable, but it was not disposable.

## Triggering event

A stop/start style recovery was attempted on the live Shape mainnet stack.

The first meaningful hard failure was:
- Docker restart failing with `no space left on device`

That immediately established two things:
- the node was not going to recover through a normal restart
- the storage situation had to be investigated before any further restart churn

## First root cause found

Host-level inspection showed the root filesystem was full.

Practical impact:
- Docker could not create or manage what it needed during restart
- any further restart attempts without cleanup would mostly generate noise

## Most dangerous discovery

Container mount inspection showed:
- `/root/Upload` was mounted into the live geth container as `/data`

This changed the whole safety picture.

What looked like a random upload folder was actually:
- the live geth datadir
- expensive to reconstruct
- already prepared through a local-PC unpack workflow because VPS disk limits made the process awkward

Deleting it would have destroyed the best preserved recovery asset.

## Safe cleanup branch

Unused Reth experiment directories were inspected and confirmed to be separate from the live geth stack.

Those were safe to remove.

That cleanup restored large amounts of free space without touching `/root/Upload`.

## After disk cleanup

Disk pressure was gone, but the node was still not healthy.

New reality:
- `shape-mainnet-op-geth` still failed to start normally
- the container exited with code `127`
- RPC on `8545` was unavailable

This proved the incident had two layers:
1. full disk blocked restart
2. geth container state was also broken and required explicit recreation

## Recovery branch that actually worked

### 1. Preserve evidence first
Before destructive container work:
- capture `docker inspect`
- keep the live datadir untouched
- preserve the exact image, args, mounts, network mode, and restart policy

### 2. Remove only the dead geth container object
The datadir stayed in place.

The container metadata did not.

### 3. Recreate geth with the same known-good runtime shape
That included:
- same image family
- same host networking pattern
- same JWT mount
- same `/root/Upload:/data` bind
- same Shape-specific rollup and override arguments

### 4. Start geth first
Only after geth was sufficiently alive again did op-node become worth restarting.

### 5. Start op-node second
This mattered because op-node depended on the local execution engine being there.

## Earliest positive signs

The recovery was not judged from a single green-looking status line.

The important evidence was repeated head movement.

Observed sequence over roughly a minute:
- `28,521,198`
- `28,521,248`
- `28,521,322`
- `28,521,592`
- `28,521,661`

Interpretation:
- not just “RPC answers”
- not just “container is up”
- actual forward execution progress

## Catch-up phase

During catch-up, there was still a period where:
- execution looked much healthier
- `safe_l2` and `finalized_l2` still trailed `unsafe_l2`

That was acceptable.

The critical distinction was:
- the execution head moved
- lag was improving
- logs showed real work instead of a sterile loop

## Final healthy verification snapshot

The stack was later verified healthy when:
- local execution head matched public Shape RPC exactly
- `eth_syncing = false`
- both containers were up
- logs showed normal processing

Additional context that matters for future readers:
- `net_peerCount = 0` was not treated as a fault here because current Shape mainnet EL peering is intentionally absent

## Documentation lessons extracted from the incident

### Lesson 1
A folder with a bad name can still be the most important production asset on the machine.

### Lesson 2
Full disk can be the first blocker without being the last blocker.

### Lesson 3
After storage is fixed, container recreation can still be required.

### Lesson 4
For Shape specifically, head movement matters more than peer-count superstition.

### Lesson 5
Jovian needed explicit treatment in docs and flags.
There was not enough comfortable operator-facing guidance that the hardfork timing would be obvious exactly when needed.

### Lesson 6
The snapshot transfer path matters operationally.
Because the datadir was downloaded and unpacked on a local PC before upload to the VPS, preserving that state was more important than treating it as cheap temporary upload residue.
