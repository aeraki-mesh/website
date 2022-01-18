---
title: 如何设置本地限流
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

Aeraki 的限流规则设计直观而灵活，既支持对一个服务的所有入向请求进行限流，也支持按照不同的条件对一个服务器的请求进行细粒度的限流控制。

## 对服务的所有入向请求进行限流

下面的规则可以对 thrift-sample-server.meta-thrift.svc.cluster.local 服务的所有入向请求进行限流，限流设置为 2次请求/每分钟。

```bash
kubectl apply -f- <<EOF
apiVersion: metaprotocol.aeraki.io/v1alpha1
kind: MetaRouter
metadata:
  name: test-metaprotocol-thrift-route
  namespace: meta-thrift
spec:
  hosts:
  - thrift-sample-server.meta-thrift.svc.cluster.local
  localRateLimit:
    tokenBucket:
      fillInterval: 60s
      maxTokens: 2
      tokensPerFill: 2
EOF
```

> 备注：因为本地限流是在每一个服务实例上单独进行处理的，因此当服务有多个实例，实际的限流效果为限流次数乘以实例数量。


使用 aerakictl 命令来查看客户端的应用日志，可以看到客户端每分钟只能成功执行 4 次请求（有两个服务实例，每个服务实例限流为每分钟 2 次）：

```bash
➜  ~ aerakictl_app_log client meta-thrift -f --tail 10
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-842l6/172.17.0.40
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-hpx7n/172.17.0.41
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-842l6/172.17.0.40
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-hpx7n/172.17.0.41
org.apache.thrift.TApplicationException: meta protocol local rate limit: request '5' has been rate limited
        at org.apache.thrift.TServiceClient.receiveBase(TServiceClient.java:79)
        at org.aeraki.HelloService$Client.recv_sayHello(HelloService.java:61)
        at org.aeraki.HelloService$Client.sayHello(HelloService.java:48)
        at org.aeraki.HelloClient.main(HelloClient.java:44)
Connected to thrift-sample-server
org.apache.thrift.TApplicationException: meta protocol local rate limit: request '1' has been rate limited
...
```

## 按条件对服务的入向请求进限流

Aeraki 支持按照条件为服务设置多个限流规则，以满足细粒度的限流要求。例如按照用户或者对接口对请求进行分组，并对每个分组设置不同的限流规则。 

分组限流的匹配条件和路由匹配条件相同，任何可以从请求数据包中提取出来的属性都可以用于限流规则的匹配条件。

例如下面的规则为 sayHello 和 ping 两个接口分别设置了不同的限流条件：

```yaml
apiVersion: metaprotocol.aeraki.io/v1alpha1
kind: MetaRouter
metadata:
  name: test-metaprotocol-thrift-route
  namespace: meta-thrift
spec:
  hosts:
  - thrift-sample-server.meta-thrift.svc.cluster.local
  localRateLimit:
    conditions:
    - match:
        attributes:
          method:
            exact: sayHello
      tokenBucket:
        fillInterval: 60s
        maxTokens: 10
        tokensPerFill: 10
    - match:
        attributes:
          method:
            exact: ping
      tokenBucket:
        fillInterval: 60s
        maxTokens: 100
        tokensPerFill: 100
```

## 同时设置按服务和按条件的限流规则

可以同时设置服务粒度的限流规则和按照条件的限流规则，这适用于需要对一个服务的所有请求设置一个整体的限流规则，同时又需要对某一组或者几组请求设置例外的情况。

例如下面的限流规则为服务设置了一个 1000 条/分钟的整体限流规则，同时单独为 ping 接口设置了 100 条/分钟的限流条件。

```yaml
apiVersion: metaprotocol.aeraki.io/v1alpha1
kind: MetaRouter
metadata:
  name: test-metaprotocol-thrift-route
  namespace: meta-thrift
spec:
  hosts:
  - thrift-sample-server.meta-thrift.svc.cluster.local
  localRateLimit:
    tokenBucket:
      fillInterval: 60s
      maxTokens: 1000
      tokensPerFill: 1000
    conditions:
    - match:
        attributes:
          method:
            exact: ping
      tokenBucket:
        fillInterval: 60s
        maxTokens: 100
        tokensPerFill: 100
```

## 理解原理

在向 Sidecar Proxy 下发的配置中， Aeraki 在 VirtualInbound Listener 中服务对应的 FilterChain 中设置了 MetaProtocol Proxy。

Aeraki 会将 MetaRouter 中配置的限流规则翻译为 local rate limit filter 的限流配置，通过 Aeraki 下发给 MetaProtocol Proxy。

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
     "name": "aeraki.meta_protocol.filters.local_ratelimit",
     "config": {
      "@type": "type.googleapis.com/aeraki.meta_protocol_proxy.filters.local_ratelimit.v1alpha.LocalRateLimit",
      "stat_prefix": "thrift-sample-server.meta-thrift.svc.cluster.local",
      "token_bucket": {
       "max_tokens": 2,
       "tokens_per_fill": 2,
       "fill_interval": "60s"
      }
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







