---
title: Making Redis deployment mode transparent to clients
description: 
weight: 03
---

Redis can be deployed in either single-node or cluster mode. Cluster mode offers several advantages over single-node mode, including:

* Supports horizontal scaling by increasing the number of nodes to improve performance and capacity.
* Supports automatic partitioning to distribute data across different nodes, enhancing load balancing and availability.
* Provides a certain degree of fault tolerance, enabling automatic failover and recovery even if a node fails or network issues arise.

Redis requires different client APIs for accessing single-node and cluster modes. However, by leveraging Aeraki Mesh’s Redis traffic management features, you can easily switch between these two modes without needing to modify client code, thus simplifying application development.

As an example, you might use a small, single-instance Redis service for testing purposes, while deploying a Redis cluster consisting of multiple instances for production workloads. Through the use of Aeraki Mesh, you can seamlessly connect our application to these different Redis services without needing to adjust application code or configurations, [ensuring that development, staging, and production environments remain consistent throughout the software deployment cycle](https://12factor.net/dev-prod-parity).

The Redis cluster deployed in the demo consists of six instances divided into three shards, each with one master node and one slave node. You can view the topology of this Redis cluster using the following command:

 ```bash
 kubectl exec -it redis-cluster-0 -c redis -n redis -- redis-cli cluster shards
 ``` 

![](../redis-cluster.png)

The Redis cluster’s topology is depicted in the above diagram, which shows that the cluster consists of three shards, with each shard responsible for a specific range of slots. Shard 0 handles slots 0 through 5460, Shard 1 handles slots 5461 through 10922, and Shard 2 handles slots 10923 through 16383. Importantly, each key is associated with a specific slot number, which is calculated using the formula CRC16(key) mod 16384.

Therefore, when writing or reading data to the Redis cluster, clients must first calculate the slot number based on the CRC16(key) mod 16384 algorithm, and then send the request to the corresponding node in the appropriate shard.

If you attempt to access the Redis cluster in the demo, you may encounter the following access error:

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster

redis-cluster:6379> set foo bar
(error) MOVED 15495 10.244.0.23:6379
```

The reason for this error is that the Redis client is using a regular API, which means that requests are sent to the redis-cluster service and then forwarded to a random Redis node in the cluster. If that node is not responsible for handling the corresponding slot of the key in the request, the node will return a MOVED error indicating which node the client should connect to instead.

Unfortunately, since the client is using a regular API, it is unable to interpret this error and automatically resend the request to the appropriate node. As a result, the client continues to return an error.

By setting the mode parameter in [RdisDestination](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisDestination) to CLUSTER, Aeraki Mesh is able to abstract away the differences between Redis’ Cluster mode and standalone mode. 

```yaml

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

Now, you can interact with the redis-cluster service just as you would with a standalone Redis node.

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster

redis-cluster:6379> set foo bar
OK
```

With this approach, you can seamlessly switch between development and production environments without modifying application code. Additionally, as our business grows and the demands on our Redis infrastructure become too burdensome, you can effortlessly migrate from a single-node Redis deployment to a Cluster.

In a Redis cluster, different keys are stored across different shards, allowing us to scale up by either increasing the number of replica nodes within each shard or adding additional shards to the cluster itself. This ensures that you can effectively manage any increases in data pressure that may result from ongoing business expansion. By integrating with Envoy proxy and leveraging its powerful traffic management functionality, Aeraki Mesh makes the entire migration and scaling process fully transparent, ensuring that online business operations remain uninterrupted throughout.
