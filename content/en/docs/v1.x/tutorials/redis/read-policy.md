---
title: Read/write separation
description: 
weight: 05
---

In Redis Cluster, there are multiple shards, each consisting of a master node and one or more replica(slave) nodes. The master nodes are responsible for handling write operations and synchronizing updates with their associated replicas. Replicas, in turn, serve as backup nodes and can respond to client read requests since they maintain an identical dataset to that of their master.

Aeraki Mesh supports setting different read policies for Redis through [RedisService](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisService):

* MASTER: the default read mode. It only reads data from the master node and should be used when strong consistency is required. This mode places significant pressure on the master node and cannot distribute read workload among replicas within the same shard.
* PREFER_MASTER: prioritizes reading data from the master node, but will fall back to replica nodes if it becomes unavailable.
* REPLICA: only read data from the replica nodes. Since the replication process between master and replica is asynchronous, this mode may return outdated data and is therefore suitable for scenarios where clients do not require strict data consistency. Multiple replicas can effectively distribute read load in this mode.
* PREFER_REPLICA: prioritize reading data from the replica nodes, but will revert to the master node if they become unavailable.
* ANY: read data from any available node.

By setting the read mode to REPLICA, you can reduce the workload on the master node by having it only handle write operations while the replica nodes handle read operations. As our business grows, you can also increase the number of replica nodes within a shard to distribute read operations across multiple nodes.

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