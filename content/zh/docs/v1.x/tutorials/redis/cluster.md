---
title: 屏蔽 Redis 的部署模式差异
description: 
weight: 03
---

Redis 可以部署为单机模式或者 [Cluster 模式](https://redis.io/docs/reference/cluster-spec/)。Cluster 模式相对于单机模式有以下优点：

* 支持水平扩展，可以通过增加节点数量可提高集群的性能和容量。
* 支持自动分区，可以将数据分散到不同的节点上，提高负载均衡和可用性。
* 具备一定的容错能力，即使某个节点宕机或网络出现问题，也可以自动进行故障转移和恢复。

Redis 要求分别采用不同的 client API 访问单机模式和 Cluster 模式。通过采用 Aeraki Mesh 的 Redis 流量管理功能，我们可以在不修改客户端代码的前提下切换后端的 Redis 部署模式，从而降低了应用开发的复杂度。

比如我们可能在测试集群使用一个小型的单实例 Redis ，而在生产环境使用一个包含多个实例的 Redis Cluster。通过 Aeraki Mesh，我们无需修改应用的代码/配置就可以将应用对接到不同环境的 Redis 服务上，从而尽可能的保证了 [12-Factor 应用](https://12factor.net/zh_cn/)中的[开发，预发布，线上环境的一致性](https://12factor.net/zh_cn/dev-prod-parity)，实现应用的持续部署。

Demo 中部署的 Redis Cluster 由 6 台 Redis 服务器组成，Cluster 中有三个分片（Shard），每个分片中有一个 Master 节点，一个 Slave 节点。我们可以通过下面的命令来查看该 Redis Cluster 的拓扑结构：

 ```bash
 kubectl exec -it redis-cluster-0 -c redis -n redis -- redis-cli cluster shards
 ``` 

该 Redis Cluster 的拓扑结构如下图所示。从图中可以看到，该 Cluster 有三个分片（Shard）每个 Shard 负责一个范围的槽位（Slot），Shard 0 负责处理 Slot 0 到 5460，Shard 1 负责处理 Slot 5461 到 10922，Shard 2 负责处理 10923 到 16383。每个 Key 对应的 Slot 是固定的，其计算方式为 ```CRC16(key) mod 16384```。

当我们向 Redis Cluster 中写入或者读取数据时，要求客户端根据算法 ```CRC16(key) mod 16384``` 计算出数据所属的 Slot，然后将数据发送到对应 Shard 中的节点上。

![](../redis-cluster.png)


当我们尝试通过 Redis 客户端访问 demo 中部署的 Redis cluster，会出现下面的访问错误：

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster

redis-cluster:6379> set foo bar
(error) MOVED 15495 10.244.0.23:6379
```

这是因为 Redis 客户端采用了普通模式的 API。客户端将请求发送到 redis-cluster 服务，然后该请求被随机分发到该服务后端的一个 Redis 节点上。如果该节点不是请求的 key 根据算法算出的 Slot 所在的分片，那么该节点会返回一个 MOVED 错误，告诉客户端应该访问哪个节点。客户端会根据该错误，重新发送请求到正确的节点。但是，由于客户端采用的是普通模式的 API，因此客户端无法解析该错误，也就无法重新发送请求。因此，客户端会一直报错。

通过配置 [RdisDestination](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisDestination) 中的模式为 CLUSTER，Aeraki Mesh 可以为我们屏蔽 Cluster 模式和独立模式的差异。

```yaml
kubectl apply -f- <<EOF
apiVersion: redis.aeraki.io/v1alpha1
kind: RedisDestination
metadata:
  name: external-redis
  namespace: redis
spec:
  host: external-redis.redis.svc.cluster.local
  trafficPolicy:
    connectionPool:
      redis:
        mode: CLUSTER
EOF
```

此时我们就可以像访问一个独立 Redis 节点一样访问 redis-cluster 服务了。

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster

redis-cluster:6379> set foo bar
OK
```

采用该方法，我们可以在开发和生产环境中无缝切换，而无需修改应用代码。我们也可以在应用业务规模逐渐扩张，单一 Redis 节点压力过大时，将系统中的 Redis 从单节点无缝迁移到集群模式。

在集群模式下，不同 key 的数据被缓存在不同的数据分片中，我们可以增加分片中 Replica 节点的数量来对一个分片进行扩容，也可以增加分片个数来对整个集群进行扩展，以应对由于业务不断扩展而增加的数据压力。Aeraki Mesh 通过配置 Envoy 代理了 Redis 的流量，整个迁移和扩容过程对客户端完全透明，不会影响到线上业务的正常运行。