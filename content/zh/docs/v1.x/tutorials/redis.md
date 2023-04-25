---
title: Redis 流量管理
description: 
weight: 1001
---

Redis 是一种高性能的键值数据库，通常被用作缓存、会话存储和消息代理等用途。Redis 可以部署为单机模式或者 Cluster 模式，但不同模式的客户端访问方式不同，这增加了应用使用 Redis 的开发成本。通过采用 Aeraki Mesh 的 Redis 流量管理功能，我们可以在不修改客户端代码的前提下切换后端的 Redis 部署模式，实现客户端无感知的 Redis Cluster 数据分片，并提供读写分离、流量镜像等高级流量管理功能。

比如您在测试集群使用一个小型的单实例的 Redis ，而在生产环境使用一个复杂而稳定的多副本实例的集群模式的Redis，通过 Aeraki Mesh，您将无需修改您应用的代码/配置，从而尽可能的保证了开发，预发布，线上环境相同（[Dev/prod parity](https://12factor.net/dev-prod-parity)）。

# 安装示例程序

从 github 下载 Aeraki。

```bash
git clone https://github.com/aeraki-mesh/aeraki.git
```

安装 Istio 和 Aeraki。

```bash
make demo
```

在 aerkai 代码目录下执行 redis demo 的安装脚本，该脚本会创建一个 redis namespace，并在其中部署 redis demo 应用。

```bash
./demo/redis/install.sh
```

demo 应用中包含一个 redis cluster（statefulset），一个单机模式的 redis，和一个 redis 客户端。后面我们将使用该 demo 来演示 Aeraki Mesh 对 Redis 的流量管理相关功能。
```bash
kubectl -n redis get pod
NAME                            READY   STATUS    RESTARTS   AGE
redis-client-c4db4746c-dzcdr    2/2     Running   0          4m26s
redis-cluster-0                 1/1     Running   0          4m26s
redis-cluster-1                 1/1     Running   0          4m24s
redis-cluster-2                 1/1     Running   0          4m22s
redis-cluster-3                 1/1     Running   0          4m21s
redis-cluster-4                 1/1     Running   0          4m20s
redis-cluster-5                 1/1     Running   0          4m19s
redis-single-5bf9d77d69-qh7f4   1/1     Running   0          95s
```

# 为 Redis 服务设置访问密码

demo 中部署的 redis-single 服务设置了访问密码，此时如果通过客户端去访问该服务，redis 服务器会提示需要用户认证，拒绝访问请求。

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-single
redis-single:6379> set a a
(error) NOAUTH Authentication required.
```

一般情况下，我们需要告知客户端其运行环境中访问的 Redis 服务是否需要进行认证，以及认证的用户名和密码。客户端代码根据配置来判断如何和 Redis 服务器进行认证。这增加了客户端代码及配置管理的复杂度。

通过采用 Aeraki Mesh 的 [RdisDestination](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisDestination) ，可以设置访问 Redis 的密码，由 Sidecar Proxy 来替客户端和 Redis 服务器进行认证。这样 Redis Client 就无需再关注 Redis 部署的密码信息。

```bash
kubectl apply -f- <<EOF
apiVersion: redis.aeraki.io/v1alpha1
kind: RedisDestination
metadata:
  name: redis-single
spec:
  host: redis-single.redis.svc.cluster.local
  trafficPolicy:
    connectionPool:
      redis:
        auth:
          plain:
            password: testredis
EOF
```



通过配置 Auth 字段，应用在访问 Redis 时，Aeraki 会自动为其完成鉴权。

RedisDestination 支持两种密钥获取的途径：secret 与 plain。

secret 代表所需要的用户名、密码信息存在于 secret 配置中。

默认情况下，RedisDestination 使用所引用 secret 中定义的 username 的值作为 auth 使用的用户名，password 或 token 作为 auth 时使用的密码。

通过配置 passwordField 和 usernameField 您可以指定做为auth时使用的密码和用户名的 secret 配置的字段。

# 屏蔽 Cluster 模式和普通模式的差异

Redis 要求客户端采用不同的 API 来访问普通模式和 [Cluster 模式](https://redis.io/topics/cluster-spec)。通过配置 RedisDestination，Aeraki Mesh 可以协助你的应用程序屏蔽掉 Cluster 模式 与普通模式的区别。

采用 Aeraki Mesh 后，当应用使用 Cluster 模式的 Redis 集群时，可以使用任何语言实现的非集群模式客户端链接到 Redis 集群，就像连接到单实例 Redis 一样。Aeraki Mesh 将自动同步 Redis 集群的拓扑信息，并将请求转发到集群中正确的节点上。


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
  name: app-cache
spec:
  host: app-cache.redis.svc.cluster.local
  trafficPolicy:
    connectionPool:
      redis:
        mode: CLUSTER
        auth:
          secret:
            name: redis-service-secret
EOF
```

使用 RedisDestination 另一个好处是：您可以让 Aeraki 帮助您自动的完成 Redis 的认证，从而避免通过代码或应用配置泄漏出资源的密钥。此时，您的服务就像是访问一个无鉴权的 Redis。

此外，您还可以配置访问 Redis 集群的链接池信息