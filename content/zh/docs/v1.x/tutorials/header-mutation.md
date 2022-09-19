---
title: 如何修改消息头
description: 
weight: 60
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

## 修改请求消息 Header

Aeraki 支持采用 MetaRouter 来修改消息 Header，我们可以采用下面的规则来修改请求消息的 Header。

> 备注：Aeraki 和 MetaProtocol 在框架层支持了消息 header 的修改，至于某一种协议是否支持 header 修改，则取决于该协议 codec 的实现。如果要支持 header 修改，codec 实现需要处理 MetaProtocol 框架层传入的 mutation 结构体，在协议编码时将 mutation 结构体中的 key/value 键值写入到消息中。

MetaProtocol Dubbo codec 实现已经提供了 header 修改的能力。下面我们创建一条 MetaRouter 规则，为发送给 demoservice 的 Dubbo 请求增加 foo/bar, foo1/bar1 两个消息头：

```bash
kubectl apply -f- <<EOF
apiVersion: metaprotocol.aeraki.io/v1alpha1
kind: MetaRouter
metadata:
  name: test-metaprotocol-dubbo-route
  namespace: meta-dubbo
spec:
  hosts:
    - org.apache.dubbo.samples.basic.api.demoservice
  routes:
    - name: header-mutation
      route:
        - destination:
            host: org.apache.dubbo.samples.basic.api.demoservice
      requestMutation:
        - key: foo
          value: bar
        - key: foo1
          value: bar1
EOF
```

使用 aerakictl 命令来查看 provider 的日志，可以看到 provider 端收到的消息中已经带上了增加的 header：

```bash
➜  ~  aerakictl_app_log provider meta-dubbo --tail 0 -f
[05:06:34] Hello Aeraki, request from consumer: /127.0.0.6:60521
Message headers:
 key: input value: 297
 key: remote.application value: dubbo-sample-consumer
 key: foo value: bar
 key: foo1 value: bar1
```

## 理解原理

在向 Sidecar Proxy 下发的配置中， Aeraki 在服务对应的 Outbound Listener 的 FilterChain 中设置了 MetaProtocol Proxy，并在 MetaProtocol Proxy 配置中指定 Aeraki 为 RDS 服务器。

Aeraki 会将 MetaRouter 中配置的路由规则翻译为 MetaProtocol Proxy 的路由规则，通过 Aeraki 内置的 RDS 服务器下发给 MetaProtocol Proxy。

可以通过下面的命令查看 sidecar proxy 的配置：

``` bash
aerakictl_sidecar_config consumer meta-dubbo |fx
```

其中 Dubbo 服务的 Outbound Listener 中的 MetaProtocol Proxy 配置如下所示：

```yaml
{
 "name": "envoy.filters.network.meta_protocol_proxy",
 "typed_config": {
  "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
  "type_url": "type.googleapis.com/aeraki.meta_protocol_proxy.v1alpha.MetaProtocolProxy",
  "value": {
   "stat_prefix": "outbound|20880||org.apache.dubbo.samples.basic.api.demoservice",
   "application_protocol": "dubbo",
   "rds": {
    "config_source": {
     "api_config_source": {
      "api_type": "GRPC",
      "grpc_services": [
       {
        "envoy_grpc": {
         "cluster_name": "aeraki-xds"
        }
       }
      ],
      "transport_api_version": "V3"
     },
     "resource_api_version": "V3"
    },
    "route_config_name": "org.apache.dubbo.samples.basic.api.demoservice_20880"
   },
   "codec": {
    "name": "aeraki.meta_protocol.codec.dubbo"
   },
   "meta_protocol_filters": [
    {
     "name": "aeraki.meta_protocol.filters.router"
    }
   ]
  }
 }
}
```

在导出的文件中还可以查看到目前 Proxy 中生效的 RDS 路由信息，可以看到路由中增加了相应的 header，如下所示：

```yaml
{
@type": "type.googleapis.com/aeraki.m{
 "version_info": "1658895106",
 "route_config": {
  "@type": "type.googleapis.com/aeraki.meta_protocol_proxy.config.route.v1alpha.RouteConfiguration",
  "name": "org.apache.dubbo.samples.basic.api.demoservice_20880",
  "routes": [
   {
    "name": "header-mutation",
    "route": {
     "cluster": "outbound|20880||org.apache.dubbo.samples.basic.api.demoservice"
    },
    "request_mutation": [
     {
      "key": "foo",
      "value": "bar"
     },
     {
      "key": "foo1",
      "value": "bar1"
     }
    ]
   }
  ]
 },
 "last_updated": "2022-07-27T04:11:46.143Z"
}
```







