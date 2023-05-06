---
title: 屏蔽 Redis 的部署模式差异
description: 
weight: 03
---


例如，当我们通过 Redis 客户端访问 demo 中部署的 Redis cluster，会出现下面的访问错误：

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster

redis-cluster:6379> set foo bar
(error) MOVED 15495 10.244.0.23:6379
```

这是因为 Redis 客户端采用了普通模式的 API，而 redis-cluster 服务则是一个 Redis Cluster，该 Cluster 由6台 Redis 服务器组成。通过配置 [RdisDestination](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisDestination)，Aeraki Mesh 可以屏蔽掉这种差异。

```yaml
kubectl apply -f- <<EOF
apiVersion: redis.aeraki.io/v1alpha1
kind: RedisDestination
metadata:
  name: redis-cluster
  namespace: redis
spec:
  host: redis-cluster.redis.svc.cluster.local
  trafficPolicy:
    connectionPool:
      redis:
        mode: CLUSTER
EOF
```

此时再通过客户端访问 redis-cluster 服务，就可以正常访问了。

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster

redis-cluster:6379> set foo bar
OK
```

