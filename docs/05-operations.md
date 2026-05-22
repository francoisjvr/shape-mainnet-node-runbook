# Operations

## Restart order

If you need to restart services intentionally:

1. check disk first
2. restart geth first if execution is the thing being repaired
3. restart op-node after Engine API is available
4. verify block movement and RPC health

## Before any restart

Run:

```bash
df -h /
docker ps -a --format 'table {{.Names}}	{{.Status}}	{{.Image}}' | grep 'shape-mainnet-op-'
```

Why:
- restarting into a full disk just creates noise
- container state often tells you more than guesswork

## Routine checks

Daily or before maintenance:
- disk usage
- container uptime and restart count
- local block vs public block
- `eth_syncing`
- recent geth and op-node logs

## Safe cleanup policy

Only remove data that is:
- not mounted into the live stack
- not the only copy of chain state
- not needed for rollback

If you are unsure whether a folder is safe, inspect container mounts first.

## Backup policy for config truth

Before changing a live container definition:

```bash
mkdir -p /root/service-backups
docker inspect shape-mainnet-op-geth > /root/service-backups/shape-mainnet-op-geth.$(date +%Y%m%d-%H%M%S).inspect.json
docker inspect shape-mainnet-op-node > /root/service-backups/shape-mainnet-op-node.$(date +%Y%m%d-%H%M%S).inspect.json
```

## What not to overreact to

- `net_peerCount = 0` on this deployment
- `safe_l2` or `finalized_l2` trailing `unsafe_l2` modestly

## What deserves immediate attention

- root disk nearly full
- local RPC refusing connections
- geth exiting repeatedly
- op-node unable to process payloads for a sustained period
- divergence between local execution head and trusted public reference that keeps growing
