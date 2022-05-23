---
title: How to configure global rate limit
description: 
weight: 30
---

## Installing the demo program

If you haven't installed the demo program yet, Please refer to [Quick Start](../../quickstart/) to install Aeraki, Istio, and the demo.

After installation, you can see that the following two NSs have been added to the cluster, and the Dubbo and Thrift demo applications are installed in these two NSs. You can choose either of them to test.

```bash
➜  ~ kubectl get ns|grep meta
meta-dubbo        Active   16m
meta-thrift       Active   16m
```

## What is global rate limiting?

Unlike [local rate limiting](../local-rate-limit/), when using a global rate limiting, all service instances share a single rate limiting quota. A global rate limiting server is used to ensure that. When receiving a request, Sidecar Proxy on the server side will first send a rate limiting query request to rate limiting server, which will read the rules in its own configuration file, then determine whether the request triggers a rate limiting condition according to the rules, finally return result to Sidecar Proxy. Based on the result of this rate limiting, Sidecar Proxy decides whether to reject this request or to continue processing it.

![](../global-rate-limit.png)

## When to use global rate limiting?
In global rate limiting, the rate limiting decision is made at the global rate limiting server with a fixed quota, therefore, the number of client requests that are allowed to send to the server won't change even the number of service instances increases. Global rate limiting also introduces an additional hop at the request path, resulting in more latency. When there's a large number of client requests, the global rate limiting server may become a potential bottleneck. Besides, global rate limiting is more complicated to deploy and manage than local limiting.

If you just want to keep the number of requests sent to a single service instance within a reasonable limit, you should use local rate limiting. That's because local rate limiting is handled separately at each service instance's Sidecar Proxy, local rate limiting is more precise and reasonable for controlling inbound requests to service instances in this scenario. Used along with an appropriate HPA policy trigger by rate limiting metrics, we can keep the server cluster healthy by horizontally scaling the server cluster when all the existing service instances reach their limiting quota.

If you want to enforce a global access control policy to a particular resource, then global rate limiting should be used. A typical scenario is to set how often a user can access an API according to the user's service level agreement. Docker Hub, for example, has a rate limiting policy for how often a user can pull images from its registy. Paid users have a higher quota than free users.

## Deploy the rate limiting server

The rate limiting server has already been deployed in the example application, so you don't need to deploy the rate limiting server by youself.

The rate limiting rules for global rate limiting need to be set in the rate limiting server side. The following rate limiting rule has already been set in the demo, which allows 5 request per minute for the sayHello interface.

```yaml
domain: production
descriptors:
  - key: method
    value: "sayHello"
    rate_limit:
      unit: minute
      requests_per_unit: 5
```

The deployment scripts can be found here: https://github.com/aeraki-mesh/aeraki/tree/master/sample/metaprotocol-thrift/rate-limit-server

> Note: Since the rate limiting decesion is made on the limiting server, the rate limiting rules need to be set in the configuration file of the rate limiting server.

## Enable rate limiting for services

Once rate limiting is enabled for a service via MetaRouter, the service's Sidecar Proxy will initiate a rate limiting request to the rate limiting server upon receipt of the request and will decide whether to continue processing the request or reject it based on the response of the rate limiting server.

The following configuration sets rate limiting to the sayHello interface of the thrift-sample-server.meta-thrift.svc.cluster.local service. The value of the domain in the rate limiting request sent to the rate limiting server is production and the method attribute is added to the rate limiting request as descriptor. Note that the domain and descriptor set here must match the rate limiting configuration in the rate limiting server.

If the match condition is present, only requests that match the condition will be sent to the rate limiting server. If the match condition is not set, then all requests for that service will be sent to the rate limiting server. The match condition can be used to filter out requests locally that do not require rate limiting, improving the efficiency of request processing.

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

Using the aerakictl command to view the client's application log, you can see that the client can only successfully send out 5 requests per minute.

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

## Understand what happened

In the configuration sent to the Sidecar Proxy, Aeraki sets up the MetaProtocol Proxy in the FilterChain corresponding to the service in the VirtualInbound Listener.

Aeraki translates MetaRouter to global rate limiting filter route configuration, which is distributed to the MetaProtocol Proxy via Aeraki.

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







