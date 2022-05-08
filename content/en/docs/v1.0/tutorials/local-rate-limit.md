---
title: How to configure local rate limit
description: 
weight: 20
---

## Installing the sample program

If you haven't installed the sample program,Please refer to [Quick Start](./../quickstart.md) to install Aeraki, Istio, and sample programs.

After the installation, you can see that the following two NSs have been added to the cluster, and the sample applications for Dubbo and Thrift protocols based on the MetaProtocol implementation are installed in these two NSs.
You can choose any of the programs to test.

```bash
➜  ~ kubectl get ns|grep meta
meta-dubbo        Active   16m
meta-thrift       Active   16m
```

Aeraki's flow restriction rules are designed to be intuitive and flexible, supporting both flow restriction for all inbound requests to a service and fine-grained flow restriction control for requests to a server based on different conditions.

## Restrict the flow of all inbound requests to the service

The following rule limits all inbound requests to the thrift-sample-server.meta-thrift.svc.cluster.local service to 2 requests / minute.

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

> Note: Because local flow limiting is handled separately on each service instance, when the service has multiple instances, the actual flow limiting effect is the number of flow limiting times the number of instances.


Using the aerakictl command to view the client's application log, you can see that the client can only successfully execute 4 requests per minute (with two service instances, each limited to 2 requests per minute).

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

## Restrict the flow of inbound requests to services by condition

Aeraki supports multiple flow restriction rules for services based on conditions to meet fine-grained flow restriction requirements. For example, grouping requests by user or pair of interfaces and setting different flow restriction rules for each group. 

The matching conditions for grouped flow restriction are the same as those for routing, and any attributes that can be extracted from the request packet can be used for the matching conditions of the flow restriction rule.

For example, the following rules set different flow restrictions for the sayHello and ping interfaces.

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

## Set rules for restriction of flow by service and by condition

It is possible to set both service granular flow restriction rules and conditional flow restriction rules, which is suitable for cases where an overall flow restriction rule needs to be set for all requests of a service, while exceptions need to be set for a certain group or groups of requests.

For example, the following flow restriction rule sets an overall flow restriction rule for the service of 1000 messages/minute, and a separate flow restriction condition of 100 messages/minute for the ping interface.

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

## Understand the principles

In the configuration issued to the Sidecar Proxy, Aeraki sets the MetaProtocol Proxy in the FilterChain corresponding to the service in the VirtualInbound Listener.

Aeraki translates the flow-limiting rules configured in the MetaRouter into a flow-limiting configuration for the local rate limit filter, which is distributed to the MetaProtocol Proxy through Aeraki.

The configuration of the service's sidecar proxy can be viewed with the following command.

``` bash
aerakictl_sidecar_config server-v1 meta-thrift |fx
```

The configuration of the MetaProtocol Proxy in the Inbound Listener of the Thrift service is shown below.

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







