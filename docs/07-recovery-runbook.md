# Recovery Runbook

This is the battle-tested recovery path for the outage that actually happened.

## Non-negotiable safety rule

Before deleting or moving anything, determine whether `/root/Upload` is mounted as the live geth datadir.

In the incident documented here, `/root/Upload` was the live datadir. Deleting it would have destroyed the working chain state and forced a painful re-upload.

## Recovery order

Always move from least destructive to most destructive.

1. inspect
2. confirm storage state
3. confirm mounts and live datadir
4. back up container config
5. free safe disk space only
6. retry
7. recreate broken container only if necessary
8. verify health in multiple ways

## Step 1: confirm container states

```bash
docker ps -a --format 'table {{.Names}}	{{.Status}}	{{.Image}}' | grep 'shape-mainnet-op-'
```

What to look for:
- `Exited (127)` on geth strongly suggests a broken container or entrypoint problem
- if op-node is up but geth is down, op-node errors are usually secondary

## Step 2: confirm root disk usage

```bash
df -h /
docker system df
```

If root is effectively full:
- treat disk pressure as the immediate blocker
- do **not** blindly restart containers over and over

## Step 3: confirm the live datadir mount

```bash
docker inspect shape-mainnet-op-geth --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

What to look for:
- if you see `/root/Upload -> /data`, then `/root/Upload` is live production state

## Step 4: identify safe cleanup targets

Safe means:
- not mounted into the live stack
- not referenced by current container config
- not needed for rollback

In the documented incident, these old Reth experiment directories were safe to remove:
- `/root/shape-mainnet-reth-fresh-data`
- `/root/shape-mainnet-reth-fresh-download`

## Step 5: back up container inspect before any destructive change

```bash
mkdir -p /root/service-backups
docker inspect shape-mainnet-op-geth > /root/service-backups/shape-mainnet-op-geth.$(date +%Y%m%d-%H%M%S).inspect.json
```

Why this matters:
- if container recreation becomes necessary, this preserves the exact image, args, mounts, network mode, and restart policy

## Step 6: free safe disk space

Example pattern:

```bash
rm -rf /root/shape-mainnet-reth-fresh-data /root/shape-mainnet-reth-fresh-download
df -h /
```

Do not remove:
- `/root/Upload`
- JWT files
- op-node live data
- current config mounts

## Step 7: retry and inspect the new failure mode

If disk was the only problem, a restart may now work.

If geth still fails with something like:

```text
exec: "geth": executable file not found in $PATH
```

or exits with `127`, the existing container may be broken even if the image is fine.

## Step 8: recreate the broken geth container

Use the backed-up inspect output as the source of truth.

High-level process:
1. stop op-node if needed to reduce engine spam
2. remove only the broken geth container
3. recreate geth with the same image, args, mounts, host networking, and restart policy
4. ensure `/root/Upload` is still mounted as `/data`
5. start geth first
6. start op-node after Engine API is healthy

Important:
- this is **container recreation**, not datadir recreation
- the chain state lives in the mounted datadir, not in the deleted container metadata

## Step 9: verify recovery

Do not trust a single signal.

Verify all of these:
- RPC on `8545` answers again
- execution block number advances in decimal
- public and local block numbers converge
- `eth_syncing` eventually returns `false`
- geth logs show fresh chain progress
- op-node logs show processed payloads

### What to expect after the stack comes back up

Do not expect every signal to become perfect instantly.

Normal recovery progression:
1. RPC starts responding again
2. execution head starts moving again
3. op-node resumes meaningful payload processing
4. lag to public RPC shrinks over time
5. `eth_syncing` eventually returns `false`
6. `safe_l2` and `finalized_l2` continue to settle behind `unsafe_l2`

### Practical timeframe guidance

For this documented recovery path using a preserved uploaded datadir:
- signs of life should appear quickly once the fix is real
- visible head movement should be apparent within minutes, not half a day of blind hope
- full convergence can still take longer depending on backlog and upstream conditions

The important thing is not demanding instant zero lag. The important thing is seeing real movement.

### What we actually observed

During the successful recovery, sampled block progression over roughly a minute showed:
- `28,521,198`
- `28,521,248`
- `28,521,322`
- `28,521,592`
- `28,521,661`

That is the kind of movement pattern you want to see. It does not have to match those exact numbers, but the pattern should look alive rather than frozen.

## Step 10: remove temporary watchers once stable

If a progress cron or similar watcher was created just for the incident, remove it once the node is back to healthy steady state. Otherwise you keep stale operational noise around forever.

## Incident-specific summary

What actually fixed the outage:
- preserving `/root/Upload`
- freeing disk by deleting only unused Reth experiment data
- backing up container inspect state
- recreating the broken `shape-mainnet-op-geth` container against the same live datadir
- only then restarting `shape-mainnet-op-node`

## Stop conditions

Stop and reassess before proceeding if:
- you are not sure whether a directory is mounted live
- you are about to delete the only copy of chain state
- the current container config is not backed up
- upstream L1 RPC or beacon availability is the real failure instead of local state
