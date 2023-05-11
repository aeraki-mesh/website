---
title: Fault injection
description: 
weight: 10
---

Using [RedisService](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisService), you can inject faults into Redis services, which can be useful for conducting chaos testing and other scenarios to ensure that the system properly handles Redis failures.

RedisService supports two types of fault configurations:

- Delay (DELAY): Adds delay to responses.
- Error (ERROR): Returns errors on requests.

For instance, the following configuration will result in 50% of GET commands directly returning errors.

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

Apply the above configuration, then connect to redis-cluster through the client, half of the GET commands will return the error message `Fault Injected: Error`.

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster

redis-cluster:6379> get a
(error) Fault Injected: Error
redis-cluster:6379> get a
"a"
```