# Architecture

## Components

The recovered Shape mainnet stack has four practical layers:

1. **Persistent execution state**
   - live geth datadir: `/root/Upload`
2. **Execution client**
   - `shape-mainnet-op-geth`
3. **Rollup node**
   - `shape-mainnet-op-node`
4. **Upstream dependencies**
   - L1 execution RPC
   - L1 beacon RPC
   - Shape public RPC used for comparison and some rollup endpoints in the working setup

## Data flow

```text
Alchemy L1 execution RPC ─┐
                          ├─> op-node ──Engine API──> op-geth ──> local JSON-RPC users
Alchemy L1 beacon RPC  ───┘

Persistent geth chain state <── /root/Upload mounted at /data
```

## Dependency notes

- `op-node` is not useful if `op-geth` is down
- `op-geth` can be healthy while `op-node` still catches up derivation state
- L1 execution and L1 beacon health matter for the whole rollup stack
- the geth container is disposable, the datadir is not

## Failure domains

### Storage failure

Symptoms:
- Docker restart failures
- `no space left on device`
- inability to write metadata

### Container integrity failure

Symptoms:
- `Exited (127)`
- binary or entrypoint failure despite healthy underlying datadir

### Upstream dependency failure

Symptoms:
- L1 request errors
- op-node stalls despite healthy local execution

### Misconfiguration

Symptoms:
- fork mismatch behavior
- missing bootnode behavior
- engine auth failures
- wrong datadir mount
