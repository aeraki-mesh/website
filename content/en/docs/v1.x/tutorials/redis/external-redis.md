---
title: Connect to external redis
description: 
weight: 11
---

While the Redis service in the demo is deployed within a Kubernetes cluster, it’s possible to use Aeraki Mesh to connect to a Redis service that’s outside of the cluster. This can be done by creating a [service without selectors](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors), followed by creating an EndpointSlice for the service to specify the address of the external Redis. Once that’s done, RedisService and Redis Destination can be used to manage traffic for the service, just as they would for Redis within the cluster.

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
      - 10.244.0.26 # The address of the external Redis, for example, one provided by a cloud provider
EOF
```