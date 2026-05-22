# Reth Migration Staging Notes

This document is intentionally a staging bridge, not the complete Reth playbook.

The dedicated Reth path now has its own companion repo and should be treated as the primary working area for upcoming Reth work:
- `shape-mainnet-op-reth-journey`

Use this file to understand how the future Reth work relates to the currently recovered geth stack.

## Why Reth is separate

The geth recovery and the Reth migration are related, but they are not the same problem.

This repo exists to preserve the known-good geth recovery path.

The Reth track exists to answer a different question:
- how to bring up a usable Shape mainnet Reth stack without losing the current geth safety net

## Current staging facts

- A dedicated staging directory was created for later manual upload:
  - `/root/shape-mainnet-op-reth-upload`
- The active geth datadir remained:
  - `/root/Upload`
- The user preference is to try **Reth first** when both Reth and geth are viable options for future Shape experiments.
- The user only wants status reporting on the new Reth process unless they explicitly ask about the old geth stack.

## Hard rule

Do not destroy the working geth stack just because Reth is the next desired state.

Until Reth proves healthy, the current geth recovery should remain the rollback anchor.

## What Reth prep must get right

1. correct binary family
   - prefer `op-reth`, not generic upstream `reth`, unless there is a very deliberate reason otherwise

2. chain identity
   - `op-node --network=shape-mainnet` can be valid while `op-reth --chain shape-mainnet` is not
   - tested behavior showed `op-reth --chain shape` as the built-in Shape chain name

3. hardfork handling
   - explicit `Jovian` treatment mattered
   - relying only on stale built-in defaults was unsafe

4. execution health interpretation
   - `unsafe_l2` movement alone is not enough
   - execution `eth_blockNumber` must move meaningfully

5. isolation
   - Reth work should use separate paths and avoid trampling the current geth datadir

## Reth-specific failure patterns already learned

These are high-signal warnings from previous Shape Reth investigations.

### Bad pattern: fake movement
- `op-node` inserts unsafe payloads
- `unsafe_l2` advances
- execution `eth_blockNumber` stays flat or zero

Interpretation:
- not healthy
- likely canonical-head, forkchoice, chain-spec, or engine-state trouble

### Bad pattern: wrong assumptions about peers
- `net_peerCount = 0` alone is not proof of misconfiguration on Shape mainnet
- if execution remains frozen, focus on engine state rather than mythical missing EL peers

### Bad pattern: snapshot rerestore optimism
- re-extracting the same Reth snapshot is not automatically a fix
- identical re-restore can reproduce the same stuck execution state

### Bad pattern: chain-spec complacency
- newer fork eras can require explicit custom chain-spec updates
- Jovian was the main warning here

## What this repo will continue to do

This geth runbook should retain:
- the stable fallback path
- exact incident history
- precise description of why `/root/Upload` must be preserved until intentionally retired

## Where to continue Reth work

Continue in the companion repo:
- `shape-mainnet-op-reth-journey`

That repo is where the following should evolve rapidly:
- preflight checklist
- chain-spec handling
- upload folder preparation
- compose and port isolation notes
- staged bring-up
- cutover criteria
- rollback criteria
