# Differences vs Official Shape Docs

This file matters because the official references are useful, but they are not a complete operator runbook for this exact failure mode.

## Sources compared

- Shape docs homepage: `https://docs.shape.network/`
- Shape node providers page: `https://docs.shape.network/tools/node-providers`
- Superchain registry Shape config: `superchain/configs/mainnet/shape.toml`
- Superchain chain list: `CHAINS.md`
- Optimism operator docs for generic OP Stack node operations

## 1. Official docs emphasize RPC usage more than self-hosted incident recovery

### Official reference

Shape docs clearly document public and Alchemy RPC endpoints.

### Operational reality

That is useful, but it does not tell you how to recover a self-hosted `op-node` plus `op-geth` stack when:
- root storage is full
- Docker cannot rewrite container metadata
- the live datadir is hidden behind a vague folder name
- the old container exits with `127`

### Side note

If you are building production operations around Shape, assume you will need your own runbook beyond the public docs.

## 2. Official registry separates public RPC and sequencer RPC

### Official reference

The Shape entry in Superchain registry includes:
- public RPC: `https://mainnet.shape.network/`
- sequencer RPC: `https://shape-mainnet-sequencer.g.alchemy.com`

### Our recovered setup

The recovered geth command used:
- `--rollup.sequencerhttp=https://mainnet.shape.network`
- `--rollup.historicalrpc=https://mainnet.shape.network`

### Side note

That is a real difference. The live setup recovered successfully with `mainnet.shape.network` for both of those flags, even though the registry presents a distinct sequencer endpoint.

This should be treated as a deliberate compatibility note, not a typo.

## 3. Fork overrides were explicit in the working setup

### Official reference

The registry exposes Shape chain config such as Granite timing. It is the canonical source of chain metadata.

### Our recovered setup

The live containers used explicit override flags:
- `--override.granite=1727370000`
- `--override.holocene=1739880000`
- `--override.isthmus=1774530000`
- `--override.jovian=1778157001`

### Side note

Using explicit overrides reduced ambiguity during troubleshooting. We did not trust embedded defaults after observing signs that bundled config might be stale or incomplete for Shape.

Jovian was the sharpest example of this. In practice, there was no reassuring operator-facing indication that the hardfork would simply be surfaced clearly enough at the exact timestamp when it mattered. That is why the final working setup records the Jovian override explicitly.

## 4. Bootnode handling was explicit

### Official reference

The public Shape docs surfaced in this review did not provide a detailed operator-focused bootnode runbook.

### Our recovered setup

`op-node` used explicit Shape bootnodes.

### Side note

This matters because discovery confusion wastes hours. When in doubt, document the working bootnodes rather than assuming autodiscovery will be enough.

## 5. Version pairing differed from upstream release expectations

### Official reference

Upstream `op-geth` release notes indicate a corresponding `op-node` release for some versions.

### Our recovered setup

- `op-geth:v1.101603.4`
- `op-node:v1.18.0`

### Side note

This pairing was operationally healthy in the final verified state. That does not make it universally correct, but it does make it a proven local reference point.

## 6. Storage layout was the biggest undocumented trap

### Official reference

Generic docs assume you know where your datadir lives.

### Our recovered setup

The live datadir was `/root/Upload`.

### Side note

That name is misleading enough that a rushed operator could delete it during cleanup. This was the single most dangerous operational footgun in the incident.

There was also an operational constraint that the docs did not cover: because of VPS disk-space limits, the snapshot workflow was done by downloading the snapshot to a local PC, unpacking it there, and then uploading the unpacked data to the VPS. That means the uploaded datadir was not just a cache artifact. It represented a time-consuming staged transfer process.

## 7. Zero EL peers was not treated as failure

### Official reference

Many generic node guides teach operators to worry immediately about low peer count.

### Our recovered setup

`net_peerCount = 0` was expected for this Shape mainnet deployment model.

### Side note

Do not import Ethereum mainnet instincts blindly into every OP Stack chain.

## 8. Docs do not replace local truth

The official docs are a starting point.

The actual source of truth for recovery was:
- live container inspect output
- current bind mounts
- current command lines
- current logs
- current JSON-RPC health

When docs and live state disagree, investigate. Do not assume the docs win.
