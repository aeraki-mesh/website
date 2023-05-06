---
title: 安装示例程序
description: 
weight: 01
---


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
