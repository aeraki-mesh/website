---
title: How to modify headers
description: 
weight: 60
---

## Installing the demo program

If you haven't installed the demo program yet, Please refer to [Quick Start](../../quickstart/) to install Aeraki, Istio, and the demo.

After installation, you can see that the following two NSs have been added to the cluster, and the Dubbo and Thrift demo applications are installed in these two NSs. You can choose either of them to test.

```bash
➜  ~ kubectl get ns|grep meta
meta-dubbo        Active   16m
meta-thrift       Active   16m
```

## Modify the request header

Aeraki supports header mutation. For example, we can use the following MetaRouter to modify the request message header.

> Note: Aeraki and MetaProtocol framework support header mutation. Whether a certain application protocol supports header mutation depends on the implementation of the codec of that protocol. To support header modification, the codec implementation needs to get out the key/value pairs in the mutation structure and wrriten them into the outgoing message during the encoding phase.

Create a MetaRouter rule and add two message headers foo/bar, foo1/bar1 in the `requestMutation` part:

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

Use the aerakictl command to view the client-side log, you can see the added header:

```bash
➜  ~ aerakictl_sidecar_enable_debug client meta-thrift
➜  ~ aerakictl_sidecar_log client meta-thrift  --tail 0 -f|grep mutation
2022-03-10T06:42:25.605305Z	info	envoy filter	thrift: codec mutation foo : bar
2022-03-10T06:42:25.605316Z	info	envoy filter	thrift: codec mutation foo1 : bar1
```

## Understand what happened

In the configuration delivered to the Sidecar Proxy, Aeraki sets the MetaProtocol Proxy in the FilterChain of the Outbound Listener corresponding to the service, and specifies Aeraki as the RDS server in the MetaProtocol Proxy configuration.

Aeraki will translate the routing rules configured in MetaRouter into the routing rules of MetaProtocol Proxy, and send them to MetaProtocol Proxy through Aeraki's built-in RDS server.

You can view the configuration of the sidecar proxy with the following command:

``` bash
aerakictl_sidecar_config client meta-thrift |fx
```

The MetaProtocol Proxy configuration in the Outbound Listener of the Thrift service is as follows:

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

You can also view the RDS routing information that is currently in effect in the Proxy in the exported file, and you can catch sight of that the corresponding header is added to the routing, as shown below:

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

