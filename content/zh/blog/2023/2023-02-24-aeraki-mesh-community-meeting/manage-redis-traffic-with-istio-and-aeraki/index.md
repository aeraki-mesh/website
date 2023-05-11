---
layout:     post
title:      "Database Mesh: 使用 Istio 和 Aeraki 对 Redis 进行流量管理"
subtitle:   ""
description: "Aeraki Mesh 提供了对 Redis 的流量管理能力，可以实现客户端无感知的 Redis Cluster 数据分片，按 key 将客户端请求路由到不同的 Redis 服务，读写分离，流量镜像，故障注入等高级流量管理功能。"
author: "赵化冰"
date: 2023-05-09
keywords: [istio,aeraki,redis]
---

![](https://images.unsplash.com/photo-1473186578172-c141e6798cf4?ixlib=rb-4.0.3&amp;ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&amp;auto=format&amp;fit=crop&amp;w=1973&amp;q=80)

Redis 是一种高性能的键值数据库，支持丰富的数据结构和操作，包括字符串、哈希、列表、集合、有序集合等。由于其强大的能力，Redis 被广泛应用于缓存、会话存储、消息代理等场景中。

Aeraki Mesh 提供了对 Redis 的完善的流量管理能力，可以实现客户端无感知的 Redis Cluster 数据分片，支持按 key 将客户端请求路由到不同的 Redis 服务，还提供了读写分离，流量镜像，故障注入等能力。本文将使用 Aeraki Mesh 自带的 Demo 来演示如何使用 Aeraki Mesh 来对 Redis 进行流量管理。

# 系统架构

Aeraki 和 Istio 工作在控制面，数据面则由 Envoy sidecar 组成。Aeraki 在控制面提供了 [RedisService](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisDestination) 和 [RedisDestination](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisDestination) 两个用户友好的 Kubernetes CRD 来作为面向运维的操作接口，并将用户配置的流量规则转换为数据面配置，通过 xDS 下发到 Envoy Sidecar。Envoy Sidecar 拦截客户端的 Redis 访问请求，并根据控制面下发的配置提供相关的流量管理能力。
![](/img/2023-05-08-manage-redis-with-aeraki-mesh/aeraki-redis.png)

# 安装示例程序

首先从 github 下载 Aeraki Mesh 的源码。

```bash
git clone https://github.com/aeraki-mesh/aeraki.git
```

安装 Istio 和 Aeraki。Aeraki 是基于 Istio 的，所以需要先安装 Istio。Aeraki 的安装脚本会自动安装 Istio，所以只需要执行 Aeraki 的安装脚本即可。

```bash
make demo
```

在 Aerkai 代码目录下执行 redis demo 的安装脚本，该脚本会创建一个 redis namespace，并在其中部署 redis demo 应用。

```bash
./demo/redis/install.sh
```

demo 应用中包含一个 6 节点的 redis cluster（statefulset），一个单机模式的 redis，和一个 redis 客户端。后面我们将使用该 demo 来演示 Aeraki Mesh 对 Redis 的流量管理相关功能。
```bash
kubectl -n redis get pod
NAME                            READY   STATUS    RESTARTS   AGE
redis-client-644c965f48-dvjc7   2/2     Running   0          2m17s
redis-cluster-0                 1/1     Running   0          2m17s
redis-cluster-1                 1/1     Running   0          2m15s
redis-cluster-2                 1/1     Running   0          2m13s
redis-cluster-3                 1/1     Running   0          2m12s
redis-cluster-4                 1/1     Running   0          2m11s
redis-cluster-5                 1/1     Running   0          2m10s
redis-single-ccbb984dc-qz22v    1/1     Running   0          2m17s
```

# 设置 Redis 访问密码

## 为 Redis 服务设置访问密码

demo 中部署的 redis-single 服务设置了访问密码，此时如果通过客户端去访问该服务，redis 服务器会提示需要用户认证，拒绝访问请求。

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-single
redis-single:6379> set a a
(error) NOAUTH Authentication required.
```

在未使用 Aeraki Mesh 对 Redis 进行流量管理时，客户端代码需要从配置中得知其运行环境中访问的 Redis 服务是否需要进行认证，以及认证的用户名和密码。然后根据配置和 Redis 服务器进行认证，这增加了客户端代码及配置管理的复杂度。

通过采用 Aeraki Mesh 的 [RdisDestination](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisDestination) ，可以设置访问 Redis 的密码，由 Sidecar Proxy 来替客户端和 Redis 服务器进行认证。这样 Redis Client 就无需再关注 Redis 服务的密码信息。

```yaml
kubectl apply -f- <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: redis-service-secret
  namespace: redis
type: Opaque
data:
  password: dGVzdHJlZGlzCg==
---
apiVersion: redis.aeraki.io/v1alpha1
kind: RedisDestination
metadata:
  name: redis-single
  namespace: redis
spec:
  host: redis-single.redis.svc.cluster.local
  trafficPolicy:
    connectionPool:
      redis:
        auth:
          secret:
            name: redis-service-secret
EOF
```

此时再通过客户端访问 redis-single 服务，就可以正常访问了。

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-single

redis-single:6379> set foo bar
OK
```

## 为客户端设置独立的访问密码

在某些情况下，我们可能希望客户端使用的访问密码与 Redis 服务的真实密码不同。这样会带来一些好处，比如可以在不影响客户端的情况下更换 Redis 服务密码。这样也减少了 Redis 服务密码暴露的风险。通过配置 [RedisService](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisService) ，我们可以单独设置客户端的访问密码。

```yaml
kubectl apply -f- <<EOF
apiVersion: redis.aeraki.io/v1alpha1
kind: RedisService
metadata:
  name: redis-single
  namespace: redis
spec:
  host:
    - redis-single.redis.svc.cluster.local
  settings:
    auth:
      plain:
        password: testredis123!
EOF
```

此时访问 redis-single 服务，需要使用 RedisService 中设置的新密码。

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-single

redis-single:6379> AUTH testredis
(error) ERR invalid password
redis-single:6379> AUTH testredis123!
OK
redis-single:6379> get foo
"bar"
```

> 备注：RedisDestination 和 RedisService 中的 auth 字段均支持两种密钥获取的途径：secret 与 plain。secret 代表所需要的用户名、密码信息存在于 Kubernetes secret 资源中。在这种情况下，默认将使用所引用 secret 中定义的 username 的值作为 auth 使用的用户名，password 或 token 作为 auth 时使用的密码。如果使用了 secret 中不同的字段来进行认证，我们通过配置 passwordField 和 usernameField 来指定认证使用的密码和用户名字段。

# 屏蔽 Redis 的部署模式差异

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

![](/img/2023-05-08-manage-redis-with-aeraki-mesh/redis-cluster.png)


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

# Redis 流量路由

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

# Redis 读写分离

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

# 流量镜像

采用 [RedisService](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisService) ，我们可以将发送到 Redis 服务的请求同时发送一份到另一个 Redis 服务上。客户端只会收到来自主 Redis 服务的响应，而镜像 Redis 服务的响应会被丢弃。我们可以设置镜像流量的百分比，例如将 50% 的流量镜像到另一个 Redis 服务上。

例如下面的 RedisService 将所有发送到 redis-cluster 服务的请求同时发送到 redis-single 服务。

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

此时通过客户端访问 redis-cluster 服务，设置 `test-traffic-mirroring` key 的值。

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster  

redis-cluster:6379> set test-traffic-mirroring "this key goes to both redis-cluster and redis-single"
OK
redis-cluster:6379> get test-traffic-mirroring
"this key goes to both redis-cluster and redis-single"
```

再通过客户端访问 redis-single 服务，可以看到 redis-single 服务中也有 `test-traffic-mirroring` 这个 key，说明客户端发送给 redis-cluster 的请求被同时发送到了 redis-single。

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-single

redis-single:6379> AUTH testredis123!
OK
redis-single:6379> get test-traffic-mirroring
"this key goes to both redis-cluster and redis-single"
```

# 故障注入

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

# 连接集群外 Redis 服务

Demo 中的 Redis 是部署在 Kubernetes 集群中的，但其实我们也可以通过 Aeraki Mesh 连接集群外的 Redis 服务。我们可以在 Kubernetes 集群中创建一个 [无选择器服务](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors)，然后创建一个该服务对应的 EndpointSlice 来指定外部 Redis 的地址。然后就可以像集群内服务一样使用 RedisService 和 Redis Destination 来对该服务进行流量管理了。

```yaml
kubectl apply -f- <<EOF 
apiVersion: v1
kind: Service
metadata:
  name: external-redis
  namespace: redis
spec:
  ports:
    - name: tcp-redis
      protocol: TCP
      port: 6379
      targetPort: 6379
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-redis
  namespace: redis
  labels:
    kubernetes.io/service-name: external-redis
addressType: IPv4
ports:
  - name: tcp-redis
    port: 6379
    protocol: TCP
endpoints:
  - addresses:
      - 10.244.0.26 # 集群外 Redis 实例的地址，比如云厂商提供的 Redis 服务。
EOF
```

# 小结
本文介绍了如何使用 Aeraki Mesh 来对 Redis 进行流量治理，包括流量路由、流量镜像、故障注入等功能。目前该功能已经在 Aeraki 的最新版本中提供，欢迎大家试用并提出宝贵的意见和建议。更多 Redis 流量治理的介绍请参考 Aeraki Mesh 官网教程中的 [Redis 流量管理](https://www.aeraki.net/zh/docs/v1.x/tutorials/redis/)。

# 参考资料
- [Redis 流量管理](https://www.aeraki.net/zh/docs/v1.x/tutorials/redis/)
- [RedisService](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisService)
- [Redis Destination](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisDestination)