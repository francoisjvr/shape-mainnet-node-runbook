# Operator Playbook

This page is the blunt version for future-us.

## If the node is down

1. Check disk first.
2. Check whether geth container is exited `127`.
3. Check whether `/root/Upload` is mounted as `/data`.
4. Do not call it healthy just because a container is running.
5. Measure head movement repeatedly.

## If the node is “moving” but feels wrong

Ask these exact questions:
- Is execution `eth_blockNumber` rising?
- Is lag versus public Shape RPC shrinking?
- Is `unsafe_l2` the only thing moving?
- Are logs meaningfully changing or just repeating a failure loop?

If only `unsafe_l2` moves, you probably do **not** have a healthy node.

## If the disk is full

Do not free space by vibe.

Use evidence:
- inspect mounts
- inspect container config
- identify non-live large paths
- keep rollback paths intact

## If someone suggests deleting `/root/Upload`

Refuse first, verify second.

Only reconsider if current container mounts prove it is no longer the active or intended datadir.

## If you need to explain why the upload folder mattered

Because the workflow was not cheap:
- snapshot downloaded to local PC
- snapshot unpacked on local PC
- unpacked datadir uploaded to VPS

The resulting folder name looked temporary, but the recovery value was production-grade.

## If the docs seem incomplete

Assume they may be incomplete.

Especially around:
- hardfork timing clarity
- exact overrides needed in practice
- incident response branching
- what “healthy” really looks like during recovery

## If Jovian is in play

Remember:
- it happened
- the operator-facing indication was not strong enough to rely on memory or vibes
- explicit `Jovian` handling belongs in the runtime and in the docs

## If Reth is the next task

Switch mental models.

You are no longer just preserving service.
You are doing a migration experiment with rollback constraints.

That means:
- isolate ports and paths
- preserve geth until cutover criteria are met
- treat separate repo docs as primary for fast-changing Reth work

## Gold standard for reporting health

A good status report includes:
- local execution head in decimal
- public execution head in decimal
- lag in blocks
- `eth_syncing` status
- `unsafe_l2`, `safe_l2`, `finalized_l2`
- whether values are improving over repeated checks
- one-line interpretation

## Red flags that justify escalation

- root disk full
- geth exited `127`
- execution head flat across repeated checks
- `unsafe_l2` moving while execution stays pinned
- logs repeating forkchoice or payload problems without real head advancement
- anyone proposing deletion of `/root/Upload` without mount proof
