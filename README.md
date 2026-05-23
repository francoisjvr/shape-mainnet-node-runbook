# Shape Mainnet Node Runbook

Detailed operator runbook for the **current self-hosted Shape mainnet Reth stack**.

This repo exists because the official docs are not enough on their own when the node is sick at 2am and you need exact, working, battle-tested steps.

## Scope

This repo now covers:
- the current known-good **`op-reth` + `op-node`** runtime
- snapshot-first bootstrap for Shape mainnet
- health checks that go beyond "the container is up"
- provider requirements for L1 RPC + beacon access
- cutover and retirement guidance for the old geth lane
- historical recovery context from the earlier geth-era incidents

This repo does **not** include:
- secrets
- private provider keys
- SSH credentials
- production-only values that should not be published

## Start here

Current primary path:
- [Current Reth runbook](docs/13-current-reth-runbook.md)
- [Health checks](docs/06-health-checks.md)
- [Operator playbook](docs/12-operator-playbook.md)

Historical / archival context:
- [Overview](docs/00-overview.md)
- [Architecture](docs/01-architecture.md)
- [Bootstrap and initial sync](docs/03-bootstrap-and-initial-sync.md)
- [Final working config](docs/04-final-working-config.md)
- [Recovery runbook](docs/07-recovery-runbook.md)
- [Differences vs official Shape docs](docs/09-differences-vs-official-shape-docs.md)
- [Incident timeline](docs/10-incident-timeline.md)
- [Reth migration staging notes](docs/11-reth-migration-staging-notes.md)

Fast triage tools:
- [Decision tree](docs/02-decision-tree.md)
- [Incident and postmortem](docs/08-incidents-and-postmortems.md)

## Current verified state

Last verified against the live server: `2026-05-23`

- `op-reth` image: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-reth:v2.2.2`
- `op-node` image: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.18.0`
- `op-node --syncmode=consensus-layer`
- local RPC ports in use: `18545`, `18546`, `18551`, `19545`, `17300`, `19222`
- execution head, `unsafe_l2`, `safe_l2`, and `finalized_l2` were all advancing sanely
- `eth_syncing = false`
- Shape mainnet can still show `net_peerCount = 0` on the execution client without that alone meaning failure

## Why this repo matters

The main lesson from the last round of work was simple:

**the live server had moved on, but the written runbook had not.**

What mattered in practice:
- the current stable path is now **Reth-first**, not geth-first
- the node can look healthy while still being wrong unless you check head movement and public block parity
- snapshot handling matters because interrupted downloads and disk pressure are common real operator problems
- explicit runtime config files are safer than hand-waving about whatever built-in chain defaults happen to exist
- private RPC is not really private unless the ports are actually restricted

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
4. Before restarting anything, check disk space first.
5. Treat this repo as Shape-specific operational truth, not generic OP Stack gospel.
6. Do not call an RPC "private" unless access control is actually in place.
7. Keep old geth-era docs as history, but do not present them as the main operator path anymore.

## Reference sources

Primary official references used for comparison and validation:
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
