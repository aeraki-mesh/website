---
title: 屏蔽 Redis 的部署模式差异
description: 
weight: 03
---

Redis 可以部署为单机模式或者 [Cluster 模式](https://redis.io/docs/reference/cluster-spec/)。Cluster 模式相对于单机模式有以下优点：

* 支持水平扩展，可以通过增加节点数量可提高集群的性能和容量。
* 支持自动分区，可以将数据分散到不同的节点上，提高负载均衡和可用性。
* 具备一定的容错能力，即使某个节点宕机或网络出现问题，也可以自动进行故障转移和恢复。

Redis 要求采用不同的 client API 访问单机模式和 Cluster 模式。通过采用 Aeraki Mesh 的 Redis 流量管理功能，我们可以在不修改客户端代码的前提下切换后端的 Redis 部署模式，从而降低了应用开发成本。

比如在测试集群使用一个小型的单实例的 Redis ，而在生产环境使用一个复杂而稳定的多副本实例的集群模式的Redis，通过 Aeraki Mesh，我们无需修改应用的代码/配置，从而尽可能的保证了开发，预发布，线上环境相同（[Dev/prod parity](https://12factor.net/dev-prod-parity)）。


当我们通过 Redis 客户端访问 demo 中部署的 Redis cluster，会出现下面的访问错误：

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster

redis-cluster:6379> set foo bar
(error) MOVED 15495 10.244.0.23:6379
```

这是因为 Redis 客户端采用了普通模式的 API，而 redis-cluster 服务则是一个 Redis Cluster，该 Cluster 由 6 台 Redis 服务器组成。通过配置 [RdisDestination](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisDestination)，我们可以屏蔽掉这种差异。

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

采用该方法，我们可以在应用业务规模逐渐扩张，单一 Redis 节点压力过大时，将系统中的 Redis 从单节点无缝迁移到集群模式。在集群模式下，不同 key 的数据被缓存在不同的数据分片中，我们可以增加分片中 Replica 节点的数量来对一个分片进行扩容，也可以增加分片个数来对整个集群进行扩展，以应对由于业务不断扩展而增加的数据压力。Aeraki Mesh 通过配置 Envoy 代理了 Redis 的流量，整个迁移和扩容过程对客户端完全透明，不会影响到线上业务的正常运行。
