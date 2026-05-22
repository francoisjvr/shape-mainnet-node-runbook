# Reth Migration Staging Notes

This file is intentionally future-facing.

The current healthy stack is geth-based. Do not destabilize it casually. Any Reth work should happen in isolation first.

## Current posture

- production recovery used `op-geth`
- a separate staging directory was created for future uploaded Reth data
- staging path:

```text
/root/shape-mainnet-op-reth-upload
```

## Why stage separately

Because the worst possible migration mistake would be confusing experiment data with the current live geth datadir.

Keep these worlds separate:
- current working geth data: `/root/Upload`
- future Reth staging data: `/root/shape-mainnet-op-reth-upload`

## Migration rules

1. never test Reth by repointing the existing geth container blindly
2. never reuse `/root/Upload` for early migration experiments
3. do not delete the proven geth state until a cutover is verified and reversible
4. record exact client versions and startup flags for every Reth experiment
5. keep rollback to geth simple

## Suggested cutover plan

1. upload unpacked Reth data into the dedicated staging directory
2. validate ownership, structure, and expected size
3. inspect whether the intended Reth client and OP Stack integration expect any path normalization or rename
4. run Reth in a separate, non-destructive staging configuration first
5. verify RPC, sync behavior, engine compatibility, and metrics
6. define rollback before any production swap
7. only then consider replacing the execution client in the main stack

## Unknowns to answer before real cutover

- exact Shape compatibility expectations for the chosen Reth build
- whether additional Shape-specific flags or overrides are required
- how health signals differ from `op-geth`
- whether snapshot/import assumptions differ materially from geth

## Exit criteria for a safe migration

- head progression matches trusted public reference
- op-node interacts cleanly with the execution client
- restart behavior is predictable
- disk growth is acceptable
- rollback to the current geth setup remains easy
