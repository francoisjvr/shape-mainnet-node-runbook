# Appendix B: Command Cheatsheet

## Check containers

```bash
docker ps -a --format 'table {{.Names}}	{{.Status}}	{{.Image}}' | grep 'shape-mainnet-op-'
```

## Check mounts

```bash
docker inspect shape-mainnet-op-geth --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

## Check disk

```bash
df -h /
docker system df
```

## Check local block

```bash
curl -s -H 'Content-Type: application/json'   -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'   http://127.0.0.1:8545
```

## Check syncing

```bash
curl -s -H 'Content-Type: application/json'   -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'   http://127.0.0.1:8545
```

## Check rollup sync status

```bash
curl -s -H 'Content-Type: application/json'   -d '{"jsonrpc":"2.0","method":"optimism_syncStatus","params":[],"id":1}'   http://127.0.0.1:9545
```

## Tail geth logs

```bash
docker logs --tail 200 shape-mainnet-op-geth
```

## Tail op-node logs

```bash
docker logs --tail 200 shape-mainnet-op-node
```

## Back up geth container inspect

```bash
mkdir -p /root/service-backups
docker inspect shape-mainnet-op-geth > /root/service-backups/shape-mainnet-op-geth.$(date +%Y%m%d-%H%M%S).inspect.json
```
