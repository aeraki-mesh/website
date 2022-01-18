---
title: 如何设置全局限流
description: 
weight: 200
---

## 安装示例程序

如果你还没有安装示例程序，请参照 [快速开始](/zh/docs/v1.0/quickstart/) 安装 Aeraki，Istio 及示例程序。

安装完成后，可以看到集群中增加了下面两个 NS，这两个 NS 中分别安装了基于 MetaProtocol 实现的 Dubbo 和 Thrift 协议的示例程序。
你可以选用任何一个程序进行测试。

```bash
➜  ~ kubectl get ns|grep meta
meta-dubbo        Active   16m
meta-thrift       Active   16m
```

## 什么是全局限流

和[本地限流](/zh/docs/v1.0/tutorials/local-rate-limit/)不同的是，在使用全局限流时，所有的服务实例共用一个限流配额。全局限流通过一个限流服务器来实现在多个服务实例之间共享限流配额。在收到请求时，服务服务端的 Sidecar Proxy 会先向限流服务器发送一个限流查询请求，限流服务器在其自身的配置文件中读取限流的规则，根据规则判断一个限流请求是否触发了限流条件，然后将限流结果返回给 Sidecar Proxy。 Sidecar Proxy 根据该限流结果决定是禁止本次请求还是继续处理本次请求。

![](../global-rate-limit.png)

## 在何时使用全局限流
全局限流的特点是限流判断在限流服务器处统一进行的，因此不会受到服务实例数量的影响。但全局限流也会引入限流服务器这额外一跳，会带来附加的网络延迟。在大量客户请求的情况下，限流服务器自身是一个潜在的瓶颈点，部署和管理比本地限流更为复杂。

如果用限流的目的是为了将服务实例的压力控制在合理的范围内，建议使用本地限流。因为本地限流是在每个服务实例的 Sidecar Proxy 处分别处理的，因此在这种场景下本地限流对服务实例的入向请求控制更为精确和合理。如果配合适当的 HPA 策略，在现有服务实例负载都打满的情况下还可以对服务实例进行水平扩容，保障服务的稳定运行。

如果限流的目的是为了对某一个资源的访问实施全局访问策略，则应该使用全局限流。一个典型的例子是按照用户等级设置用户可以访问服务的频率。例如 Docker Hub 就按照用户等级对用户拉取镜像的次数实施了不同的限流策略，付费用户比免费用户享有更高次数的镜像拉取限额。

## 部署限流服务器

在示例程序中已经部署了限流服务器，并通过配置文件配置了限流规则，无需再单独部署。

全局限流的限流规则需要在限流服务器的配置文件中进行设置。下面的限流规则表示对 sayHello 接口设置 10条/每分钟 的限流。

```yaml
domain: production
descriptors:
  - key: method
    value: "sayHello"
    rate_limit:
      unit: minute
      requests_per_unit: 5
```

部署的相关脚本参见：https://github.com/aeraki-mesh/aeraki/tree/master/demo/metaprotocol-thrift/rate-limit-server

> 备注：因为全局限流的判断逻辑是在限流服务器中执行的，因此限流规则需要在限流服务器的配置文件中进行设置。

## 为服务启用限流

通过 MetaRouter 为服务器启用限流，启用后，服务的 Sidecar Proxy 在收到请求后会向限流服务器发起限流请求，并根据请求的返回结果决定是继续处理该请求还是终止请求。

下面的全局限流配置表示对 thrift-sample-server.meta-thrift.svc.cluster.local 服务的 sayHello 接口进行限流。向限流服务器发送的限流请求中的 domain 的值为 production，并且会将 method 属性作为 descriptor 加入限流请求中。注意这里设置的 domain 和 descriptor 必须和限流服务器中的限流配置相吻合。

