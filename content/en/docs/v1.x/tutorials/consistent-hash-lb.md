---
title: How to configure load balancing policy
description: 
weight: 45
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

## 缺省的负载均衡算法

Aeraki Mesh 缺省采用 Round Robin 负载均衡算法，在没有进行 LB 配置的情况下，客户端的请求会被 sidecar proxy 依次发送到 upstream cluster 中的每一个 endpoint 上。meta-dubbo ns 中的 dubbo 服务中有两个 pod，查看 meta-dubbo ns 中示例程序的输入，我们可以看到客户端请求被依次发送到这两个 pod 上：

```bash
➜  ~ aerakictl_app_log consumer meta-dubbo --tail 0  -f
Hello Aeraki, response from dubbo-sample-provider-v2-5576f45c6c-897g5/172.16.0.8
Hello Aeraki, response from dubbo-sample-provider-v1-98c5b9fd6-gslv6/172.16.0.69
Hello Aeraki, response from dubbo-sample-provider-v2-5576f45c6c-897g5/172.16.0.8
Hello Aeraki, response from dubbo-sample-provider-v1-98c5b9fd6-gslv6/172.16.0.69
```

## 设置采用 Consistent Hash

通过下面的命令创建一个 Destination Rule，指定 dubbo 示例程序采用 Consistent Hash 负载均衡算法，并采用 Metadata 中的 method 属性来生成负载均衡所需的哈希值。

```bash
➜  ~ kubectl apply -f- <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dubbo-sample-provider
  namespace: meta-dubbo
spec:
  host: org.apache.dubbo.samples.basic.api.demoservice
  trafficPolicy:
    loadBalancer:
      consistentHash:
        httpHeaderName: method
EOF
```

> 备注： Aeraki Mesh 采用了 Istio 的 Destination Rule 中的 trafficPolicy 来设置负载均衡。httpHeaderName 对应于 MetaProtocol codec 解码生成的 Metadata 中的属性 key。Metadata 中的任何属性都可以用于生成 Consistent Hash 负载均衡中使用的哈希值。

此时查看客户端的输出，可以看到客户端的请求被固定发送到同一个服务端。

```bash
➜  ~ aerakictl_app_log consumer meta-dubbo --tail 0  -f
Hello Aeraki, response from dubbo-sample-provider-v2-5576f45c6c-897g5/172.16.0.8
Hello Aeraki, response from dubbo-sample-provider-v2-5576f45c6c-897g5/172.16.0.8
Hello Aeraki, response from dubbo-sample-provider-v2-5576f45c6c-897g5/172.16.0.8
Hello Aeraki, response from dubbo-sample-provider-v2-5576f45c6c-897g5/172.16.0.8
```

## 理解原理

Aeraki 会将 Destination Rule 中配置的 LB 策略通过 RDS 下发给 MetaProtocol Proxy。

可以通过下面的命令查看 dubbo 服务 的 cluster 和路由配置：

``` bash
aerakictl_sidecar_config consumer meta-dubbo |fx
```

从 `outbound|20880||org.apache.dubbo.samples.basic.api.demoservice` cluster 的配置中可以看到该 cluster 指定采用 `RING_HAS` 负载均衡策略。

```yaml
{
     "version_info": "2022-03-29T04:56:04Z/12",
     "cluster": {
      "@type": "type.googleapis.com/envoy.config.cluster.v3.Cluster",
      "name": "outbound|20880||org.apache.dubbo.samples.basic.api.demoservice",
      "type": "EDS",
      "eds_cluster_config": {
       "eds_config": {
        "ads": {},
        "initial_fetch_timeout": "0s",
        "resource_api_version": "V3"
       },
       "service_name": "outbound|20880||org.apache.dubbo.samples.basic.api.demoservice"
      },
      "connect_timeout": "10s",
      "lb_policy": "RING_HASH",
      ......
```

再查看 Aeraki 下发的 RDS 路由，其中 `org.apache.dubbo.samples.basic.api.demoservice_20880` 这条路由的配置如下所示。可以看到路由中设置的 hash_policy，指定采用 metadata 中的 method 来生成 Consistent hash 负载均衡的哈希值。 

```yaml
{
     "version_info": "1648529765",
     "route_config": {
      "@type": "type.googleapis.com/aeraki.meta_protocol_proxy.config.route.v1alpha.RouteConfiguration",
      "name": "org.apache.dubbo.samples.basic.api.demoservice_20880",
      "routes": [
       {
        "name": "default",
        "route": {
         "cluster": "outbound|20880||org.apache.dubbo.samples.basic.api.demoservice",
         "hash_policy": [
          "method"
         ]
        }
       }
      ]
     },
     "last_updated": "2022-03-29T04:56:05.817Z"
    }
   ]
  }
  ```