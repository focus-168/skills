# deployment.md — 部署、发布、回滚

## 目录
1. [Docker 容器部署](#docker)
2. [systemd 服务管理](#systemd)
3. [Nginx 配置与热重载](#nginx)
4. [蓝绿部署 / 滚动发布](#blue-green)
5. [CI/CD 自动化](#cicd)
6. [回滚操作](#rollback)

---

## 1. Docker 容器部署 {#docker}

### 标准发布流程
```bash
# Phase 1: 拉取新镜像
docker pull <registry>/<image>:<tag>

# 查看当前运行容器
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"

# Phase 2: 停旧起新（零停机需用 compose 或 K8s）⚠️
docker stop <container_name>
docker rm <container_name>
docker run -d \
  --name <container_name> \
  --restart unless-stopped \
  -p <host_port>:<container_port> \
  -v /data/<app>:/app/data \
  --env-file /etc/<app>/.env \
  <registry>/<image>:<tag>

# Phase 3: 验证
docker ps | grep <container_name>
docker logs <container_name> --tail 50 -f
curl -sf http://localhost:<port>/health && echo "OK"
```

### Docker Compose 滚动更新
```bash
# 📋 备份当前 compose 文件
cp docker-compose.yml docker-compose.yml.bak.$(date +%Y%m%d%H%M%S)

# 拉取 + 重建（仅重建变更服务）
docker compose pull
docker compose up -d --no-deps --build <service_name>

# 查看所有服务状态
docker compose ps
docker compose logs <service_name> --tail 100 -f
```

### 容器健康检查
```bash
# 检查容器资源占用
docker stats --no-stream

# 进入容器调试
docker exec -it <container_name> /bin/sh

# 查看容器网络
docker inspect <container_name> | grep -A 20 '"Networks"'

# 清理悬空镜像（定期执行）
docker image prune -f
docker system df
```

---

## 2. systemd 服务管理 {#systemd}

### 服务生命周期
```bash
# 查看服务状态（详细）
systemctl status <service> --no-pager -l

# 启动 / 停止 / 重启
systemctl start <service>
systemctl stop <service>
systemctl restart <service>      # 完全重启
systemctl reload <service>       # 热重载配置（不中断连接）

# 开机自启
systemctl enable <service>
systemctl disable <service>

# 查看服务日志（最近 100 行）
journalctl -u <service> -n 100 --no-pager

# 实时跟踪日志
journalctl -u <service> -f

# 按时间过滤
journalctl -u <service> --since "2024-01-01 10:00" --until "2024-01-01 11:00"
```

### 编写 systemd 服务单元
```bash
# 📋 创建服务文件
cat > /etc/systemd/system/<app>.service << 'EOF'
[Unit]
Description=<App> Service
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=<user>
WorkingDirectory=/opt/<app>
EnvironmentFile=/etc/<app>/.env
ExecStart=/usr/bin/<app> --config /etc/<app>/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# 重载 systemd 配置并启动
systemctl daemon-reload
systemctl enable --now <app>
systemctl status <app>
```

### PM2（Node.js 进程管理）
```bash
# 启动 / 重启
pm2 start ecosystem.config.js --env production
pm2 restart <app_name>
pm2 reload <app_name>    # 零停机热重载

# 查看状态
pm2 list
pm2 logs <app_name> --lines 100
pm2 monit

# 保存进程列表（开机自启）
pm2 save
pm2 startup
```

---

## 3. Nginx 配置与热重载 {#nginx}

### 配置管理
```bash
# 📋 修改前必备份
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak.$(date +%Y%m%d%H%M%S)
cp /etc/nginx/sites-available/<site> /etc/nginx/sites-available/<site>.bak.$(date +%Y%m%d%H%M%S)

# 验证配置语法
nginx -t

# 热重载（不中断现有连接）🔄
systemctl reload nginx

# 若 reload 失败，查看错误
nginx -T 2>&1 | grep error
journalctl -u nginx -n 20 --no-pager
```

### 常用 Nginx 配置片段

**反向代理**
```nginx
server {
    listen 80;
    server_name <domain>;

    location / {
        proxy_pass http://127.0.0.1:<port>;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 300s;
        proxy_connect_timeout 10s;
    }
}
```

**SSL + HTTP/2**
```nginx
server {
    listen 443 ssl http2;
    server_name <domain>;
    ssl_certificate     /etc/letsencrypt/live/<domain>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    add_header Strict-Transport-Security "max-age=63072000" always;
}
```

### 访问日志分析
```bash
# 请求量 Top10 IP
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# 状态码分布
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 最慢的请求（需开启 $request_time）
awk '{print $NF, $7}' /var/log/nginx/access.log | sort -rn | head -20

# 实时流量
tail -f /var/log/nginx/access.log | awk '{print $9}' | grep -v "200\|304"
```

---

## 4. 蓝绿部署 / 滚动发布 {#blue-green}

### Nginx 蓝绿切换
```bash
# 当前 Blue 在跑，部署 Green
docker run -d --name app-green -p 3001:3000 <image>:<new_tag>

# 验证 Green 健康
curl -sf http://localhost:3001/health && echo "Green OK"

# 切流量到 Green ⚠️
sed -i 's/proxy_pass http:\/\/127.0.0.1:3000/proxy_pass http:\/\/127.0.0.1:3001/' \
    /etc/nginx/sites-available/<app>
nginx -t && systemctl reload nginx

# 观察 30 秒，确认无报错
sleep 30 && curl -sf http://<domain>/health

# 停掉 Blue
docker stop app-blue && docker rm app-blue
```

### 脚本化蓝绿（可复用模板）
```bash
#!/bin/bash
# blue-green-deploy.sh
set -euo pipefail

IMAGE=$1
HEALTH_URL="http://localhost"
TIMEOUT=30

deploy_and_verify() {
    local color=$1 port=$2
    echo "==> Deploying $color on port $port"
    docker run -d --name "app-$color" -p "$port:3000" "$IMAGE"
    
    echo "==> Waiting for health check..."
    for i in $(seq 1 $TIMEOUT); do
        curl -sf "$HEALTH_URL:$port/health" && break
        sleep 1
    done || { echo "Health check failed"; docker rm -f "app-$color"; exit 1; }
    echo "==> $color healthy"
}
```

---

## 5. CI/CD 自动化 {#cicd}

### Ansible 远程部署
```bash
# 测试连通性
ansible -i inventory.ini all -m ping

# 执行 playbook（dry-run 先）
ansible-playbook -i inventory.ini deploy.yml --check
ansible-playbook -i inventory.ini deploy.yml -v

# 只对特定主机
ansible-playbook -i inventory.ini deploy.yml --limit "web01,web02"

# 传递变量
ansible-playbook -i inventory.ini deploy.yml -e "version=1.2.3 env=prod"
```

### GitHub Actions 触发手动部署
```bash
# 通过 SSH 触发远端脚本
ssh -o StrictHostKeyChecking=no deploy@<host> \
  "cd /opt/<app> && git pull && ./scripts/deploy.sh"

# 查看部署日志
ssh deploy@<host> "journalctl -u <app> -n 50 --no-pager"
```

---

## 6. 回滚操作 {#rollback}

```bash
# Docker 回滚到上一个镜像 🔄
docker stop <container>
docker rm <container>
docker run -d --name <container> <image>:<previous_tag>

# Docker Compose 回滚
docker compose down
# 编辑 docker-compose.yml 改回旧 tag
docker compose up -d

# Git 代码回滚（服务器端）⚠️
git log --oneline -10          # 确认目标 commit
git revert HEAD                # 安全回滚（新增一个撤销 commit）
# 或强制回滚（危险，会丢失提交）
git reset --hard <commit_hash>

# systemd 服务回滚（替换二进制）
systemctl stop <app>
cp /opt/<app>/bin/<app>.bak /opt/<app>/bin/<app>
systemctl start <app>
systemctl status <app>
```