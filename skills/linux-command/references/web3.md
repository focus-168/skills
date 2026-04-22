# web3.md — Web3 节点 / Validator / RPC / 合约部署

## 目录
1. [以太坊节点（Geth / Lighthouse）](#ethereum)
2. [Solana 节点与 Validator](#solana)
3. [Cosmos / Tendermint 节点](#cosmos)
4. [RPC 节点管理与负载均衡](#rpc)
5. [智能合约部署](#contract)
6. [链上数据诊断](#onchain)

---

## 1. 以太坊节点（Geth / Lighthouse）{#ethereum}

### Geth 执行层

```bash
# 节点状态（IPC 连接）
geth attach /data/ethereum/geth.ipc --exec "eth.syncing"
# false = 已同步  {currentBlock, highestBlock} = 同步中

# 同步进度
geth attach /data/ethereum/geth.ipc --exec \
  "var s=eth.syncing; s ? (s.currentBlock/s.highestBlock*100).toFixed(2)+'%' : 'synced'"

# peer 连接数
geth attach /data/ethereum/geth.ipc --exec "net.peerCount"
# < 5 = peer 不足，检查防火墙 30303 端口

# 查看 chainId
geth attach /data/ethereum/geth.ipc --exec "eth.chainId()"

# 实时日志监控
journalctl -u geth -f | grep -E "Imported|Syncing|error|warn"

# 磁盘占用（归档节点 ~17TB，全节点 ~1TB，快照节点 ~500GB）
du -sh /data/ethereum/

# Geth 内存调优
# ExtraArgs 加: --cache 8192 --cache.gc 25 --cache.snapshot 25
```

### Geth 常见问题
```bash
# 磁盘 IO 慢导致同步卡
iostat -x 1 5 | grep -E "vda|sda|nvme"
# await > 20ms = IO 瓶颈，考虑 NVMe

# peer 不足（检查 ENR / bootnode）
geth attach /data/ethereum/geth.ipc --exec "admin.peers.length"
# 手动添加 peer
geth attach /data/ethereum/geth.ipc --exec \
  "admin.addPeer('enode://<pubkey>@<ip>:<port>')"

# 数据库损坏修复 ⚠️
geth db integrity --datadir /data/ethereum
```

### Lighthouse 共识层
```bash
# 同步状态
curl -s http://localhost:5052/eth/v1/node/syncing | python3 -m json.tool

# 节点健康
curl -s http://localhost:5052/eth/v1/node/health

# peer 数量
curl -s http://localhost:5052/eth/v1/node/peer_count | python3 -m json.tool

# validator 状态（需 validator client）
curl -s http://localhost:5062/lighthouse/validators | python3 -m json.tool

# 日志
journalctl -u lighthouse-bn -f | grep -E "INFO|WARN|ERRO"
journalctl -u lighthouse-vc -f | grep -E "attestation|proposal|error"
```

### Validator 运维
```bash
# 检查 attestation 命中率（MEV-boost / 原生）
curl -s http://localhost:5052/eth/v1/validator/duties/attester/<epoch> \
  -H "Content-Type: application/json" \
  -d '["<validator_pubkey>"]'

# 检查是否有 slash 风险（双签检测）
lighthouse account validator slashing-protection export \
  --datadir /data/lighthouse \
  --output /backup/slashing-protection.json

# 导入 validator keys（新节点）🔒 ⚠️
lighthouse account validator import \
  --datadir /data/lighthouse \
  --directory /tmp/validator_keys
```

---

## 2. Solana 节点与 Validator {#solana}

```bash
# 节点状态
solana-validator --ledger /data/solana/ledger monitor

# catchup 进度
solana catchup --our-localhost

# 集群版本信息
solana cluster-version

# 验证者信息
solana validators | grep <identity_pubkey>

# 实时日志
journalctl -u solana-validator -f | grep -E "Error|Warning|vote|leader"

# 查看当前 slot
solana slot

# 磁盘清理（快照保留）⚠️
solana-validator --ledger /data/solana/ledger \
  set-log-filter warn
# 清理旧快照（保留最新 N 个）
ls -t /data/solana/ledger/snapshot/ | tail -n +3 | xargs rm -f

# 硬件要求检查
solana-sys-tuner --username solana   # 🔒 调优系统参数

# Vote account 余额（需保持 > 0.5 SOL）
solana balance <vote_account>
solana vote-account <vote_account>
```

### Solana RPC
```bash
# 测试 RPC 响应
curl -s http://localhost:8899 -X POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"getHealth"}' | python3 -m json.tool

# getSlot
curl -s http://localhost:8899 -X POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"getSlot"}'
```

---

## 3. Cosmos / Tendermint 节点 {#cosmos}

```bash
# 同步状态
<chain>d status | python3 -m json.tool | grep -E "catching_up|latest_block_height"
# catching_up: false = 已同步

# 节点信息
<chain>d status | python3 -m json.tool | grep -E "id|listen_addr|version"

# peers 数量
<chain>d net-info | python3 -m json.tool | grep n_peers

# 添加 peer（state sync 加速同步）
# 编辑 config.toml:
# persistent_peers = "<id>@<ip>:26656"
systemctl restart <chain>d

# Validator 状态
<chain>d query staking validator <operator_address>

# 查看 slashing 信息
<chain>d query slashing signing-info \
  $(<chain>d tendermint show-validator)

# 实时区块生产监控
journalctl -u <chain>d -f | grep -E "committed|WARN|ERR"

# 治理投票（validator 必须参与）
<chain>d query gov proposals --status voting_period
<chain>d tx gov vote <proposal_id> yes \
  --from <key_name> --chain-id <chain_id> --gas auto
```

### Cosmovisor（自动升级）
```bash
# 状态
systemctl status cosmovisor

# 查看当前/升级版本
cat /data/<chain>/cosmovisor/current/bin/<chain>d version
ls /data/<chain>/cosmovisor/upgrades/

# 手动触发升级测试
cosmovisor run version
```

---

## 4. RPC 节点管理与负载均衡 {#rpc}

### Nginx RPC 负载均衡
```nginx
upstream eth_rpc {
    least_conn;
    server 127.0.0.1:8545 weight=1 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8546 weight=1 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

server {
    listen 443 ssl;
    server_name rpc.<domain>;

    location / {
        proxy_pass http://eth_rpc;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;

        # 限速（防滥用）
        limit_req zone=rpc_limit burst=50 nodelay;
        limit_req_status 429;
    }
}
```

### RPC 健康检查
```bash
# 批量检查多个 RPC 节点
RPC_NODES=(
    "http://localhost:8545"
    "http://node2:8545"
    "https://mainnet.infura.io/v3/<key>"
)

for rpc in "${RPC_NODES[@]}"; do
    result=$(curl -s -X POST "$rpc" \
        -H "Content-Type: application/json" \
        -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
        --max-time 3)
    block=$(echo "$result" | python3 -c "import sys,json; print(int(json.load(sys.stdin)['result'],16))" 2>/dev/null)
    echo "$rpc → block: ${block:-ERROR}"
done

# WebSocket 连接测试
wscat -c ws://localhost:8546 \
  -x '{"jsonrpc":"2.0","method":"eth_subscribe","params":["newHeads"],"id":1}'
```

### eth_getLogs 性能优化
```bash
# 查询大区间 log 时限制 block range（避免超时）
# 推荐单次 < 2000 blocks

# 监控 RPC 响应时间
time curl -s -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["latest",false],"id":1}' \
  -o /dev/null
```

---

## 5. 智能合约部署 {#contract}

### Hardhat / Foundry
```bash
# Foundry 编译 + 测试
forge build
forge test -vvv
forge test --match-test testTransfer -vvvv   # 单个测试详细输出

# Gas 报告
forge test --gas-report

# 部署（verify on Etherscan）🔒 ⚠️
forge script script/Deploy.s.sol \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY \
  --broadcast \
  --verify \
  --etherscan-api-key $ETHERSCAN_KEY \
  -vvvv

# 仅模拟（不广播）
forge script script/Deploy.s.sol \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY

# 验证已部署合约
forge verify-contract <contract_address> \
  src/MyContract.sol:MyContract \
  --etherscan-api-key $ETHERSCAN_KEY \
  --chain mainnet

# cast 查询链上数据
cast call <contract_address> "balanceOf(address)(uint256)" <wallet>
cast storage <contract_address> 0    # 查看 slot 0 存储
cast block latest
cast gas-price
```

### 安全检查（部署前必做）
```bash
# Slither 静态分析 📋
slither . --print human-summary
slither . --detect reentrancy-eth,suicidal

# 检查 .env 没有提交 ⚠️
git log --all --full-history -- .env
git ls-files | grep -i "\.env\|secret\|private"
```

---

## 6. 链上数据诊断 {#onchain}

```bash
# 查看 pending 交易
cast rpc eth_getBlockByNumber pending true | \
  python3 -c "import sys,json; txs=json.load(sys.stdin)['result']['transactions']; print(f'Pending txs: {len(txs)}')"

# 查看账户 nonce（卡单排查）
cast nonce <address>
cast rpc eth_getTransactionCount <address> pending

# 加速卡住的交易（替换 nonce）⚠️
cast send \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY \
  --nonce <stuck_nonce> \
  --gas-price <higher_gas_price> \
  <to_address> ""

# mempool 监控（本地节点）
geth attach /data/ethereum/geth.ipc --exec \
  "txpool.status"   # queued = 缺 nonce，pending = 等待打包

# 查看 MEV / Base Fee 趋势
cast basefee
cast rpc eth_feeHistory 10 latest '[]'
```