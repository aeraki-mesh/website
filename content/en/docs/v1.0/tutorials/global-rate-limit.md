---
title: How to configure global rate limit
description: 
weight: 30
---

## Installing the demo program

If you haven't installed the demo program,Please refer to  [Quick Start](/docs/v1.0/quickstart/) to install Aeraki, Istio and demo programs.

Once the installation is complete, you can see the following two NSs have been added to the cluster,The two NSs have demo applications installed for Dubbo and Thrift protocols based on the MetaProtocol implementation respectively.
You can choose either of them to test.

```bash
➜  ~ kubectl get ns|grep meta
meta-dubbo        Active   16m
meta-thrift       Active   16m
```

## What is global rate limit?

Unlike [local rate limit](/zh/docs/v1.0/tutorials/local-rate-limit/), when using a global rate limit, all service instances share a single limit rate quota. The global rate limit is used to share the limit rate quota between multiple service instances via a rate limiting server.When receiving a request, Sidecar Proxy on the server side will first send a rate limiting query request to rate limiting server, which will read the rules in its own configuration file, then determine whether the request triggers a rate limiting condition according to the rules, finally return result to Sidecar Proxy. Based on the result of this rate limiting, Sidecar Proxy decides whether to disable this request or to continue processing it.

![](../global-rate-limit.png)

## When to use global rate limit?
The feature of global rate limit is that the limiting judgement is made uniformly at the rate limiting server and is therefore not affected by the number of service instances. However, global rate limit also introduces an additional hop to the rate limiting server, which can introduce additional network latency. In the case of a large number of client requests, the rate limiting server itself is a potential bottleneck point and is more complex to deploy and manage than local limit.

If the purpose of using rate limit is to keep the pressure on the service instance within reasonable limits, it is recommended that local rate limit be used. Because local rate limit is handled separately at each service instance's Sidecar Proxy, local rate limit is more precise and reasonable for controlling inbound requests to service instances in this scenario. With an appropriate HPA policy, it is also possible to horizontally scale the service instances when the existing service instances are fully loaded, ensuring stable service operation.

If the purpose of rate limiting is to enforce a global access policy for access to a particular resource, then full rate limiting should be used. A typical example is to set how often a user can access a service according to user level. Docker Hub, for example, has a different rate limit policy for the number of times a user can pull an image according to user level, with paid users having a higher limit on the number of image pulls than free users.

## Deploy the rate limiting server

The rate limiting server is already deployed in the example application and the rate limiting rules are configured via the configuration file, so there is no need to deploy it separately.

The rate limiting rules for global rate limit need to be set in the configuration file of the rate limiting server. The following rate limiting rule indicates a rate limiting of 10 streams per minute for the sayHello interface.

```yaml
domain: production
descriptors:
  - key: method
    value: "sayHello"
    rate_limit:
      unit: minute
      requests_per_unit: 5
```

For deployment scripts see:https://github.com/aeraki-mesh/aeraki/tree/master/demo/metaprotocol-thrift/rate-limit-server

> Note: Since the logic of global rate limit is executed in the rate limiting server, the rate limiting rules need to be set in the configuration file of the rate limiting server.

## Enable rate limit for services

Once rate limit is enabled for the server via MetaRouter, the service's Sidecar Proxy will initiate a rate limit request to the rate limit server upon receipt of the request and will decide whether to continue processing the request or terminate it based on the return of the request.

The following global rate limit configuration represents a rate limit to the sayHello interface of the thrift-demo-server.meta-thrift.svc.cluster.local service. The value of the domain in the rate limit request sent to the rate limit server is production and the method attribute is added to the rate limit request as descriptor. Note that the domain and descriptor set here must match the rate limit configuration in the rate limit server.

The match condition can be set in the rate limit settings in MetaRouter.If the match condition is present, only requests that match the condition will be sent to the rate limit server for judgement. If the match condition is not set, then all requests for that service will be sent to the rate limit server for determination. The match condition can be used to filter out requests locally that do not require rate limiting, improving the efficiency of request processing.
```yaml
kubectl apply -f- <<EOF
apiVersion: metaprotocol.aeraki.io/v1alpha1
kind: MetaRouter
metadata:
  name: test-metaprotocol-thrift-route
  namespace: meta-thrift
spec:
  hosts:
  - thrift-demo-server.meta-thrift.svc.cluster.local
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

Using the aerakictl command to view the client's application log, you can see that the client can only successfully execute 5 requests per minute.

```bash
➜  ~ aerakictl_app_log client meta-thrift -f --tail 10
Hello Aeraki, response from thrift-demo-server-v1-5c8476684-842l6/172.17.0.40
Hello Aeraki, response from thrift-demo-server-v2-6d5bcc885-hpx7n/172.17.0.41
Hello Aeraki, response from thrift-demo-server-v1-5c8476684-842l6/172.17.0.40
Hello Aeraki, response from thrift-demo-server-v2-6d5bcc885-hpx7n/172.17.0.41
Hello Aeraki, response from thrift-demo-server-v1-5c8476684-842l6/172.17.0.40
org.apache.thrift.TApplicationException: meta protocol local rate limit: request '6' has been rate limited
        at org.apache.thrift.TServiceClient.receiveBase(TServiceClient.java:79)
        at org.aeraki.HelloService$Client.recv_sayHello(HelloService.java:61)
        at org.aeraki.HelloService$Client.sayHello(HelloService.java:48)
        at org.aeraki.HelloClient.main(HelloClient.java:44)
Connected to thrift-demo-server
org.apache.thrift.TApplicationException: meta protocol local rate limit: request '7' has been rate limited
...
```

## Understanding the principles of rate limiting

In the configuration issued to the Sidecar Proxy, Aeraki sets up the MetaProtocol Proxy in the FilterChain corresponding to the service in the VirtualInbound Listener.

Aeraki translates MetaRouter to global rate limit filter route configuration, which is distributed to the MetaProtocol Proxy via Aeraki.

The configuration of the sidecar proxy for the service can be viewed with the following command：

``` bash
aerakictl_sidecar_config server-v1 meta-thrift |fx
```

The MetaProtocol Proxy in the Inbound Listener of the Thrift service is configured as follows:

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







