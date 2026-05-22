# Overview

## Node purpose

This is a self-hosted Shape mainnet node built on the OP Stack model:
- `op-node` handles rollup derivation and consensus-side logic
- `op-geth` handles execution
- the node depends on external Ethereum L1 execution and beacon endpoints

## Known-good service pair

- `op-geth:v1.101603.4`
- `op-node:v1.18.0`

This version pairing is important because it was the pairing that was live and healthy after recovery, even though it is not the pairing most obviously implied by upstream release notes.

## Storage and persistence

Critical persistent paths:
- geth datadir: `/root/Upload`
- JWT secret: `/root/shape-mainnet-geth/jwt.hex`
- op-node data: `/root/shape-mainnet-geth/opnode-data`
- op-node config mount: `/root/.shape-mainnet-geth-config`
- future Reth staging dir: `/root/shape-mainnet-op-reth-upload`

## Runtime model

- Docker containers
- host networking
- restart policy `unless-stopped`
- local RPC exposure on standard ports used by this stack

## What healthy looked like on last verification

- execution head matched public Shape RPC
- `eth_syncing` returned `false`
- geth logs showed new chain segments importing
- op-node logs showed payloads being processed successfully
- `safe_l2` and `finalized_l2` lagged `unsafe_l2` slightly, which is normal and not a failure by itself

## What failed during the incident

There were two separate failures:
1. the root filesystem reached 100 percent usage, which stopped Docker from being able to restart containers cleanly
2. the old `shape-mainnet-op-geth` container was itself broken and exited with code `127`, so freeing disk was necessary but not sufficient
