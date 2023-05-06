---
title: 设置 Redis 访问密码
description: 
weight: 02
---

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

在某些情况下，您可能希望客户端使用的访问密码与 Redis 服务的真实密码不同。这样会带来一些好处，比如可以在不影响客户端的情况下更换 Redis 服务密码。这样也减少了 Redis 服务密码暴露的风险。通过配置 [RedisService](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisService) ，您可以单独设置客户端的访问密码。

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

> 备注：RedisDestination 和 RedisService 中的 auth 字段均支持两种密钥获取的途径：secret 与 plain。secret 代表所需要的用户名、密码信息存在于 secret 配置中。默认情况下，将使用所引用 secret 中定义的 username 的值作为 auth 使用的用户名，password 或 token 作为 auth 时使用的密码。通过配置 passwordField 和 usernameField 您可以指定做为auth时使用的密码和用户名的 secret 配置的字段。
