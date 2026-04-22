---
name: linux-command
description: Web2/Web3 生产环境 Linux 运维技能。凡涉及部署、自动化、运维、监控、故障排查、分布式服务、Docker/K8s、Nginx、systemd、日志分析、网络诊断、区块链节点管理等场景均触发。关键词：deploy、systemctl、docker、kubectl、tail、grep、netstat、iptables、nginx、pm2、ansible、node、validator、RPC、故障、排查、慢、挂了、OOM、502、端口、进程、日志。不要因为问题看起来只是「一条命令」就跳过本技能——生产环境的命令需要安全顺序和上下文。
---

# Linux Command — Web2 / Web3 生产运维

**适用范围**：Ubuntu / Debian / CentOS / RHEL / Alpine，覆盖 Web2 服务器运维与 Web3 节点管理。

你的任务：根据用户描述的场景，**先加载对应 reference 文件，再给出带注释的完整命令序列**。不要只给单条命令，要给可直接执行的操作流程，并标注危险操作。

---

## 第一步：场景路由

收到请求后，识别主场景，加载对应 reference 文件：

| 用户描述关键词 | 加载文件 |
|---|---|
| 部署、发布、上线、rollback、CI/CD、Docker、镜像、容器 | `references/deployment.md` |
| 监控、告警、CPU、内存、磁盘、OOM、性能、慢、卡 | `references/monitoring.md` |
| 故障、挂了、502、503、连不上、排查、日志、error | `references/troubleshooting.md` |
| 分布式、K8s、服务发现、负载均衡、gRPC、消息队列 | `references/distributed.md` |
| 区块链、节点、validator、RPC、钱包、合约、Web3 | `references/web3.md` |

**多场景**：同时加载多个文件，按「排查 → 修复 → 验证」顺序组织命令。

---

## 通用安全原则

在给出任何命令前，检查并标注：

```
⚠️  危险操作  ——  会修改/删除数据，需二次确认
🔒  需要权限  ——  sudo / root 才能执行
📋  建议先备份  ——  执行前快照或导出
🔄  可回滚  ——  说明回滚步骤
```

**生产环境三不原则**：
1. 不裸跑 `rm -rf`，先 `ls` 确认路径
2. 不直接 kill -9，先 `kill -15` 优雅退出
3. 不改配置不备份，`cp nginx.conf nginx.conf.bak.$(date +%Y%m%d%H%M%S)`

---

## 高频命令速查（无需加载 reference）

### 进程管理
```bash
# 查找进程
ps aux | grep <name>
pgrep -a <name>

# 查端口占用
ss -tlnp | grep <port>
lsof -i :<port>

# 优雅重启服务
systemctl restart <service>
systemctl status <service> --no-pager -l
```

### 日志快速查看
```bash
# 实时跟踪
journalctl -u <service> -f --no-pager
tail -f /var/log/<app>/app.log

# 错误过滤
journalctl -u <service> --since "10 min ago" | grep -E "ERROR|WARN|FATAL"
grep -n "ERROR" /var/log/app.log | tail -50
```

### 资源快速确认
```bash
# 内存/CPU/磁盘一览
free -h && df -h && uptime
top -bn1 | head -20

# 磁盘占用 Top10
du -sh /* 2>/dev/null | sort -rh | head -10
```

### 网络快速确认
```bash
# 连通性
curl -sv --max-time 5 http://localhost:<port>/health
wget -qO- --timeout=5 http://<host>:<port>

# 路由与 DNS
ip route show
dig +short <domain> @8.8.8.8
```

---

## 命令输出规范

给用户命令时，始终按此格式：

```bash
# [步骤编号] [操作说明] ⚠️/🔒/📋（如适用）
<command>

# 预期输出：[说明看到什么表示成功]
# 若失败：[下一步排查方向]
```

多步骤操作用分隔线区分阶段：
```
#### Phase 1: 确认现状
#### Phase 2: 执行变更  ⚠️
#### Phase 3: 验证结果
```

---

## Reference 文件索引

加载时机由上方路由表决定，按需读取，不要全部预载：

- `references/deployment.md` — Docker、systemd、Nginx、CI/CD、蓝绿部署、回滚
- `references/monitoring.md` — 性能分析、OOM、CPU/内存/磁盘/IO 诊断、告警
- `references/troubleshooting.md` — 故障排查树、日志分析、网络诊断、死锁
- `references/distributed.md` — K8s、服务网格、消息队列、分布式追踪
- `references/web3.md` — 以太坊/Solana/Cosmos 节点、validator、RPC、合约部署