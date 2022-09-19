---
title: 如何设置流量镜像
description: 
weight: 51
---

## 安装示例程序

如果你还没有安装示例程序，请参照 [快速开始](/zh/docs/v1.x/quickstart/) 安装 Aeraki，Istio 及示例程序。

安装完成后，可以看到集群中增加了下面两个 NS，这两个 NS 中分别安装了基于 MetaProtocol 实现的 Dubbo 和 Thrift 协议的示例程序。
你可以选用任何一个程序进行测试。

```bash
➜  ~ kubectl get ns|grep meta
meta-dubbo        Active   16m
meta-thrift       Active   16m
```

我们使用 meta-dubbo ns 中的 dubbo 示例应用来测试流量镜像功能。示例应用中安装了 v1 和 v2 两个版本的 provider。

v1 版本的 demo provider：

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dubbo-sample-provider-v1
  labels:
    app: dubbo-sample-provider
spec:
  selector:
    matchLabels:
      app: dubbo-sample-provider
  replicas: 1
  template:
    metadata:
      annotations:
        sidecar.istio.io/bootstrapOverride: aeraki-bootstrap-config
        sidecar.istio.io/proxyImage: aeraki/meta-protocol-proxy-debug:1.1.1
      labels:
        app: dubbo-sample-provider
        version: v1
        service_group: user
    spec:
      containers:
        - name: dubbo-sample-provider
          image: aeraki/dubbo-sample-provider
          ports:
            - containerPort: 20880
```

v2版本的 demo provider:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dubbo-sample-provider-v2
  labels:
    app: dubbo-sample-provider
spec:
  selector:
    matchLabels:
      app: dubbo-sample-provider
  replicas: 1
  template:
    metadata:
      annotations:
        sidecar.istio.io/bootstrapOverride: aeraki-bootstrap-config
        sidecar.istio.io/proxyImage: aeraki/meta-protocol-proxy-debug:1.1.1
      labels:
        app: dubbo-sample-provider
        version: v2
        service_group: batchjob
    spec:
      containers:
        - name: dubbo-sample-provider
          image: aeraki/dubbo-sample-provider
          ports:
            - containerPort: 20880
```

通过 ServiceEntry 定义服务：

```yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: dubbo-demoservice
  namespace: meta-dubbo
  annotations:
    interface: org.apache.dubbo.samples.basic.api.DemoService
spec:
  hosts:
    - org.apache.dubbo.samples.basic.api.demoservice
  ports:
    - number: 20880
      name: tcp-metaprotocol-dubbo
      protocol: TCP
  workloadSelector:
    labels:
      app: dubbo-sample-provider
  resolution: STATIC
```

## 将请求路由到 v1

通过下面的 MetaRouter 路由规则将 consumer 的请求路由到 v1。

```bash
➜  ~ kubectl apply -f- <<EOF
apiVersion: metaprotocol.aeraki.io/v1alpha1
kind: MetaRouter
metadata:
  name: test-metaprotocol-dubbo-route
  namespace: meta-dubbo
spec:
  hosts:
    - org.apache.dubbo.samples.basic.api.demoservice
  routes:
    - name: traffic-mirroring
      route:
        - destination:
            host: org.apache.dubbo.samples.basic.api.demoservice
            subset: v1
EOF
```

等待约 30 秒，然后使用下面的命令查看 consumer 的输出，可以看到请求被发送到了 v1。

```bash
➜  ~ aerakictl_app_log consumer meta-dubbo --tail 10
Hello Aeraki, response from dubbo-sample-provider-v2-6dfc65755-xrhxp/172.16.0.184
Hello Aeraki, response from dubbo-sample-provider-v1-6d67cc67fd-chfrk/172.16.0.183
Hello Aeraki, response from dubbo-sample-provider-v2-6dfc65755-xrhxp/172.16.0.184
Hello Aeraki, response from dubbo-sample-provider-v1-6d67cc67fd-chfrk/172.16.0.183
Hello Aeraki, response from dubbo-sample-provider-v1-6d67cc67fd-chfrk/172.16.0.183
Hello Aeraki, response from dubbo-sample-provider-v1-6d67cc67fd-chfrk/172.16.0.183
Hello Aeraki, response from dubbo-sample-provider-v1-6d67cc67fd-chfrk/172.16.0.183
Hello Aeraki, response from dubbo-sample-provider-v1-6d67cc67fd-chfrk/172.16.0.183
Hello Aeraki, response from dubbo-sample-provider-v1-6d67cc67fd-chfrk/172.16.0.183
Hello Aeraki, response from dubbo-sample-provider-v1-6d67cc67fd-chfrk/172.16.0.183
```

查看 v1 版本 provider 的日志，可以看到 v1 版本收到了 consumer 发出的请求。

```bash
➜  ~ kubectl  -n meta-dubbo logs dubbo-sample-provider-v1-6d67cc67fd-chfrk  --tail 0 -f
[04:00:42] Hello Aeraki, request from consumer: /127.0.0.6:46971
[04:00:47] Hello Aeraki, request from consumer: /127.0.0.6:46971
[04:00:52] Hello Aeraki, request from consumer: /127.0.0.6:46971
[04:00:57] Hello Aeraki, request from consumer: /127.0.0.6:46971
```

