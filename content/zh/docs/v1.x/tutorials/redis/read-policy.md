---
title: Redis 读写分离
description: 
weight: 05
---

在 Redis Cluster 中有多个分片（Shard），每个 Redis 分片中，通常有一个 Master 节点，一到多个 Slave（Replica）节点。Master 节点负责写操作，并将数据变化同步到 Slave 节点。Slave 节点作为备份节点，当 Master 不可用时，Slave 可以被选举成为新的 Master。由于 Slave 中保存了和 Master 中相同的数据，因此也可以响应客户端的读操作。

Aeraki Mesh 支持通过 [RedisService](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisService) 来为 Redis 设置不同的读策略：

* MASTER： 缺省的读模式。只从 Master 节点读取数据，当客户端要求数据强一致性时需要采用该模式。该模式对 Master 压力较大，在同一个分片内无法采用多个节点对读操作进行负载分担。
* PREFER_MASTER： 优先从 Master 节点读取数据，当 Master 节点不可用时，从 Replica 节点读取。
* REPLICA： 只从 Replica 节点读取数据，由于 Master 到 Replica 的数据复制过程是异步执行的，采用该方式有可能读取到过期的数据，因此适用于客户端对数据一致性要求不高的场景。该模式下可以采用多个 Replica 节点来分担来自客户端的读负载。
* PREFER_REPLICA： 优先从 Replica 节点读取数据，当 Replica 节点不可用时，从 Master 节点读取。
* ANY： 从任意节点读取数据。

如果客户端对缓存数据不要求强一致性，我们可以把读模式设置为 REPLICA。在该模式下，Master 节点只处理写操作，Slave 节点处理读操作，减少了 Master 节点的工作压力。随着业务的扩展，我们还可以在分片中增加更多的 Replica，以对读操作进行负载分担。

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
  settings:
    readPolicy: REPLICA  
  redis:
    - route:
        host: redis-cluster.redis.svc.cluster.local
EOF
```

