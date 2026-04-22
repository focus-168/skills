# troubleshooting.md — 故障排查

## 目录
1. [故障排查通用流程](#general)
2. [Web 服务故障（502/503/504）](#http-errors)
3. [服务无响应 / 挂起](#unresponsive)
4. [日志分析技巧](#logs)
5. [网络故障诊断](#network)
6. [数据库故障](#database)
7. [死锁 & 僵尸进程](#deadlock)

---

## 1. 故障排查通用流程 {#general}

**黄金步骤（每次排查都从这里开始）**：

```bash
# Step 1: 确认问题范围和时间
date && uptime
dmesg | tail -20                         # 内核事件
journalctl -p err -b --no-pager | tail -30   # 本次启动所有错误

# Step 2: 服务状态
systemctl list-units --state=failed      # 所有失败的 unit
systemctl status <service> -l --no-pager

# Step 3: 资源快照
free -h && df -h && ss -s
ps aux --sort=-%cpu | head -10

# Step 4: 最近日志
journalctl -u <service> --since "15 min ago" --no-pager | grep -iE "error|warn|fatal|panic|exception"

# Step 5: 网络端口
ss -tlnp | grep <expected_port>
```

**判断树**：
```
服务挂了？
├── 进程存在（ps 能找到）？
│   ├── 端口在监听（ss 能找到）？
│   │   └── 能连上（curl health 成功）？ → 上层（Nginx/LB）问题
│   │   └── 不能连 → 防火墙 / iptables 问题
│   └── 端口不在监听 → 进程启动了但 bind 失败，看日志
└── 进程不存在
    ├── OOM Killed？ → dmesg 找 OOM
    ├── 崩溃退出？ → journalctl 找 signal
    └── 从未启动？ → 配置问题，看 systemd status
```

---

## 2. Web 服务故障（502 / 503 / 504）{#http-errors}

### 502 Bad Gateway
```bash
# Nginx 收到了后端的异常响应（或后端挂了）

# 1. 后端进程是否在跑
ps aux | grep <backend_process>
systemctl status <backend_service>

# 2. 后端端口是否在监听
ss -tlnp | grep <backend_port>

# 3. Nginx 错误日志
tail -50 /var/log/nginx/error.log | grep "connect()\|upstream"

# 4. 手动直连后端（绕过 Nginx）
curl -v http://127.0.0.1:<backend_port>/health

# 常见修复
systemctl restart <backend_service>     # 重启后端
# 若 upstream 连接数用尽：
grep worker_processes /etc/nginx/nginx.conf
grep worker_connections /etc/nginx/nginx.conf
```

### 503 Service Unavailable
```bash
# 通常是后端过载或 upstream 全部不健康

# 查看连接队列是否积压
ss -tn state syn-recv | wc -l      # TCP 半连接数
ss -tn state established | wc -l   # 已建立连接数

# Nginx upstream 状态（需开启 status 模块）
curl http://localhost/nginx_status

# 后端线程/进程是否耗尽
# Node.js: 检查 event loop lag
# Java: jstack <pid> | grep "BLOCKED" | wc -l
```

### 504 Gateway Timeout
```bash
# Nginx 等后端超时

# 1. 查看当前 timeout 配置
grep -r "proxy_read_timeout\|proxy_connect_timeout" /etc/nginx/

# 2. 后端是否真的在处理（慢请求）
# 查看正在执行的慢 SQL
mysql -e "SHOW PROCESSLIST;" | awk '$6 > 10'   # 执行超过 10 秒的查询

# 3. 临时延长 timeout（nginx）📋
# proxy_read_timeout 300s;
# proxy_send_timeout 300s;
nginx -t && systemctl reload nginx

# 4. 跟踪慢请求
curl -v --max-time 30 http://localhost:<port>/api/slow-endpoint
```

---

## 3. 服务无响应 / 挂起 {#unresponsive}

```bash
# 进程是否存在
pid=$(pgrep -f <process_name>)
echo "PID: $pid"

# 进程状态（D = 不可中断睡眠，通常是 IO 阻塞）
cat /proc/$pid/status | grep State
# R=running S=sleeping D=disk wait Z=zombie T=stopped

# D 状态（IO 卡住）
# 找是哪个 IO 操作阻塞了
cat /proc/$pid/wchan    # 正在等待的内核函数
ls -la /proc/$pid/fd    # 打开的文件描述符

# 查看线程堆栈
cat /proc/$pid/stack    # 内核栈

# 对 Java 应用：
jstack $pid | grep -A 10 "BLOCKED\|WAITING" | head -50

# 对 Go 应用（发送 SIGQUIT 打印 goroutine）
kill -QUIT $pid
journalctl -u <service> -n 50 --no-pager  # 查看输出

# 最后手段：强制重启 ⚠️
systemctl restart <service>
# 若 systemctl 也卡住：
kill -9 $pid && systemctl start <service>
```

---

## 4. 日志分析技巧 {#logs}

```bash
# 错误频率分析（找最多的错误类型）
grep "ERROR" /var/log/app/app.log | \
  sed 's/[0-9]\{4\}-[0-9]\{2\}-[0-9]\{2\}.*ERROR/ERROR/' | \
  sort | uniq -c | sort -rn | head -20

# 按分钟统计错误量（找错误爆发点）
grep "ERROR" /var/log/app/app.log | \
  awk '{print $1, substr($2,1,5)}' | \
  uniq -c | sort -rn | head -20

# 提取 Request ID 追踪单个请求全链路
grep "req-id-xxx" /var/log/app/app.log

# 多服务日志合并（按时间排序）
sort -m \
  <(awk '{print FILENAME, $0}' /var/log/service-a/app.log) \
  <(awk '{print FILENAME, $0}' /var/log/service-b/app.log) | \
  head -100

# 实时多文件同时监控
tail -f /var/log/nginx/error.log /var/log/app/app.log | \
  grep --line-buffered -E "ERROR|WARN|502|504"

# 统计某时间段请求量（Nginx）
awk '$4 >= "[01/Jan/2024:10:00" && $4 <= "[01/Jan/2024:11:00"' \
  /var/log/nginx/access.log | wc -l

# 日志文件被删除但进程还在写（inode 未释放）
lsof | grep deleted | grep <process_name>
# 修复：重启进程，或让进程重新打开日志文件（kill -HUP）
```

---

## 5. 网络故障诊断 {#network}

```bash
# 诊断层次：应用 → 端口 → 防火墙 → 路由 → DNS

# Layer 1: DNS 解析
dig +short <domain>
dig +short <domain> @8.8.8.8     # 用公网 DNS 对比
nslookup <domain>

# Layer 2: 连通性
ping -c 4 <ip>
traceroute -n <ip>               # 找丢包节点
mtr -n --report <ip>             # 更详细的路由分析

# Layer 3: 端口可达
nc -zv <ip> <port>               # TCP 端口测试
curl -v --connect-timeout 5 telnet://<ip>:<port>

# Layer 4: 防火墙
iptables -L INPUT -n -v --line-numbers   # 🔒 查看规则
iptables -L OUTPUT -n -v

# 临时放行端口（测试用）⚠️
iptables -I INPUT -p tcp --dport <port> -j ACCEPT
# 测试完记得删除：
iptables -D INPUT -p tcp --dport <port> -j ACCEPT

# ufw 操作
ufw status verbose
ufw allow <port>/tcp
ufw deny <port>/tcp

# Layer 5: 应用层抓包
tcpdump -i any -n port <port> -c 50 -A 2>/dev/null | head -100

# 查看连接超时/重置
ss -tn | grep -E "CLOSE_WAIT|TIME_WAIT|FIN_WAIT" | wc -l
# CLOSE_WAIT 过多 = 应用没有正确关闭连接（代码问题）
# TIME_WAIT 过多 = 高并发后正常现象，可调内核参数
```

**TIME_WAIT 优化（高并发场景）**：
```bash
# 🔒 开启 TCP 快速回收（谨慎，NAT 环境可能有问题）
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_fin_timeout=30
# 持久化：echo "net.ipv4.tcp_tw_reuse=1" >> /etc/sysctl.conf
```

---

## 6. 数据库故障 {#database}

### MySQL
```bash
# 连接与基础状态
mysql -e "SHOW STATUS LIKE 'Threads_connected';"
mysql -e "SHOW STATUS LIKE 'Questions';"
mysql -e "SHOW VARIABLES LIKE 'max_connections';"

# 慢查询
mysql -e "SHOW VARIABLES LIKE 'slow_query_log%';"
tail -50 /var/log/mysql/mysql-slow.log

# 锁等待
mysql -e "SELECT * FROM information_schema.INNODB_LOCK_WAITS\G"
mysql -e "SHOW ENGINE INNODB STATUS\G" | grep -A 30 "LATEST DETECTED DEADLOCK"

# 主从复制延迟
mysql -e "SHOW SLAVE STATUS\G" | grep -E "Seconds_Behind|Running"
```

### Redis
```bash
# 基础状态
redis-cli info server | grep redis_version
redis-cli info clients
redis-cli info memory | grep -E "used_memory_human|maxmemory_human"
redis-cli info stats | grep -E "total_commands|keyspace_hits|keyspace_misses"

# 命中率（应 > 90%）
redis-cli info stats | awk -F: '/keyspace_hits/{h=$2} /keyspace_misses/{m=$2} END{printf "Hit rate: %.1f%%\n", h/(h+m)*100}'

# 慢命令（阈值 1000 = 1ms）
redis-cli config set slowlog-log-slower-than 1000
redis-cli slowlog get 10

# 查找大 key（生产慎用，会阻塞）
redis-cli --bigkeys --sleep 0.1   # 加 sleep 减少影响
```

---

## 7. 死锁 & 僵尸进程 {#deadlock}

```bash
# 查找僵尸进程
ps aux | grep "Z"
ps -eo pid,ppid,stat,comm | awk '$3 ~ /Z/'

# 僵尸进程的父进程（通知父进程回收）
ps -eo pid,ppid,stat,comm | awk '$3 ~ /Z/{print "zombie pid:"$1, "parent:"$2}'
# 给父进程发 SIGCHLD
kill -SIGCHLD <parent_pid>
# 若父进程不响应，Kill 父进程（子进程由 init 回收）⚠️
kill -9 <parent_pid>

# 系统级死锁（D 状态进程积累）
ps aux | awk '$8 == "D"' | wc -l    # D 状态进程 > 10 = 严重 IO 问题

# strace 追踪进程系统调用（诊断卡在哪）🔒
strace -p <pid> -e trace=all -tt 2>&1 | head -50

# 文件锁
lsof | grep "POSIX\|FLOCK" | head -20
flock --nonblock /var/lock/<app>.lock echo "lock free" || echo "lock held"
```