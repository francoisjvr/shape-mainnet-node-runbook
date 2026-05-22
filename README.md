# Shape Mainnet Node Runbook

Detailed operator runbook for a self-hosted Shape mainnet node using `op-node` and `op-geth`.

This repo exists because the official docs are not enough on their own when the node is sick at 2am and you need exact, working, battle-tested steps.

## Scope

This runbook covers:
- the final known-good Shape mainnet `op-node` + `op-geth` stack
- the exact outage that was caused by disk exhaustion plus a broken geth container
- the recovery path that actually worked
- day-2 operations and health checks
- differences between official Shape references and the real-world setup that ended up stable
- future Reth staging notes

This repo does **not** include:
- secrets
- private provider keys
- SSH credentials
- production-only values that should not be published

## Start here

- [Overview](docs/00-overview.md)
- [Final working config](docs/04-final-working-config.md)
- [Health checks](docs/06-health-checks.md)
- [Recovery runbook](docs/07-recovery-runbook.md)
- [Differences vs official Shape docs](docs/09-differences-vs-official-shape-docs.md)
- [Incident and postmortem](docs/08-incidents-and-postmortems.md)
- [Reth migration staging notes](docs/11-reth-migration-staging-notes.md)

## Final verified state

Last verified: `2026-05-22`

- `op-geth` image: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-geth:v1.101603.4`
- `op-node` image: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.18.0`
- execution head matched public Shape RPC
- `eth_syncing = false`
- `net_peerCount = 0` was expected for this deployment shape
- preserved geth datadir lived at `/root/Upload`

## Why this repo matters

The recovery only succeeded after separating **documentation assumptions** from **what the live node was actually doing**.

The key lessons were:
- `/root/Upload` was not a random upload folder. It was the live geth datadir.
- `no space left on device` was the immediate reason Docker could not restart the stack.
- after free space was recovered, `shape-mainnet-op-geth` still failed because the container itself was broken and had to be recreated
- Shape-specific fork overrides and bootnode handling mattered
- the public RPC is useful for comparison, but not sufficient as the only source of truth

## Repository layout

```text
shape-mainnet-node-runbook/
├── README.md
├── docs/
├── config/
├── appendices/
├── templates/
└── .github/
```

## Safety rules

1. Never commit secrets.
2. Never publish JWT contents.
3. Never publish provider API keys.
4. Never delete `/root/Upload` unless you are intentionally abandoning the current geth state.
5. Before restarting anything, check disk space first.

## Reference sources

Primary official references used for the comparison sections:
- Shape docs: `https://docs.shape.network/`
- Shape node provider page: `https://docs.shape.network/tools/node-providers`
- Superchain registry Shape config: `https://github.com/ethereum-optimism/superchain-registry/blob/main/superchain/configs/mainnet/shape.toml`
- Superchain registry chain list: `https://github.com/ethereum-optimism/superchain-registry/blob/main/CHAINS.md`
- Optimism operator docs: `https://docs.optimism.io/operators/node-operators/`

## Audience

This is written for:
- future us
- any operator inheriting this stack
- anyone trying to understand where official Shape references stop and operational reality begins
