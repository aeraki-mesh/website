---
title: Traffic mirroring
description: 
weight: 05
---

Using [RedisService](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisService), you have the ability to duplicate any requests sent to a Redis service and route them to another Redis service concurrently. The client will only receive the response from the primary Redis service, while the response from the mirrored Redis service is discarded. You can also set the percentage of traffic that is mirrored, for example, mirroring 50% of the traffic to another Redis service.

For example, consider the following RedisService configuration, which mirrors any requests sent to the redis-cluster service to the redis-single service:

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

Apply the above configuration, then let’s connect to redis-cluster and set the value of `test-traffic-mirroring`.

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster  

redis-cluster:6379> set test-traffic-mirroring "this key goes to both redis-cluster and redis-single"
OK
redis-cluster:6379> get test-traffic-mirroring
"this key goes to both redis-cluster and redis-single"
```

Now if you connect to redis-single, you can see that the key “test-traffic-mirroring” also exists in the redis-single service. This implies that the requests sent to the redis-cluster were mirrored to the redis-single.

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-single

redis-single:6379> AUTH testredis123!
OK
redis-single:6379> get test-traffic-mirroring
"this key goes to both redis-cluster and redis-single"
```