MetaRouter 中的限流设置中可以设置 match 条件，如果 match 条件存在，则表示只将符合条件的请求发送到限流服务器进行限流判断。如果 match 条件未设置，则表示该服务的所有请求都会发送到限流服务器进行判断。合理使用 match 条件可以在本地过滤掉不需要限流的请求，提高请求处理效率。
```yaml
kubectl apply -f- <<EOF
apiVersion: metaprotocol.aeraki.io/v1alpha1
kind: MetaRouter
metadata:
  name: test-metaprotocol-thrift-route
  namespace: meta-thrift
spec:
  hosts:
  - thrift-sample-server.meta-thrift.svc.cluster.local
  globalRateLimit:
    domain: production
    match:
      attributes:
        method:
          exact: sayHello
    rateLimitService: outbound|8081||rate-limit-server.meta-thrift.svc.cluster.local
    requestTimeout: 100ms
    denyOnFail: true
    descriptors:
    - property: method
      descriptorKey: method
EOF
```

使用 aerakictl 命令来查看客户端的应用日志，可以看到客户端每分钟只能成功执行 5 次请求：

```bash
➜  ~ aerakictl_app_log client meta-thrift -f --tail 10
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-842l6/172.17.0.40
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-hpx7n/172.17.0.41
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-842l6/172.17.0.40
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-hpx7n/172.17.0.41
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-842l6/172.17.0.40
org.apache.thrift.TApplicationException: meta protocol local rate limit: request '6' has been rate limited
        at org.apache.thrift.TServiceClient.receiveBase(TServiceClient.java:79)
        at org.aeraki.HelloService$Client.recv_sayHello(HelloService.java:61)
        at org.aeraki.HelloService$Client.sayHello(HelloService.java:48)
        at org.aeraki.HelloClient.main(HelloClient.java:44)
Connected to thrift-sample-server
org.apache.thrift.TApplicationException: meta protocol local rate limit: request '7' has been rate limited
...
```

## 理解原理

在向 Sidecar Proxy 下发的配置中， Aeraki 在 VirtualInbound Listener 中服务对应的 FilterChain 中设置了 MetaProtocol Proxy。

Aeraki 会将 MetaRouter 中配置的限流规则翻译为 global rate limit filter 的限流配置，通过 Aeraki 下发给 MetaProtocol Proxy。

可以通过下面的命令查看服务的 sidecar proxy 的配置：

``` bash
aerakictl_sidecar_config server-v1 meta-thrift |fx
```

其中 Thrift 服务的 Inbound Listener 中的 MetaProtocol Proxy 配置如下所示：

```yaml
{
 "name": "envoy.filters.network.meta_protocol_proxy",
 "typed_config": {
  "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
  "type_url": "type.googleapis.com/aeraki.meta_protocol_proxy.v1alpha.MetaProtocolProxy",
  "value": {
   "stat_prefix": "inbound|9090||",
   "application_protocol": "thrift",
   "route_config": {
    "name": "inbound|9090||",
    "routes": [
     {
      "route": {
       "cluster": "inbound|9090||"
      }
     }
    ]
   },
   "codec": {
    "name": "aeraki.meta_protocol.codec.thrift"
   },
   "meta_protocol_filters": [
    {
     "name": "aeraki.meta_protocol.filters.ratelimit",
     "config": {
      "@type": "type.googleapis.com/aeraki.meta_protocol_proxy.filters.ratelimit.v1alpha.RateLimit",
      "match": {
       "metadata": [
        {
         "name": "method",
         "exact_match": "sayHello"
        }
       ]
      },
      "domain": "production",
      "timeout": "0.100s",
      "failure_mode_deny": true,
      "rate_limit_service": {
       "grpc_service": {
        "envoy_grpc": {
         "cluster_name": "outbound|8081||rate-limit-server.meta-thrift.svc.cluster.local"
        }
       }
      },
      "descriptors": [
       {
        "property": "method",
        "descriptor_key": "method"
       }
      ]
     }
    },
    {
     "name": "aeraki.meta_protocol.filters.router"
    }
   ]
  }
 }
}
```







