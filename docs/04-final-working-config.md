# Final Working Config

This is the most important document in the repo.

It records the final known-good runtime characteristics of the Shape mainnet node after recovery.

## Last verified

- Date: `2026-05-22`
- Status: healthy
- Execution lag vs public Shape RPC: `0` blocks

## op-geth

### Image

```text
us-docker.pkg.dev/oplabs-tools-artifacts/images/op-geth:v1.101603.4
```

### Runtime characteristics

- network mode: `host`
- restart policy: `unless-stopped`
- bind mounts:
  - `/root/shape-mainnet-geth/jwt.hex:/jwt.txt:ro`
  - `/root/Upload:/data`

### Effective command line

```text
--op-network=shape-mainnet
--datadir=/data
--http
--http.addr=0.0.0.0
--http.port=8545
--http.api=eth,net,web3,debug,txpool
--ws
--ws.addr=0.0.0.0
--ws.port=8546
--ws.api=eth,net,web3,debug,txpool
--authrpc.addr=127.0.0.1
--authrpc.port=8551
--authrpc.jwtsecret=/jwt.txt
--authrpc.vhosts=*
--port=30303
--verbosity=3
--nat=extip:<SERVER_PUBLIC_IP>
--rollup.sequencerhttp=https://mainnet.shape.network
--rollup.historicalrpc=https://mainnet.shape.network
--rollup.disabletxpoolgossip=true
--syncmode=full
--history.transactions=0
--history.logs=0
--override.granite=1727370000
--override.holocene=1739880000
--override.isthmus=1774530000
--override.jovian=1778157001
```

### Why these flags mattered

- `--datadir=/data` mattered because the live chain state was mounted from `/root/Upload`
- `--syncmode=full` was part of the successful branch and should not be changed casually
- `--history.transactions=0` and `--history.logs=0` reduced storage load and were part of the stable setup
- explicit fork overrides were important because relying on embedded defaults was not trusted during debugging
- `--rollup.disabletxpoolgossip=true` aligned with the current Shape deployment model

## op-node

### Image

```text
us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.18.0
```

### Runtime characteristics

- network mode: `host`
- restart policy: `unless-stopped`
- bind mounts:
  - `/root/shape-mainnet-geth/opnode-data:/data`
  - `/root/shape-mainnet-geth/jwt.hex:/shared/jwt.hex:ro`
  - `/root/.shape-mainnet-geth-config:/config:ro`

### Effective command line

```text
op-node
--l1=https://eth-mainnet.g.alchemy.com/v2/<REDACTED>
--l1.rpckind=alchemy
--l1.beacon=https://eth-mainnetbeacon.g.alchemy.com/v2/<REDACTED>
--l2=http://127.0.0.1:8551
--l2.jwt-secret=/shared/jwt.hex
--network=shape-mainnet
--l2.enginekind=geth
--syncmode=consensus-layer
--rpc.addr=127.0.0.1
--rpc.port=9545
--metrics.enabled
--metrics.port=7300
--p2p.bootnodes=<SHAPE_BOOTNODES>
--p2p.listen.tcp=9222
--p2p.listen.udp=9222
--override.granite=1727370000
--override.holocene=1739880000
--override.isthmus=1774530000
--override.jovian=1778157001
```

### Why these flags mattered

- `--syncmode=consensus-layer` was part of the successful recovery path
- `--l1.rpckind=alchemy` matched the upstream provider behavior we were actually using
- explicit `--p2p.bootnodes` removed ambiguity about discovery on Shape
- explicit fork overrides prevented silent mismatch with stale or incomplete embedded config

## Ports

- `8545`: geth HTTP RPC
- `8546`: geth WebSocket RPC
- `8551`: Engine API between op-node and op-geth
- `9545`: op-node RPC
- `7300`: op-node metrics
- `30303`: geth p2p
- `9222`: op-node p2p

## Red flags

Treat these as change-controlled values:
- datadir mount path
- JWT secret path
- L1 provider style
- op-node syncmode
- op-geth syncmode
- explicit fork override timestamps
- Shape bootnode list

If any of these drift, compare carefully against both upstream docs and this runbook before restarting the node.
