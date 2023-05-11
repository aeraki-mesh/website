---
title: Prefix routing
description: 
weight: 04
---

You can route traffic to different Redis services based on the prefix of the requested key with the help of RedisService.

As an example, consider the following [RedisService](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisService), which routes any key beginning with “cluster” to the redis-cluster service, while all other keys are directed to the redis-single service.

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

With this configuration in place, first, you use a Redis client to access the redis-cluster service and set values for two distinct keys, “test-route” and “cluster-test-route”. You can then retrieve the values of these keys using the get command.

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

If you connect to the redis-single service, you can see that it only contains the “test-route” key, while the value of the “cluster-test-route” key is nil. This behavior confirms our RedisService configuration, as it shows that “test-route” has been correctly routed to the redis-single service, while “cluster-test-route” is being handled by the redis-cluster service.

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-single 

redis-single:6379> AUTH testredis123!
OK
redis-single:6379> get test-route
"this key goes to redis-single"
redis-single:6379> get cluster-test-route
(nil)
```