# Incidents and Postmortems

## 2026-05-21 to 2026-05-22: restart failure, full disk, broken geth container

### Summary

The Shape mainnet node failed to come back cleanly after restart attempts. The immediate blocker was full root storage, and the secondary blocker was that the existing `shape-mainnet-op-geth` container was broken and exited with code `127`.

### Impact

- local geth RPC was unavailable
- op-node could not make useful forward progress because execution was down
- restart attempts failed until storage pressure and container integrity were both addressed

### Symptoms

- Docker restart returned `no space left on device`
- root filesystem was at 100 percent usage
- `shape-mainnet-op-geth` later showed `Exited (127)`
- geth RPC on `8545` refused connections

### Root cause

Primary operational root cause:
- root filesystem exhaustion

Secondary root cause:
- broken geth container state that required recreation

### What made this incident dangerous

The folder named `/root/Upload` looked disposable at first glance, but container inspection showed it was actually mounted as `/data` for the live geth node.

Deleting it would have destroyed the live chain state and created a much larger recovery problem.

There was also a practical snapshot-handling constraint behind this. Because the VPS did not have comfortable spare disk for the whole workflow, the snapshot was downloaded to a local PC first, unpacked there, and only then uploaded to the VPS as prepared data. That made the uploaded state more valuable than a normal temporary download and increased the cost of any accidental deletion.

### Corrective actions taken

1. confirmed live container mounts
2. confirmed `/root/Upload` was the active geth datadir
3. identified old Reth experiment directories as safe cleanup targets
4. removed only those old Reth directories
5. backed up geth container inspect output
6. removed and recreated only the broken geth container
7. restarted geth, then restarted op-node
8. verified catch-up and final healthy state

### Verification after fix

- local block matched public Shape RPC
- `eth_syncing = false`
- geth logs showed progress
- op-node logs showed payload processing

### Preventive actions

- check disk before restart attempts
- keep a dedicated runbook instead of relying on memory
- preserve inspect backups before recreating containers
- treat vague folder names with suspicion until mounts are verified
- keep future Reth experiments isolated from the live geth datadir
- explicitly document hardfork timestamps such as Jovian instead of assuming upstream docs will make them obvious at the moment they matter
