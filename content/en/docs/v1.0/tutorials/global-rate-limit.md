---
title: How to configure global rate limit
description: 
weight: 30
---

## Install the sample program

If you haven't already installed the sample program，please Install  Aeraki，Istio and the sample program by referring to [Quick start](/docs/v1.0/quickstart/) .

After installation,you can catch sight of the following two NS to the cluster,which install the sample program for Dubbo and Thrift protocols ,based on MetaProtocol implementations, respectively.
You can test whichever program you want to choose.

```bash
➜  ~ kubectl get ns|grep meta
meta-dubbo        Active   16m
meta-thrift       Active   16m
```

## What is global_rate_limit

Different from [local-rate-limit](/zh/docs/v1.0/tutorials/local-rate-limit/)，When using global_rate_limit,all the examples of service share a limiting quota.Global_rate_limit uses a rate_limit  server to share the rate_limit quota among multiple service instances.When receiving a request, the Sidecar Proxy which services for server will first send send a query request for rate_limit to the rate_limit server，and the rate_limit server will read the rate_limit rules in its own configuration file，determine whether a rate_limit request triggers a rate_limit condition according to the rules,then return the rate_limit result to Sidecar Proxy.Finally Sidecar Proxy decide whether to prohibit this request or continue to process this request according to the rate_limit result.

![](../global-rate-limit.png)

## When to use global_rate_limit
Because of the feature of global_rate_limit,which is that the rate_limit judgment is made uniformly at the rate_limit server,it is not affected by the number of service instances。But global_rate_limit also introduces an extra hop of rate_limit servers.,which will bring additional network latency.In the case of a large number of customer requests，The rate_limit server itself is a potential bottleneck，its deployment and management are more complex than local rate_limit.

If the purpose of using rate_limit is to control the pressure of service instances within a reasonable range, it is recommended to use local_rate-limit.Because the local_rate_limit is processed separately at Sidecar proxy of each service instance, in this scenario, the local_rate_limit can control the incoming request of the service instance more accurately and reasonably.With the appropriate HPA strategy, the service instance can be horizontally expanded to ensure the stable operation of the service when the existing service instance is fully loaded.

If the purpose of rate_limit is to enforce a global access policy for access to a resource, global_rate-limit should be used. A typical example is setting how often users can access services by user level. For example, Docker Hub implements different rate_limit policies for the number of times users pull images according to the user level. Paid users enjoy higher image pull quotas than free users.

## Deploy a rate_limit server

In the sample program, the rate_limit server has been deployed, and the rate_limit rules have been configured through the configuration file, so there is no need to deploy it separately.

The rate_limit rules for global_rate_limit need to be set in the configuration file of the rate_limit server. The following rate_limit rules indicate that a rate_limit of 10 per minute is set for the sayHello interface.

```yaml
domain: production
descriptors:
  - key: method
    value: "sayHello"
    rate_limit:
      unit: minute
      requests_per_unit: 5
```

For deployment related scripts, see：https://github.com/aeraki-mesh/aeraki/tree/master/demo/metaprotocol-thrift/rate-limit-server

> Remarks: Because the judgment logic of global_rate_limit is executed in the rate_limit server, the rate_limit rules need to be set in the configuration file of the rate_limit server.

## Enable rate-limit for the service

Enable rate_limit  for the server through MetaRouter. After enabling, the sidecar proxy of the service will initiate a rate_limit request to rate_limit server after receiving the request, and decide whether to continue processing the request or terminate the request according to the return result of the request.

The following global_rate_limit configuration means limiting rate to the sayHello interface of the thrift-sample-server.meta-thrift.svc.cluster.local service. The value of domain in the rate_limit request sent to rate_limit server is production, and the method attribute will be added to the rate_limit request as a descriptor. Note that the domain and descriptor set here must match the rate_limit configuration in the rate_limit server.

A match condition can be set in the rate_limit settings in MetaRouter. If the match condition exists, it means that only requests that meet the conditions will be sent to the rate_limit server for rate_limit judgment. If the match condition is not set, it means that all requests for the service will be sent to rate_limit server for judgment. Reasonable use of match conditions can filter out requests that do not require rate_limit locally, improving request processing efficiency.
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

Using the aerakictl command to view the application log of the client, you can see that the client can only successfully execute 5 requests per minute:

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

## Understand the principle

In the configuration delivered to the Sidecar Proxy, Aeraki sets the MetaProtocol Proxy in the FilterChain corresponding to the service in the VirtualInbound Listener.

Aeraki will translate the rate_limit rules configured in the MetaRouter into the rate_limit configuration of the global_rate_limit filter, and send it to the MetaProtocol Proxy through Aeraki.

The configuration of the service's sidecar proxy can be viewed with the following command:

``` bash
aerakictl_sidecar_config server-v1 meta-thrift |fx
```

The MetaProtocol Proxy configuration in the Inbound Listener of the Thrift service is as follows:

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

