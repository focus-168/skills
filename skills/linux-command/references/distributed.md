# distributed.md — 分布式服务 / K8s / 消息队列

## 目录
1. [Kubernetes 日常运维](#kubernetes)
2. [服务发现与负载均衡](#service-discovery)
3. [消息队列](#mq)
4. [分布式追踪](#tracing)
5. [集群网络诊断](#cluster-network)

---

## 1. Kubernetes 日常运维 {#kubernetes}

### 快速状态检查
```bash
# 集群节点状态
kubectl get nodes -o wide
kubectl describe node <node_name> | grep -A 20 "Conditions:"

# 所有命名空间概览
kubectl get all -A | grep -v Running | grep -v Completed

# Pod 状态（找异常）
kubectl get pods -A --field-selector=status.phase!=Running | grep -v Completed
kubectl get pods -n <namespace> -o wide

# 异常 Pod 排查
kubectl describe pod <pod_name> -n <namespace> | grep -A 10 "Events:"
kubectl logs <pod_name> -n <namespace> --tail=100
kubectl logs <pod_name> -n <namespace> --previous   # 上一次崩溃的日志
```

### Pod 生命周期管理
```bash
# 重启 Deployment（触发滚动更新）
kubectl rollout restart deployment/<name> -n <namespace>

# 查看滚动更新状态
kubectl rollout status deployment/<name> -n <namespace>

# 回滚 ⚠️
kubectl rollout undo deployment/<name> -n <namespace>
kubectl rollout undo deployment/<name> -n <namespace> --to-revision=2

# 查看历史版本
kubectl rollout history deployment/<name> -n <namespace>

# 强制删除卡住的 Pod ⚠️
kubectl delete pod <pod_name> -n <namespace> --grace-period=0 --force

# 扩缩容
kubectl scale deployment/<name> --replicas=3 -n <namespace>
kubectl autoscale deployment/<name> --cpu-percent=70 --min=2 --max=10
```

### 资源诊断
```bash
# 节点资源请求量
kubectl describe nodes | grep -A 5 "Resource\|Allocated"

# Pod 实际资源使用（需 metrics-server）
kubectl top pods -n <namespace> --sort-by=memory
kubectl top nodes

# 查看 Pod 资源限制
kubectl get pods -n <namespace> -o jsonpath=\
'{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].resources}{"\n"}{end}'

# 事件按时间倒序（找最近异常）
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20
kubectl get events -n <namespace> --field-selector type=Warning
```

### 进入 Pod 调试
```bash
# 进入容器 Shell
kubectl exec -it <pod_name> -n <namespace> -- /bin/sh
kubectl exec -it <pod_name> -n <namespace> -c <container_name> -- bash

# 临时调试容器（不修改原 Pod）
kubectl debug -it <pod_name> -n <namespace> \
  --image=nicolaka/netshoot --target=<container_name>

# 端口转发（本地调试）
kubectl port-forward svc/<service_name> 8080:80 -n <namespace>
kubectl port-forward pod/<pod_name> 9090:9090 -n <namespace>

# 复制文件
kubectl cp <namespace>/<pod_name>:/app/logs/app.log ./app.log
```

### ConfigMap & Secret 管理
```bash
# 查看
kubectl get configmap <name> -n <namespace> -o yaml
kubectl get secret <name> -n <namespace> -o jsonpath='{.data.<key>}' | base64 -d

# 更新 ConfigMap 触发滚动更新
kubectl create configmap <name> --from-file=config.yaml \
  --dry-run=client -o yaml | kubectl apply -f -

# 强制触发更新（加 annotation）
kubectl patch deployment <name> -n <namespace> \
  -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"config-hash\":\"$(date +%s)\"}}}}}"
```

---

## 2. 服务发现与负载均衡 {#service-discovery}

### Kubernetes Service 诊断
```bash
# Service 端点是否正常
kubectl get endpoints <service_name> -n <namespace>
# 若 Endpoints 为空 = Pod 的 label 与 selector 不匹配

# 检查 selector 匹配
kubectl get svc <service_name> -n <namespace> -o jsonpath='{.spec.selector}'
kubectl get pods -n <namespace> -l <key>=<value>

# Service 连通性（从集群内测试）
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl -v http://<service_name>.<namespace>.svc.cluster.local:<port>/health

# Ingress 状态
kubectl get ingress -A
kubectl describe ingress <name> -n <namespace>
```

### HAProxy / Keepalived（传统 LB）
```bash
# HAProxy 状态
systemctl status haproxy
haproxy -c -f /etc/haproxy/haproxy.cfg   # 验证配置

# 查看后端健康状态（需开启 stats）
curl -u admin:password http://localhost:8404/stats | grep -E "FRONTEND|BACKEND|DOWN"

# 实时日志
tail -f /var/log/haproxy.log | grep -E "error|DOWN|UP"

# 热重载配置 🔄
haproxy -c -f /etc/haproxy/haproxy.cfg && systemctl reload haproxy
```

---

## 3. 消息队列 {#mq}

### Kafka
```bash
# 查看 topic 列表
kafka-topics.sh --bootstrap-server localhost:9092 --list

# 查看 topic 详情（分区/副本）
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --topic <topic_name>

# 消费者组 Lag（关键监控指标）
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group <group_id>
# LAG 列持续增长 = 消费跟不上生产

# 查看所有消费者组
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# 消费消息（调试用）
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic <topic> --from-beginning --max-messages 10

# 生产测试消息
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic <topic>

# 查看 broker 状态
kafka-broker-api-versions.sh --bootstrap-server localhost:9092
```

### RabbitMQ
```bash
# 状态概览
rabbitmqctl status
rabbitmqctl cluster_status

# 队列列表（含消息数）
rabbitmqctl list_queues name messages consumers

# 找积压队列（messages > 0 且 consumers = 0）
rabbitmqctl list_queues name messages consumers | awk '$2 > 0 && $3 == 0'

# 管理界面（需插件）
rabbitmq-plugins enable rabbitmq_management
# 访问 http://localhost:15672  默认 guest/guest

# 清空队列 ⚠️
rabbitmqctl purge_queue <queue_name>
```

---

## 4. 分布式追踪 {#tracing}

### 日志关联（手动追踪）
```bash
# 跨服务追踪同一个 Request ID
TRACE_ID="abc-123-def"

grep "$TRACE_ID" /var/log/service-a/app.log
grep "$TRACE_ID" /var/log/service-b/app.log
grep "$TRACE_ID" /var/log/service-c/app.log

# 合并多服务日志并按时间排序
for svc in service-a service-b service-c; do
    grep "$TRACE_ID" /var/log/$svc/app.log | \
      awk -v s=$svc '{print $1, $2, s, $0}'
done | sort -k1,2 | awk '{$1=$2=$3=""; print $0}'
```

### Jaeger / Zipkin CLI 查询
```bash
# 查询慢 Trace（> 1s）
curl "http://jaeger:16686/api/traces?service=<service>&minDuration=1000000" | \
  python3 -m json.tool | grep -E "traceID|duration"
```

### 服务依赖健康检查脚本
```bash
#!/bin/bash
# check-dependencies.sh — 启动前检查所有依赖
SERVICES=(
    "postgres:5432"
    "redis:6379"
    "kafka:9092"
    "service-b:8080"
)

all_ok=true
for svc in "${SERVICES[@]}"; do
    host="${svc%:*}"
    port="${svc#*:}"
    if nc -z -w 3 "$host" "$port" 2>/dev/null; then
        echo "✅ $svc reachable"
    else
        echo "❌ $svc unreachable"
        all_ok=false
    fi
done

$all_ok || { echo "Dependency check failed"; exit 1; }
```

---

## 5. 集群网络诊断 {#cluster-network}

```bash
# K8s 网络策略（NetworkPolicy）
kubectl get networkpolicy -A
kubectl describe networkpolicy <name> -n <namespace>

# DNS 解析（集群内）
kubectl run -it --rm dnstest --image=busybox --restart=Never -- \
  nslookup kubernetes.default

# 验证 Pod 间通信
kubectl exec -it <pod-a> -n <namespace> -- \
  curl -v http://<pod-b-ip>:<port>/health

# 查看 CNI 插件状态（以 Calico 为例）
kubectl get pods -n kube-system | grep calico
kubectl exec -it <calico-pod> -n kube-system -- \
  calicoctl node status

# iptables 规则数量（K8s 节点）
iptables -L | wc -l         # 规则太多会影响性能（> 10000 需优化）
iptables -t nat -L KUBE-SERVICES | wc -l

# 分布式环境时钟同步（NTP）
timedatectl status
chronyc tracking              # 时钟偏差 > 500ms 可能导致分布式锁、JWT 失效
chronyc sources -v
systemctl restart chronyd    # 强制同步
```

### etcd 健康（K8s 控制面）
```bash
# etcd 集群健康
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/peer.crt \
  --key=/etc/kubernetes/pki/etcd/peer.key \
  endpoint health

# etcd 性能（p99 写延迟 > 25ms 需关注）
ETCDCTL_API=3 etcdctl endpoint status --write-out=table
```