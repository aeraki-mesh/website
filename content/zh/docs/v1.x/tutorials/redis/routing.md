---
title: Redis 流量路由
description: 
weight: 04
---

采用 [RedisService](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisService) ，我们可以根据请求 key 的前缀将流量路由到不同的 Redis 服务上。

例如下面的 RedisService 将以 `cluster` 开头的 key 路由到 redis-cluster 服务，其它 key 路由到 redis-single 服务。

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
    - match:
        key:
          prefix: cluster
      route:
        host: redis-cluster.redis.svc.cluster.local
    - route:
        host: redis-single.redis.svc.cluster.local
EOF
```

此时通过客户端访问 redis-cluster 服务，分别设置 `test-route` 和 `cluster-test-route` 两个 key 的值。然后这两个 key 的值也可以通过 get 命令获取到。

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster

redis-cluster:6379> set test-route "this key goes to redis-single"
OK
redis-cluster:6379> set cluster-test-route "this key goes to redis-cluster"
OK
redis-cluster:6379> get test-route
"this key goes to redis-single"
redis-cluster:6379> get cluster-test-route
"this key goes to redis-cluster
```

再通过客户端访问 redis-single 服务，可以看到 redis-single 服务中只有 `test-route` 这个 key，而 `cluster-test-route` 这个 key 的值为 nil。这说明 `test-route` 被路由到 redis-single 服务，而 `cluster-test-route` 被路由到 redis-cluster 服务。

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-single 

redis-single:6379> AUTH testredis123!
OK
redis-single:6379> get test-route
"this key goes to redis-single"
redis-single:6379> get cluster-test-route
(nil)
```