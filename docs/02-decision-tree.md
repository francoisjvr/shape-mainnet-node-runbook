# Decision Tree

Use this when the Shape mainnet node looks unhealthy and you need a quick path to the right branch.

## Step 1: Are the containers up?

Check:
```bash
docker ps -a --format 'table {{.Names}}	{{.Status}}	{{.Image}}' | grep 'shape-mainnet-op-'
```

### If both are up
Go to **Step 2**.

### If `shape-mainnet-op-geth` is exited
Go to **Step 1A**.

### If `shape-mainnet-op-node` is exited but geth is up
Go to **Step 1B**.

## Step 1A: geth is down

Check disk first:
```bash
df -h /
docker system df
```

### If root disk is full or nearly full
Interpretation:
- restart noise is not the main problem
- storage is the blocker

Action:
- identify only non-live paths safe to remove
- do **not** touch `/root/Upload` unless mounts prove it is unused

### If disk is fine but geth exited `127`
Interpretation:
- the container itself may be broken
- a plain restart is not enough

Action:
- back up `docker inspect`
- recreate only the broken geth container
- preserve mounts and datadir

## Step 1B: op-node is down but geth is up

Check:
- recent `op-node` logs
- whether Engine API on `127.0.0.1:8551` is reachable indirectly by log behavior or authenticated probe
- whether fork overrides match

Interpretation:
- if geth is healthy and RPC is moving, op-node is likely secondary
- if op-node errors mention fork parsing, think chain config / hardfork mismatch before network blame

## Step 2: Is local execution head moving?

Check local head at least 3 times, 2 to 5 minutes apart.

### If local `eth_blockNumber` rises each time
Go to **Step 3**.

### If local `eth_blockNumber` is flat
Go to **Step 4**.

## Step 3: Head is moving

Compare against public Shape RPC.

### If lag is shrinking
Interpretation:
- healthy catch-up is likely in progress

Action:
- continue monitoring
- do not overreact if `safe_l2` and `finalized_l2` still trail `unsafe_l2`

### If lag is not shrinking
Interpretation:
- the node may be alive but not converging well

Action:
- inspect logs
- verify whether movement is meaningful or tiny compared with public chain growth
- check whether the same unhealthy loop is repeating

## Step 4: Head is flat

Now distinguish fake activity from real recovery.

Check:
- `optimism_syncStatus`
- `docker logs` for both services
- public Shape head

### If `unsafe_l2` moves but execution head is flat
Interpretation:
- not healthy
- consensus-side churn exists, but execution is still pinned
- on Shape this often means engine/state/forkchoice trouble, not normal EL peering trouble

Action:
- do not report this as syncing successfully
- if on geth path, treat as stale snapshot / broken execution branch
- if on Reth path, treat as canonical-head / forkchoice / chain-spec failure until proven otherwise

### If neither `unsafe_l2` nor execution head moves
Interpretation:
- startup is stuck more fundamentally

Action:
- check fork mismatch, config mismatch, storage, and container state

## Step 5: Is peer count zero?

Check:
```bash
curl -s -H 'Content-Type: application/json'   -d '{"jsonrpc":"2.0","method":"net_peerCount","params":[],"id":1}'   http://127.0.0.1:8545
```

### If zero peers on Shape mainnet
Interpretation:
- this is not automatically a fault
- current Shape mainnet execution layer is intentionally not peered

Action:
- stop treating peer count itself as the root cause
- focus on head advancement and engine behavior

## Step 6: Is the folder safe to delete?

Before deleting any large folder, prove whether it is mounted.

Check:
```bash
docker inspect shape-mainnet-op-geth --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

### If folder appears as `/root/Upload -> /data`
Interpretation:
- it is live production state
- deleting it would destroy the preserved geth datadir

Action:
- do not delete it

### If folder is not mounted and not referenced elsewhere
Interpretation:
- it may be eligible for cleanup

Action:
- still verify it is not part of rollback or pending migration before removal

## Step 7: Is this now a Reth problem instead of a geth problem?

Treat it as Reth-specific when:
- the user explicitly wants Reth next
- geth recovery is already documented and stable enough
- the remaining work is migration rather than resurrection
- logs show Reth-side forkchoice, chain-spec, or canonical-head symptoms

Action:
- switch to the separate Reth repo/runbook
- preserve the existing geth state until Reth is demonstrably healthy
