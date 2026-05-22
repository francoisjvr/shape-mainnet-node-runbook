# Health Checks

This file is intentionally command-heavy.

Use these checks in order, from cheap to decisive.

## 1. Container status

```bash
docker ps --format 'table {{.Names}}	{{.Status}}	{{.Image}}' | grep 'shape-mainnet-op-'
```

Expected result:
- both `shape-mainnet-op-geth` and `shape-mainnet-op-node` are listed as up

## 2. Disk headroom first

```bash
df -h /
```

Expected result:
- root filesystem is not critically full
- if disk is near 100 percent, stop and treat storage as the primary suspect before restarting containers

## 3. Local execution head

```bash
curl -s -H 'Content-Type: application/json'   -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'   http://127.0.0.1:8545
```

Expected result:
- valid JSON-RPC response with a hex block number

## 4. Convert block number to decimal for humans

```bash
python3 - <<'PY'
import json, sys
obj = json.loads(sys.stdin.read())
print(int(obj['result'], 16))
PY
```

Pipe the JSON-RPC response into the snippet above if you want human-readable output.

## 5. Compare against public Shape RPC

```bash
LOCAL=$(curl -s -H 'Content-Type: application/json' -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' http://127.0.0.1:8545 | python3 -c 'import sys,json; print(int(json.load(sys.stdin)["result"],16))')
PUBLIC=$(curl -s -H 'Content-Type: application/json' -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' https://mainnet.shape.network | python3 -c 'import sys,json; print(int(json.load(sys.stdin)["result"],16))')
echo "local=$LOCAL public=$PUBLIC lag=$((PUBLIC-LOCAL))"
```

Expected result:
- small lag while syncing
- `0` lag when fully caught up

## 6. Is geth still syncing?

```bash
curl -s -H 'Content-Type: application/json'   -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'   http://127.0.0.1:8545
```

Expected result:
- `false` when caught up
- an object while syncing

## 7. Peer count

```bash
curl -s -H 'Content-Type: application/json'   -d '{"jsonrpc":"2.0","method":"net_peerCount","params":[],"id":1}'   http://127.0.0.1:8545
```

Important note:
- `0` peers is not automatically bad on this Shape mainnet setup
- do not treat `0` peers as the root cause unless other signals are bad too

## 8. Rollup sync status

```bash
curl -s -H 'Content-Type: application/json'   -d '{"jsonrpc":"2.0","method":"optimism_syncStatus","params":[],"id":1}'   http://127.0.0.1:9545
```

Expected result:
- `unsafe_l2` advances normally
- `safe_l2` and `finalized_l2` can trail behind a bit without it being an incident

## 9. Geth logs

```bash
docker logs --tail 100 shape-mainnet-op-geth
```

Healthy signs:
- importing or processing new chain segments
- chain head updates continue

Bad signs:
- repeated startup failures
- missing binary or entrypoint errors
- persistent database open failures
- no progress for long periods while public chain keeps advancing

## 10. op-node logs

```bash
docker logs --tail 100 shape-mainnet-op-node
```

Healthy signs:
- successfully processed payloads
- derivation continues

Bad signs:
- endless forkchoice updates with no real forward motion
- repeated upstream connectivity failures
- JWT or engine API auth problems

## Fast operator interpretation

If all of these are true, the stack is probably healthy:
- both containers are up
- local execution head is moving or already aligned with public RPC
- `eth_syncing = false` once caught up
- op-node continues processing payloads
- disk has comfortable free space

If the first thing you see is `no space left on device`, stop there. That is not a secondary symptom. It is often the actual problem.
