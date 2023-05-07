---
title: Redis 流量镜像
description: 
weight: 05
---

采用 [RedisService](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisService) ，我们可以将发送到 Redis 服务的请求同时发送一份到另一个 Redis 服务上。客户端只会收到来自主 Redis 服务的响应，而镜像 Redis 服务的响应会被丢弃。我们可以设置镜像流量的百分比，例如将 50% 的流量镜像到另一个 Redis 服务上。

例如下面的 RedisService 将所有发送到 redis-cluster 服务的请求同时发送到 redis-single 服务。

```yaml
kubectl apply -f- <<EOF
apiVersion: redis.aeraki.io/v1alpha1
kind: RedisService
metadata:
  name: redis-cluster
  namespace: redis
spec:
  host:
    - redis-cluster.redis.svc.cluster.local
  redis:
    - route:
        host: redis-cluster.redis.svc.cluster.local
      mirror:
        - route:
            host: redis-single.redis.svc.cluster.local
          percentage: 
            value: 100
EOF
```

此时通过客户端访问 redis-cluster 服务，设置 `test-traffic-mirroring` key 的值。

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster  

redis-cluster:6379> set test-traffic-mirroring "this key goes to both redis-cluster and redis-single"
OK
redis-cluster:6379> get test-traffic-mirroring
"this key goes to both redis-cluster and redis-single"
```

再通过客户端访问 redis-single 服务，可以看到 redis-single 服务中也有 `test-traffic-mirroring` 这个 key，说明客户端发送给 redis-cluster 的请求被同时发送到了 redis-single。

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-single

redis-single:6379> AUTH testredis123!
OK
redis-single:6379> get test-traffic-mirroring
"this key goes to both redis-cluster and redis-single"
```