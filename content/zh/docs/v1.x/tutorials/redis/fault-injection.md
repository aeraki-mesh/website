---
title: Redis 故障注入
description: 
weight: 05
---

采用 [RedisService](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisService) ，我们可以为 Redis 服务注入故障，该功能可以用于混沌测试等场景，以检验系统是否对于 Redis 故障进行了恰当的容错处理。

RedisService 支持配置两种类型的故障，分别是：

- 延迟(DELAY)：延迟是时间故障。它们模拟增加的网络延迟或一个由于超过负载而反应迟缓的 Redis 服务。
- 错误(ERROR)：错误是请求错误。它们模拟异常情况下的 Redis 服务。

例如，以下配置会使百分之五十的 GET 命令直接返回错误。

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
  faults:
    - type: ERROR
      percentage: 
        value: 50
      commands:
        - GET
EOF
```

此时通过客户端访问 redis-cluster 服务，会有一半的 GET 命令返回 ```Fault Injected: Error``` 错误信息。

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster

redis-cluster:6379> get a
(error) Fault Injected: Error
redis-cluster:6379> get a
"a"
```