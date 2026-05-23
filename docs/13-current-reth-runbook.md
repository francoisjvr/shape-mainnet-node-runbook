# Current Shape Mainnet Reth Runbook

This is the current recommended Shape mainnet node path:

- `op-reth` as the execution client
- `op-node` as the rollup node
- explicit runtime config files
- snapshot-first bootstrap
- health checks based on real chain movement, not just container uptime

Shape is an [OP Stack](https://docs.optimism.io/stack/getting-started) rollup. A working node has two clients that talk over the [Engine API](https://github.com/ethereum/execution-apis/blob/main/src/engine/common.md) and authenticate with the same JWT secret:

- **Execution client (`op-reth`)** runs the EVM and stores L2 execution state.
- **Consensus client (`op-node`)** derives the L2 chain from L1 data and feeds payloads into `op-reth`.

This runbook is intentionally **Reth-first**. `op-geth` is legacy and should now be treated as historical recovery context, not the default operator path.

## Recommended operating model

For Shape mainnet, the cleanest setup is:

1. bootstrap from the latest published Reth snapshot
2. keep runtime data, staging data, and config files in separate paths
3. run `op-reth` and `op-node` with explicit config files
4. use `op-node --syncmode=consensus-layer`
5. validate health by checking block movement, block hash parity, and rollup sync status

Shape-specific reality that matters during verification:

- `net_peerCount = 0` on the execution client may still be expected on current Shape mainnet
- zero execution-layer peers is not, by itself, a failure signal
- `safe_l2` and `finalized_l2` may lag behind `unsafe_l2` without meaning the node is broken

## Prerequisites

### Hardware

Minimum practical baseline:

- 8 vCPU or better
- 16 GB RAM minimum
- fast SSD or NVMe storage
- roughly 1 TB of disk if you want comfortable room for the datadir, temporary files, and future growth

Operational advice:

- leave headroom before unpacking a snapshot
- avoid making duplicate copies of a large extracted snapshot unless you have the disk for it
- if free disk is tight, promote the validated snapshot into the runtime path with `mv` instead of `cp`

### External dependencies

You need:

- an Ethereum mainnet RPC endpoint
- an Ethereum mainnet beacon endpoint for blob retrieval
- Docker
- `curl`
- `openssl`
- optionally `jq` for easier JSON inspection
- optionally `aria2c` if you want resumable multi-connection snapshot downloads

### Network ports

This runbook uses these ports:

- `18545` HTTP JSON-RPC for `op-reth`
- `18546` WebSocket RPC for `op-reth`
- `18551` Engine/Auth RPC for `op-reth` on localhost only
- `31303` Reth P2P
- `19545` `op-node` RPC on localhost only
- `17300` `op-node` metrics
- `19222` `op-node` P2P TCP and UDP

These are deliberately non-default so they do not collide with an older geth-based stack during migration.

## Canonical directory layout

Use explicit, role-based paths instead of ambiguous one-off names.

```bash
export RETH_RUNTIME_DIR=/root/shape-mainnet-op-reth-data
export RETH_STAGING_DIR=/root/shape-mainnet-op-reth-staging
export OP_NODE_RUNTIME_DIR=/root/shape-mainnet-op-node-data
export CONFIG_DIR=/root/.shape-mainnet-op-reth

mkdir -p \
  "$RETH_RUNTIME_DIR" \
  "$RETH_STAGING_DIR" \
  "$OP_NODE_RUNTIME_DIR" \
  "$CONFIG_DIR"
```

Path meanings:

- `RETH_RUNTIME_DIR`: live Reth datadir
- `RETH_STAGING_DIR`: temporary location for a downloaded or transferred snapshot before promotion
- `OP_NODE_RUNTIME_DIR`: persistent `op-node` state
- `CONFIG_DIR`: runtime config, rollup config, genesis file, and JWT secret

## Required config artifacts

Keep these runtime files in `CONFIG_DIR`:

- `reth.runtime.toml`
- `rollup.runtime.json`
- `genesis-l2.runtime.json`
- `jwt.hex`

If the snapshot bundle ships source files such as `reth.toml`, `rollup.json`, `genesis-l2.json`, or `known-peers.json`, keep a copy of those too:

- `reth.source.toml`
- `rollup.source.json`
- `genesis-l2.source.json`
- `known-peers.source.json`

Keep a clear split between:
- **source artifacts** copied from the snapshot bundle
- **runtime artifacts** that the services actually use

## Fast path: bootstrap from a Reth snapshot

### 1. Download or transfer the latest snapshot

Use the latest published Shape Reth snapshot for the network you want to run.

After extraction, a valid snapshot should look like a real Reth datadir. Expect at minimum:

- `db/`
- `static_files/`
- `blobstore/`
- `invalid_block_hooks/`
- `reth.toml`
- `rollup.json`
- `genesis-l2.json`

Some archives unpack directly into the datadir root. Others create an extra top-level directory. Normalize the layout before startup.

#### Resumable download example

```bash
aria2c \
  --dir="$RETH_STAGING_DIR" \
  --out="shape-mainnet-reth-snapshot.tar.zst" \
  --continue=true \
  --max-connection-per-server=16 \
  --split=16 \
  --min-split-size=64M \
  "$SNAPSHOT_URL"
```

If the transfer is interrupted, rerun the same command. `aria2c` will resume as long as the partial file and its `.aria2` state file are still present.

### 2. Extract and inspect the snapshot

Before starting containers, verify:

- `db/mdbx.dat` exists and is substantial
- `static_files/` contains header, receipt, and transaction ranges in the same general ballpark
- `rollup.json` is for Shape mainnet
- lock files like `db/lock`, `db/mdbx.lck`, or `static_files/lock` are cleanup artifacts, not automatic proof of corruption

If the archive extracted into a nested directory, fix that before launch.

### 3. Promote the validated snapshot into the runtime path

If you extracted into the staging path and disk is tight, move it into place instead of copying it:

```bash
rm -rf "$RETH_RUNTIME_DIR"
mv "$RETH_STAGING_DIR" "$RETH_RUNTIME_DIR"
mkdir -p "$RETH_STAGING_DIR"
```

If you extracted directly into `$RETH_RUNTIME_DIR`, that is fine too.

### 4. Create runtime config files

```bash
cp "$RETH_RUNTIME_DIR/reth.toml" "$CONFIG_DIR/reth.source.toml"
cp "$RETH_RUNTIME_DIR/rollup.json" "$CONFIG_DIR/rollup.source.json"
cp "$RETH_RUNTIME_DIR/genesis-l2.json" "$CONFIG_DIR/genesis-l2.source.json"

cp "$CONFIG_DIR/reth.source.toml" "$CONFIG_DIR/reth.runtime.toml"
cp "$CONFIG_DIR/rollup.source.json" "$CONFIG_DIR/rollup.runtime.json"
cp "$CONFIG_DIR/genesis-l2.source.json" "$CONFIG_DIR/genesis-l2.runtime.json"

if [ -f "$RETH_RUNTIME_DIR/known-peers.json" ]; then
  cp "$RETH_RUNTIME_DIR/known-peers.json" "$CONFIG_DIR/known-peers.source.json"
fi
```

### 5. Generate the shared JWT secret

```bash
openssl rand -hex 32 > "$CONFIG_DIR/jwt.hex"
chmod 600 "$CONFIG_DIR/jwt.hex"
```

## Start `op-reth`

```bash
docker run -d \
  --name shape-mainnet-op-reth \
  --restart unless-stopped \
  --network host \
  -v "$RETH_RUNTIME_DIR":/data \
  -v "$CONFIG_DIR":/config:ro \
  -v "$CONFIG_DIR/jwt.hex":/jwt.txt:ro \
  us-docker.pkg.dev/oplabs-tools-artifacts/images/op-reth:v2.2.2 \
  node \
  --config=/config/reth.runtime.toml \
  --chain=/config/genesis-l2.runtime.json \
  --datadir=/data \
  --http \
  --http.addr=0.0.0.0 \
  --http.port=18545 \
  --http.api=eth,net,web3,debug,txpool,admin \
  --ws \
  --ws.addr=0.0.0.0 \
  --ws.port=18546 \
  --ws.api=eth,net,web3,debug,txpool,admin \
  --authrpc.addr=127.0.0.1 \
  --authrpc.port=18551 \
  --authrpc.jwtsecret=/jwt.txt \
  --port=31303 \
  --rollup.sequencer=https://mainnet.shape.network \
  --rollup.historicalrpc=https://mainnet.shape.network \
  --rollup.disable-tx-pool-gossip \
  --disable-discovery \
  --log.stdout.format=terminal \
  -v
```

## Start `op-node`

Set your L1 endpoints first:

```bash
export L1_RPC_URL=<your-ethereum-mainnet-rpc>
export L1_RPC_KIND=<alchemy|basic|quicknode|infura|...>
export L1_BEACON_URL=<your-ethereum-mainnet-beacon-rpc>
```

Then start `op-node`:

```bash
docker run -d \
  --name shape-mainnet-op-node-reth \
  --restart unless-stopped \
  --network host \
  -v "$OP_NODE_RUNTIME_DIR":/data \
  -v "$CONFIG_DIR":/config:ro \
  -v "$CONFIG_DIR/jwt.hex":/shared/jwt.hex:ro \
  us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.18.0 \
  op-node \
  --l1="$L1_RPC_URL" \
  --l1.rpckind="$L1_RPC_KIND" \
  --l1.beacon="$L1_BEACON_URL" \
  --l2=http://127.0.0.1:18551 \
  --l2.jwt-secret=/shared/jwt.hex \
  --rollup.config=/config/rollup.runtime.json \
  --l2.enginekind=reth \
  --syncmode=consensus-layer \
  --rpc.addr=127.0.0.1 \
  --rpc.port=19545 \
  --metrics.enabled \
  --metrics.port=17300 \
  --p2p.bootnodes=enode://ae31be7c6596f4bc3657f90b65a1569c7b0db10c976649f06fe5cfb26139064d78d22b41aede293aa7358fc336d086051ac7fce78f063dce89e1da19c7b88aee@34.228.226.23:0?discport=30301,enode://0bea78f02c9bee58326b3a612dae63dc5a0193fb3dfc9be687c9757d25606405ef9f87810157598fce35688bdb5b3834dc6283afab6dc5774abbd6c5a7b17cea@3.85.27.162:0?discport=30301 \
  --p2p.listen.tcp=19222 \
  --p2p.listen.udp=19222
```

## Verify the node

Do **not** declare success just because both containers are up.

### 1. Confirm the execution head is moving

```bash
curl -s -X POST http://127.0.0.1:18545 \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

Sample it repeatedly. Convert to decimal if you want a quick human-readable trend line.

### 2. Confirm `eth_syncing`

```bash
curl -s -X POST http://127.0.0.1:18545 \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'
```

`false` is good, but not sufficient on its own.

### 3. Compare latest block against public Shape RPC

```bash
curl -s -X POST http://127.0.0.1:18545 \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["latest",false],"id":1}'
```

Then compare with the public Shape endpoint.

### 4. Check rollup sync status

```bash
curl -s -X POST http://127.0.0.1:19545 \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"optimism_syncStatus","params":[],"id":1}'
```

Interpretation:

- `unsafe_l2` should keep moving
- `safe_l2` and `finalized_l2` may lag
- lagging `safe_l2` or `finalized_l2` is not automatic proof of failure

### 5. Check recent logs

```bash
docker logs --since 10m shape-mainnet-op-reth
docker logs --since 10m shape-mainnet-op-node-reth
```

Good signs:

- `op-node` keeps inserting new unsafe blocks
- payload processing continues every few seconds
- no repeated engine auth failures
- no repeating forkchoice loops that never converge

## Private RPC checklist

A private Shape RPC is good enough when all of these are true:

1. `eth_chainId` and `net_version` return `360`
2. the local latest block keeps moving
3. the local latest block number and hash repeatedly converge to the public Shape RPC
4. `eth_syncing` is `false`
5. `unsafe_l2` keeps advancing in `optimism_syncStatus`
6. containers stay up without restart churn
7. disk has headroom and logs are clean
8. access is actually restricted by firewall, VPN, reverse proxy, or allowlist

Do **not** call it private just because it lives on a random port.

## Retire the old geth lane after cutover

Once Reth has proven stable over repeated samples:

1. confirm `op-reth` and `op-node` are stable
2. confirm local latest block number and hash match public Shape repeatedly
3. confirm `eth_syncing` is `false`
4. archive any legacy config you want to keep
5. stop and remove legacy geth containers
6. update scripts, health checks, and watchers to point at Reth
7. delete the old geth datadir only after you are sure you do not need the rollback copy

## Troubleshooting

### `401 Unauthorized: signature invalid`

The JWT secret does not match between `op-reth` and `op-node`.

### `403 Forbidden: invalid host specified`

Re-check:
- the Auth RPC bind address
- the JWT path mounted into each container
- any proxying or forwarding layer in front of the local services

### Containers stay up but the node never converges

Watch for this pattern:
- containers are running
- `unsafe_l2` moves
- but the execution head stays flat or never reaches public latest head parity

That is not healthy enough.

### `net_peerCount = 0`

On current Shape mainnet, this is **not automatically bad** for the execution client.

### Snapshot layout confusion

If the extracted archive does not contain the expected Reth datadir structure:
- stop
- inspect the extracted tree
- normalize it so `db/`, `static_files/`, `blobstore/`, and the config files are in the live datadir root
- only then start services
