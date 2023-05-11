---
title: Authentication
description: 
weight: 02
---

## Redis server authentication

In the demo environment, the redis-single service has been configured with an access password. Attempting to access this service via a client will result in Redis prompting the user for authentication and denying the request.

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-single
redis-single:6379> set a a
(error) NOAUTH Authentication required.
```

Without Aeraki Mesh, the client needs to know whether authentication is required, as well as any necessary usernames and passwords for authentication. This adds complexity to client code.

By using Aeraki Mesh’s [RedisDestination](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisDestination), the password for accessing Redis can be set at the runtime, and authentication between the client and Redis server can be handled by the Sidecar Proxy. This eliminates the need for the Client to manage password information for the Redis service.

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

With this Aeraki Redis traffic rule, you should be able to access the redis-single service without any issues.

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-single

redis-single:6379> set foo bar
OK
```


## Client-side authentication
In some cases, you may want the access password used by the client to differ from the actual password for the Redis service. Doing so provides a number of advantages, including the ability to change the Redis service password without impacting the client and reducing the likelihood of exposing sensitive credentials. Through [RedisService](https://aeraki.net/zh/docs/v1.x/reference/redis/#RedisService), clients can be assigned separate access passwords to meet these needs.

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

To access the redis-single service at this time, the new password set within the above RedisService must be used.

```bash
kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-single

redis-single:6379> AUTH testredis
(error) ERR invalid password
redis-single:6379> AUTH testredis123!
OK
redis-single:6379> get foo
"bar"
```

> Note: Both the RedisDestination and RedisService “auth” fields support two different methods for obtaining keys: “secret” and “plain”. “Secret” indicates that any necessary username and password information can be found inside a Kubernetes secret resource. In such cases, the default username value for “auth” will be the “username” defined within the specified secret, with either the “password” or “token” serving as the associated password for “auth”. However, if alternative fields within the secret are used for authentication purposes, the configuration of “passwordField” and “usernameField” can be used to specify which password and username fields should be utilized for authentication.