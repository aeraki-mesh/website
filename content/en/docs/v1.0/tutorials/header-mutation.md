---
title: How to modify headers
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

## 修改请求消息 Header

Aeraki 支持采用 MetaRouter 来修改消息 Header，我们可以采用下面的规则来修改请求消息的 Header。

> 备注：Aeraki 和 MetaProtocol 在框架层支持了消息 header 的修改，至于某一种协议是否支持 header 修改，则取决于该协议 codec 的实现。如果要支持 header 修改，codec 实现需要处理 MetaProtocol 框架层传入的 mutation 结构体，在协议编码时将 mutation 结构体中的 key/value 键值写入到消息中。

创建一条 MetaRouter 规则，增加 foo/bar, foo1/bar1 两个消息头：

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
  routes:
    - name: header-mutation
      route:
        - destination:
            host: thrift-sample-server.meta-thrift.svc.cluster.local
      requestMutation:
        - key: foo
          value: bar
        - key: foo1
          value: bar1
EOF
```

使用 aerakictl 命令来查看客户端的 sidecar 日志，可以看到增加的消息头：

```bash
➜  ~ aerakictl_sidecar_enable_debug client meta-thrift
➜  ~ aerakictl_sidecar_log client meta-thrift  --tail 0 -f|grep mutation
2022-03-10T06:42:25.605305Z	info	envoy filter	thrift: codec mutation foo : bar
2022-03-10T06:42:25.605316Z	info	envoy filter	thrift: codec mutation foo1 : bar1
```

## 理解原理

在向 Sidecar Proxy 下发的配置中， Aeraki 在服务对应的 Outbound Listener 的 FilterChain 中设置了 MetaProtocol Proxy，并在 MetaProtocol Proxy 配置中指定 Aeraki 为 RDS 服务器。

Aeraki 会将 MetaRouter 中配置的路由规则翻译为 MetaProtocol Proxy 的路由规则，通过 Aeraki 内置的 RDS 服务器下发给 MetaProtocol Proxy。

可以通过下面的命令查看 sidecar proxy 的配置：

``` bash
aerakictl_sidecar_config client meta-thrift |fx
```

其中 Thrift 服务的 Outbound Listener 中的 MetaProtocol Proxy 配置如下所示：

```yaml
{
 "name": "envoy.filters.network.meta_protocol_proxy",
 "typed_config": {
  "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
  "type_url": "type.googleapis.com/aeraki.meta_protocol_proxy.v1alpha.MetaProtocolProxy",
  "value": {
   "stat_prefix": "outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local",
   "application_protocol": "thrift",
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
    "route_config_name": "thrift-sample-server.meta-thrift.svc.cluster.local_9090"
   },
   "codec": {
    "name": "aeraki.meta_protocol.codec.thrift"
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
@type": "type.googleapis.com/aeraki.meta_protocol_proxy.admin.v1alpha.RoutesConfigDump",
dynamic_route_configs": [
{
 "version_info": "1641896797",
 "route_config": {
      "@type": "type.googleapis.com/aeraki.meta_protocol_proxy.config.route.v1alpha.RouteConfiguration",
      "name": "thrift-sample-server.meta-thrift.svc.cluster.local_9090",
      "routes": [
       {
        "name": "header-mutation",
        "route": {
         "cluster": "outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local"
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
     "last_updated": "2022-03-10T06:26:24.083Z"
    }
   ]
 },
 "last_updated": "2022-01-11T10:26:37.357Z"
}
```







