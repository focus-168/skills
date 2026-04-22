# monitoring.md — 性能监控 / OOM / 资源诊断

## 目录
1. [CPU 诊断](#cpu)
2. [内存 & OOM 诊断](#memory)
3. [磁盘 & IO 诊断](#disk)
4. [网络流量监控](#network)
5. [应用层性能](#app)
6. [告警与自动化](#alerting)

---

## 1. CPU 诊断 {#cpu}

```bash
# 实时 CPU 使用（按进程排序）
top -c -o %CPU
htop   # 推荐，需安装

# 查看 CPU 核数与型号
nproc
lscpu | grep -E "Model|Socket|Core|Thread"

# 找出 CPU 占用最高的进程（非交互）
ps aux --sort=-%cpu | head -15

# 查看 CPU 负载历史（1/5/15 分钟均值）
uptime
cat /proc/loadavg

# 判断是否 CPU 密集型（sy 高 = 系统调用多，us 高 = 用户代码）
vmstat 1 5
# 输出说明：r=运行队列 b=阻塞进程 us=用户 sy=内核 wa=IO等待 id=空闲

# 分析哪个线程占 CPU（找到 PID 后）
top -H -p <pid>

# 性能火焰图（需安装 perf）
perf top -p <pid>
perf record -g -p <pid> sleep 30
perf report
```

**CPU 高的判断树**：
- `wa` 高（>20%）→ IO 瓶颈，看磁盘
- `sy` 高（>30%）→ 系统调用/内核问题，看 `strace`
- `us` 高 → 业务代码，找具体进程/线程
- load avg > CPU 核数 × 2 → 过载，考虑限流或扩容

---

## 2. 内存 & OOM 诊断 {#memory}

```bash
# 内存总览
free -h
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|Cached|SwapUsed"

# 按内存排序进程
ps aux --sort=-%mem | head -15

# 查看 OOM Kill 记录 ⚠️
dmesg | grep -i "oom\|killed process" | tail -20
journalctl -k | grep -i "out of memory" | tail -20

# 查看某进程实际内存使用
cat /proc/<pid>/status | grep -E "VmRSS|VmSwap|VmPeak"
pmap -x <pid> | tail -5

# Swap 使用情况
swapon --show
vmstat -s | grep -i swap

# 内存泄漏粗判（持续增长则泄漏）
watch -n 5 'ps aux --sort=-%mem | head -5'
```

**OOM 处置流程**：
```bash
# 1. 确认 OOM 发生
dmesg | grep "Killed process" | tail -5
# 输出示例：Killed process 12345 (node) total-vm:2048000kB, anon-rss:1900000kB

# 2. 查看当前内存压力
free -h && cat /proc/meminfo | grep MemAvailable

# 3. 临时释放 PageCache（不影响业务）
sync && echo 1 > /proc/sys/vm/drop_caches

# 4. 调整 OOM 优先级（保护关键进程）
echo -1000 > /proc/<critical_pid>/oom_score_adj   # 🔒 -1000 = 不被 Kill
echo 500    > /proc/<less_critical_pid>/oom_score_adj

# 5. 限制进程内存（通过 systemd）
systemctl set-property <service> MemoryMax=2G
```

---

## 3. 磁盘 & IO 诊断 {#disk}

```bash
# 磁盘空间
df -hT | grep -v tmpfs
df -i            # inode 使用（inode 满了空间显示有余量但无法写入）

# 磁盘占用分析
du -sh /var/log/* | sort -rh | head -10
du -sh /home/*   | sort -rh | head -10

# 找大文件（>100M）
find / -xdev -size +100M -exec ls -lh {} \; 2>/dev/null | sort -k5 -rh | head -20

# IO 实时监控
iostat -xz 2 5   # 需安装 sysstat（重点看 %util await）
iotop -o         # 按进程看 IO（需 root）

# 查看 IO 瓶颈进程
pidstat -d 1 5   # 每秒刷新，显示各进程 IO

# 判断磁盘是否繁忙
iostat -x 1 | awk '/^sd|^vd|^nvme/{print $1, $NF}'
# %util 接近 100% = 磁盘繁忙
```

**磁盘满的快速处理**：
```bash
# 清理日志（安全）
journalctl --vacuum-size=500M       # 保留最新 500M 日志
journalctl --vacuum-time=7d         # 保留最近 7 天

# 清理 Docker（安全）
docker system prune -f              # 清理停止的容器/悬空镜像
docker volume prune -f              # 清理未使用的 volume ⚠️

# 清理旧内核（Ubuntu）🔒
apt autoremove --purge
dpkg -l 'linux-*' | grep ^ii

# 压缩大日志
gzip /var/log/app/app.log.1
find /var/log -name "*.log" -mtime +30 -exec gzip {} \;
```

---

## 4. 网络流量监控 {#network}

```bash
# 网卡实时流量
iftop -i eth0        # 需安装
nload eth0           # 简洁版

# 连接状态统计
ss -s                # 总览
ss -tnp              # 所有 TCP 连接（含进程）
ss -tnp state established | wc -l   # 当前建立连接数

# TIME_WAIT 过多（高并发后遗症）
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn

# 按进程查网络连接
ss -tnp | grep <process_name>
lsof -i -n -P | grep <process_name>

# 抓包分析（生产慎用）🔒
tcpdump -i eth0 -n port <port> -c 100 -w /tmp/capture.pcap
tcpdump -i eth0 -n 'host <ip> and port 80' -A | head -50

# 带宽测试（内网）
iperf3 -s                    # 服务端
iperf3 -c <server_ip> -t 10  # 客户端测 10 秒
```

**连接排查**：
```bash
# 服务是否在监听
ss -tlnp | grep <port>
# 或
netstat -tlnp | grep <port>

# 防火墙是否拦截
iptables -L -n -v | grep <port>
ufw status verbose

# 远端可达性
traceroute <ip>
mtr --report <ip>   # 更详细
```

---

## 5. 应用层性能 {#app}

```bash
# HTTP 响应时间测试
curl -o /dev/null -s -w \
  "DNS:%{time_namelookup}s Connect:%{time_connect}s TTFB:%{time_starttransfer}s Total:%{time_total}s\n" \
  http://<url>

# 压测（需安装 ab 或 wrk）
ab -n 1000 -c 50 http://localhost:<port>/api/health
wrk -t4 -c100 -d30s http://localhost:<port>/api/health

# 查看应用 FD（文件描述符）限制
cat /proc/<pid>/limits | grep "open files"
ls /proc/<pid>/fd | wc -l     # 当前 FD 数量

# 调整系统 FD 限制（高并发服务必备）🔒
ulimit -n 65536                # 临时
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
sysctl -w fs.file-max=2097152
```

### 数据库连接监控
```bash
# MySQL 连接数
mysql -e "SHOW STATUS LIKE 'Threads_connected';"
mysql -e "SHOW PROCESSLIST;" | grep -v Sleep | head -20

# Redis 连接与内存
redis-cli info clients | grep connected_clients
redis-cli info memory | grep used_memory_human
redis-cli --latency -h 127.0.0.1

# PostgreSQL 连接数
psql -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"
```

---

## 6. 告警与自动化 {#alerting}

### 简易磁盘告警脚本
```bash
#!/bin/bash
# disk-alert.sh — 放入 crontab，磁盘 > 85% 发告警
THRESHOLD=85
WEBHOOK="https://hooks.slack.com/services/xxx"

df -h | grep -vE '^Filesystem|tmpfs|udev' | awk '{print $5 " " $6}' | while read output; do
    usage=$(echo "$output" | awk '{print $1}' | tr -d '%')
    mount=$(echo "$output" | awk '{print $2}')
    if [ "$usage" -ge "$THRESHOLD" ]; then
        curl -s -X POST "$WEBHOOK" \
          -H 'Content-type: application/json' \
          -d "{\"text\":\"🚨 Disk Alert: $mount at ${usage}% on $(hostname)\"}"
    fi
done

# crontab 注册（每 5 分钟检查）
# */5 * * * * /opt/scripts/disk-alert.sh
```

### 进程守卫脚本
```bash
#!/bin/bash
# process-guard.sh — 服务挂了自动重启
SERVICE="<app>"
if ! systemctl is-active --quiet "$SERVICE"; then
    echo "$(date): $SERVICE down, restarting..." >> /var/log/guard.log
    systemctl restart "$SERVICE"
    # 通知
    curl -s -X POST "$WEBHOOK" -d "{\"text\":\"⚠️ $SERVICE restarted on $(hostname)\"}"
fi
# crontab: * * * * * /opt/scripts/process-guard.sh
```

### 系统健康快照（每日归档）
```bash
#!/bin/bash
# daily-snapshot.sh
OUTPUT="/var/log/snapshots/$(date +%Y%m%d).txt"
mkdir -p /var/log/snapshots

{
  echo "=== $(date) ==="
  echo "--- Uptime ---"    && uptime
  echo "--- Memory ---"    && free -h
  echo "--- Disk ---"      && df -h
  echo "--- Top CPU ---"   && ps aux --sort=-%cpu | head -8
  echo "--- Connections ---" && ss -s
  echo "--- Failed Units ---" && systemctl --failed --no-legend
} > "$OUTPUT"

# 保留 30 天
find /var/log/snapshots -mtime +30 -delete
```