查看 v2 版本的 provider 日志，可以看到 v2 版本不会收到请求。

```bash
➜  ~ kubectl  -n meta-dubbo logs dubbo-sample-provider-v2-6dfc65755-xrhxp  --tail 0 -f

```

## 将流量镜像到 v2

修改 MetaRouter 路由规则，在将请求发送到 v1 的同时，将流量镜像到 v2。

```bash
➜  ~ kubectl apply -f- <<EOF
apiVersion: metaprotocol.aeraki.io/v1alpha1
kind: MetaRouter
metadata:
  name: test-metaprotocol-dubbo-route
  namespace: meta-dubbo
spec:
  hosts:
    - org.apache.dubbo.samples.basic.api.demoservice
  routes:
    - name: traffic-mirroring
      route:
        - destination:
            host: org.apache.dubbo.samples.basic.api.demoservice
            subset: v1
      mirror:
        host: org.apache.dubbo.samples.basic.api.demoservice
        subset: v2
      mirrorPercentage:
        value: 100.0
EOF
```

查看 v2 版本的 provider 日志，可以看到 v2 版本收到了请求。

```bash
➜  ~ kubectl  -n meta-dubbo logs dubbo-sample-provider-v2-6dfc65755-xrhxp  --tail 0 -f
[04:08:12] Hello Aeraki, request from consumer: /127.0.0.6:57267
[04:08:17] Hello Aeraki, request from consumer: /127.0.0.6:57267
[04:08:22] Hello Aeraki, request from consumer: /127.0.0.6:57267
[04:08:27] Hello Aeraki, request from consumer: /127.0.0.6:57267
[04:08:32] Hello Aeraki, request from consumer: /127.0.0.6:57267
```

当然， v1 版本的 provider 也会收到请求，因为缺省的路由策略是把请求发送到 v1。

```bash
➜  ~ kubectl -n meta-dubbo logs dubbo-sample-provider-v1-6d67cc67fd-chfrk  --tail 0 -f
[04:16:18] Hello Aeraki, request from consumer: /127.0.0.6:46971
[04:16:23] Hello Aeraki, request from consumer: /127.0.0.6:46971
[04:16:28] Hello Aeraki, request from consumer: /127.0.0.6:46971
[04:16:33] Hello Aeraki, request from consumer: /127.0.0.6:46971
[04:16:38] Hello Aeraki, request from consumer: /127.0.0.6:46971
```

镜像流量采用了发送后即丢弃的策略，不会处理响应消息。我们可以从 consumer 的日志验证这一点，在 consumer 日志中可以看到，只收到了来自 v1 provider 的响应。

```bash
➜  ~ aerakictl_app_log consumer meta-dubbo --tail 0 -f
Hello Aeraki, response from dubbo-sample-provider-v1-6d67cc67fd-chfrk/172.16.0.183
Hello Aeraki, response from dubbo-sample-provider-v1-6d67cc67fd-chfrk/172.16.0.183
Hello Aeraki, response from dubbo-sample-provider-v1-6d67cc67fd-chfrk/172.16.0.183
Hello Aeraki, response from dubbo-sample-provider-v1-6d67cc67fd-chfrk/172.16.0.183
```

## 理解原理

在向 Sidecar Proxy 下发的配置中， Aeraki 在服务对应的 Outbound Listener 的 FilterChain 中设置了 MetaProtocol Proxy，并在 MetaProtocol Proxy 配置中指定 Aeraki 为 RDS 服务器。

Aeraki 会将 MetaRouter 中配置的路由规则翻译为 MetaProtocol Proxy 的路由规则，通过 Aeraki 内置的 RDS 服务器下发给 MetaProtocol Proxy。

可以通过下面的命令查看 sidecar proxy 的配置：

``` bash
aerakictl_sidecar_config consumer meta-dubbo |fx
```
其中关于流量镜像相关的配置信息在 meta_protocol_proxy 的 RoutesConfigDump 中，如下所示：

```yaml
  {
   "@type": "type.googleapis.com/aeraki.meta_protocol_proxy.admin.v1alpha.RoutesConfigDump",
   "dynamic_route_configs": [
    {
     "version_info": "1658290077",
     "route_config": {
      "@type": "type.googleapis.com/aeraki.meta_protocol_proxy.config.route.v1alpha.RouteConfiguration",
      "name": "org.apache.dubbo.samples.basic.api.demoservice_20880",
      "routes": [
       {
        "name": "traffic-mirroring",
        "route": {
         "cluster": "outbound|20880|v1|org.apache.dubbo.samples.basic.api.demoservice",
         "request_mirror_policies": [
          {
           "cluster": "outbound|20880|v2|org.apache.dubbo.samples.basic.api.demoservice",
           "runtime_fraction": {
            "default_value": {
             "numerator": 1000000,
             "denominator": "MILLION"
            }
           }
          }
         ]
        }
       }
      ]
     },
     "last_updated": "2022-07-20T04:07:57.564Z"
    }
```







