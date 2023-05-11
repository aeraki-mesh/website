---
title: Install the demo
description: 
weight: 01
---
First, download Aeraki Mesh from GitHub.

```bash
git clone https://github.com/aeraki-mesh/aeraki.git
```


Before installing Aeraki, it is necessary to first install Istio since Aeraki is built on top of Istio. However, the Aeraki installation script takes care of Istio installation automatically, so executing the Aeraki installation script is all that is required.

```bash
make demo
```

Execute the following script. This script creates a namespace called redis and deploy the Redis demo application within it.

```bash
./demo/redis/install.sh
```

The demo application includes a 6-node Redis cluster (a Kubernetes StatefulSet), a single-mode Redis instance, and a Redis client. Later on, this demo will be used to demonstrate Aeraki Mesh’s traffic management capabilities for Redis.
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