---
title: 连接集群外 Redis 服务
description: 
weight: 11
---

如果需要连接到一个集群外的 Redis 服务，我们可以采用一个[无选择器服务](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors)来定义该服务，然后创建一个 EndpointSlice 来指定该服务的外部地址。然后就可以像集群内服务一样使用 RedisService 和 Redis Destination 来对该服务进行流量管理了。

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