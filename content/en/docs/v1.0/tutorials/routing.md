---
title: How to configure traffic routing
description: 
weight: 10
---

## Installing the demo program

If you haven't installed the demo program,Please refer to [Quick Start](./../quickstart.md) to install Aeraki, Istio, and demo programs.

After the installation, you can see that the following two NSs have been added to the cluster, and the demo applications for Dubbo and Thrift protocols based on the MetaProtocol implementation are installed in these two NSs.
You can choose either of them to test.

```bash
➜  ~ kubectl get ns|grep meta
meta-dubbo        Active   16m
meta-thrift       Active   16m
```

## Request-level load balancing

Istio uses TCP proxy to proxy non-HTTP client requests, and all requests from the same client TCP connection are sent to a single server instance. This leads to a problem: when clients use long connections, multiple server instances do not receive a balanced number of requests, and when the server side is overloaded, the incoming requests that come through the existing connections can't be distributed to new instances even though more instances are added to the pool of the server cluster to share the load.

Aeraki supports layer-7(request level) load balancing for  any protocol developed based on MetaProtocol, so without any configuration, the client's proxy sends requests evenly to two different versions of the server side.
Let's use the aerakictl command to view the client's application logs and see that multiple requests on the same client connection are sent sequentially to two servers, v1 and v2.

```bash
➜  ~ aerakictl_app_log client meta-thrift -f --tail 10
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
```

## Routes requests to a specified version based on arbitrary properties

MetaProtocol supports very flexible route matching conditions, any property that can be parsed from the protocol packet can be used for route matching conditions.

> Note: Aeraki will create Listener according to the VIP of the service, and each service will have its own Listener, which avoids the routing table expansion problem caused by multiple services on the same port of the HTTP protocol, and the routing table only contains the routing information related to this service, thus greatly improving the route matching efficiency.

Create a MetaRouter routing rule that routes the request to v1.

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
    - name: v1
      match:
        attributes:
          method:
            exact: sayHello
      route:
        - destination:
            host: thrift-sample-server.meta-thrift.svc.cluster.local
            subset: v1
EOF
```

Using the aerakictl command to view the client application logs, you can see that all requests from the client are routed to the v1 version.

```bash
➜  ~ aerakictl_app_log client meta-thrift -f --tail 10
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
```

## Traffic splitting

Use MetaRouter routing rules to send client flows to different versions of the service in specified proportions.

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
    - name: traffic-split
      match:
        attributes:
          method:
            exact: sayHello
      route:
        - destination:
            host: thrift-sample-server.meta-thrift.svc.cluster.local
            subset: v1
          weight: 20
        - destination:
            host: thrift-sample-server.meta-thrift.svc.cluster.local
            subset: v2
          weight: 80
EOF
```

Using the aerakictl command to view the client application logs, you can see that the client requests were sent to v1 and v2 at the specified rate set in MetaRouter.

```bash
➜  ~ aerakictl_app_log client meta-thrift -f --tail 10
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v1-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
```

## Understand what happened

In the configuration sent to the Sidecar Proxy, Aeraki sets up the MetaProtocol Proxy in the FilterChain of the Outbound Listener corresponding to the service, and specifies Aeraki as the RDS server in the MetaProtocol Proxy configuration.

Aeraki translates MetaRouter to MetaProtocol route configuration, and then distributes them to the MetaProtocol Proxy through Aeraki's built-in RDS serve

The configuration of the sidecar proxy can be viewed with the following command.

``` bash
aerakictl_sidecar_config client meta-thrift |fx
```

The configuration of the MetaProtocol Proxy in the Outbound Listener of the Thrift service is shown below.

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

You can also view the RDS routing information that is currently in effect in the Proxy in the exported file, as follows.

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
    "name": "traffic-split",
    "match": {
     "metadata": [
      {
       "name": "method",
       "exact_match": "sayHello"
      }
     ]
    },
    "route": {
     "weighted_clusters": {
      "clusters": [
       {
        "name": "outbound|9090|v1|thrift-sample-server.meta-thrift.svc.cluster.local",
        "weight": 20
       },
       {
        "name": "outbound|9090|v2|thrift-sample-server.meta-thrift.svc.cluster.local",
        "weight": 80
       }
      ],
      "total_weight": 100
     }
    }
   }
  ]
 },
 "last_updated": "2022-01-11T10:26:37.357Z"
}
```







