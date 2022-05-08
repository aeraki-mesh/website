---
title: How to configure circuit breaking
description: 
weight: 40
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

## Simulate thrift service call failure

Create a  thrift-sample-server-fake deployment by using the command below，The deployment wil create an endpoint under thrift-sample-server service。As you can see from the yaml file below, there is no thrift sample server in the deployment, but rather an nginx container.

```bash
kubectl apply -f- <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thrift-sample-server-fake
  namespace: meta-thrift
  labels:
    app: thrift-sample-server
spec:
  selector:
    matchLabels:
      app: thrift-sample-server
  replicas: 1
  template:
    metadata:
      annotations:
        sidecar.istio.io/bootstrapOverride: aeraki-bootstrap-config
        sidecar.istio.io/proxyImage: aeraki/meta-protocol-proxy:1.0.1
        sidecar.istio.io/rewriteAppHTTPProbers: "false"
      labels:
        app: thrift-sample-server
    spec:
      containers:
        - name: thrift-sample-server
          image: nginx
          ports:
            - containerPort: 9090
EOF
```

Now there are three endpoint under the thrift-sample-server service, among them the thrift-sample-server-fake deployment have no thrift sample server, so the corresponding endpoint "172.19.0.102" can not handle client requests. From the client log, you can see that there is an error message once in every three requests:

```bash
➜  ~  aerakictl_app_log client meta-thrift --tail 0 -f
Connected to thrift-sample-server
Hello Aeraki, response from thrift-sample-server-v2-86db7567f-z8vwz/172.19.0.98
Hello Aeraki, response from thrift-sample-server-v1-c5cccb876-jmhhf/172.19.0.10
org.apache.thrift.TApplicationException: meta protocol upstream request: remote connection failure '172.19.0.102:9090'
	at org.apache.thrift.TServiceClient.receiveBase(TServiceClient.java:79)
	at org.aeraki.HelloService$Client.recv_sayHello(HelloService.java:61)
	at org.aeraki.HelloService$Client.sayHello(HelloService.java:48)
	at org.aeraki.HelloClient.main(HelloClient.java:44)
Connected to thrift-sample-server
Hello Aeraki, response from thrift-sample-server-v2-86db7567f-z8vwz/172.19.0.98
Hello Aeraki, response from thrift-sample-server-v1-c5cccb876-jmhhf/172.19.0.10
org.apache.thrift.TApplicationException: meta protocol upstream request: remote connection failure '172.19.0.102:9090'
	at org.apache.thrift.TServiceClient.receiveBase(TServiceClient.java:79)
	at org.aeraki.HelloService$Client.recv_sayHello(HelloService.java:61)
	at org.aeraki.HelloService$Client.sayHello(HelloService.java:48)
	at org.aeraki.HelloClient.main(HelloClient.java:44)
```

## Create circuit break rules

Create a Destination Rule by using the command below：When the request has 5 consecutive errors, the upstream host with the error is removed from the cluster's load balancing group.

```bash
kubectl apply -f- <<EOF
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: thrift-sample-server
  namespace: meta-thrift
spec:
  host: thrift-sample-server
  trafficPolicy:
    outlierDetection:
      baseEjectionTime: 15m
      consecutive5xxErrors: 5
      interval: 5m
EOF
```

At this point, if you look at the output of the client, you can see that the client no longer sends requests to the error endpoint "172.19.0.102" after the number of errors specified by the fusion rule.

```bash
org.apache.thrift.TApplicationException: meta protocol upstream request: remote connection failure '172.19.0.102:9090'
	at org.apache.thrift.TServiceClient.receiveBase(TServiceClient.java:79)
	at org.aeraki.HelloService$Client.recv_sayHello(HelloService.java:61)
	at org.aeraki.HelloService$Client.sayHello(HelloService.java:48)
	at org.aeraki.HelloClient.main(HelloClient.java:44)
Connected to thrift-sample-server
Hello Aeraki, response from thrift-sample-server-v2-86db7567f-z8vwz/172.19.0.98
Hello Aeraki, response from thrift-sample-server-v1-c5cccb876-jmhhf/172.19.0.10
org.apache.thrift.TApplicationException: meta protocol upstream request: remote connection failure '172.19.0.102:9090'
	at org.apache.thrift.TServiceClient.receiveBase(TServiceClient.java:79)
	at org.aeraki.HelloService$Client.recv_sayHello(HelloService.java:61)
	at org.aeraki.HelloService$Client.sayHello(HelloService.java:48)
	at org.aeraki.HelloClient.main(HelloClient.java:44)
Connected to thrift-sample-server
Hello Aeraki, response from thrift-sample-server-v2-86db7567f-z8vwz/172.19.0.98
Hello Aeraki, response from thrift-sample-server-v1-c5cccb876-jmhhf/172.19.0.10
org.apache.thrift.TApplicationException: meta protocol upstream request: remote connection failure '172.19.0.102:9090'
	at org.apache.thrift.TServiceClient.receiveBase(TServiceClient.java:79)
	at org.aeraki.HelloService$Client.recv_sayHello(HelloService.java:61)
	at org.aeraki.HelloService$Client.sayHello(HelloService.java:48)
	at org.aeraki.HelloClient.main(HelloClient.java:44)
Connected to thrift-sample-server
Hello Aeraki, response from thrift-sample-server-v2-86db7567f-z8vwz/172.19.0.98
Hello Aeraki, response from thrift-sample-server-v1-c5cccb876-jmhhf/172.19.0.10
org.apache.thrift.TApplicationException: meta protocol upstream request: remote connection failure '172.19.0.102:9090'
	at org.apache.thrift.TServiceClient.receiveBase(TServiceClient.java:79)
	at org.aeraki.HelloService$Client.recv_sayHello(HelloService.java:61)
	at org.aeraki.HelloService$Client.sayHello(HelloService.java:48)
	at org.aeraki.HelloClient.main(HelloClient.java:44)
Connected to thrift-sample-server
Hello Aeraki, response from thrift-sample-server-v2-86db7567f-z8vwz/172.19.0.98
Hello Aeraki, response from thrift-sample-server-v1-c5cccb876-jmhhf/172.19.0.10
org.apache.thrift.TApplicationException: meta protocol upstream request: remote connection failure '172.19.0.102:9090'
	at org.apache.thrift.TServiceClient.receiveBase(TServiceClient.java:79)
	at org.aeraki.HelloService$Client.recv_sayHello(HelloService.java:61)
	at org.aeraki.HelloService$Client.sayHello(HelloService.java:48)
	at org.aeraki.HelloClient.main(HelloClient.java:44)
Connected to thrift-sample-server
Hello Aeraki, response from thrift-sample-server-v1-c5cccb876-jmhhf/172.19.0.10
Hello Aeraki, response from thrift-sample-server-v2-86db7567f-z8vwz/172.19.0.98
Hello Aeraki, response from thrift-sample-server-v1-c5cccb876-jmhhf/172.19.0.10
Hello Aeraki, response from thrift-sample-server-v2-86db7567f-z8vwz/172.19.0.98
Hello Aeraki, response from thrift-sample-server-v1-c5cccb876-jmhhf/172.19.0.10
Hello Aeraki, response from thrift-sample-server-v2-86db7567f-z8vwz/172.19.0.98
Hello Aeraki, response from thrift-sample-server-v1-c5cccb876-jmhhf/172.19.0.10
Hello Aeraki, response from thrift-sample-server-v2-86db7567f-z8vwz/172.19.0.98
Hello Aeraki, response from thrift-sample-server-v1-c5cccb876-jmhhf/172.19.0.10
Hello Aeraki, response from thrift-sample-server-v2-86db7567f-z8vwz/172.19.0.98
Hello Aeraki, response from thrift-sample-server-v1-c5cccb876-jmhhf/172.19.0.10
```

## Understand what happened

By looking at envoy's stats output, you can see that thrift-sample-server is experiencing continuous request errors, and the host with the error has been removed from the cluster's load balancing group.

```bash
aerakictl_sidecar_stats client  meta-thrift|grep -i outlier
cluster.outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local.outlier_detection.ejections_active: 1
cluster.outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local.outlier_detection.ejections_consecutive_5xx: 1
cluster.outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local.outlier_detection.ejections_detected_consecutive_5xx: 1
cluster.outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local.outlier_detection.ejections_enforced_consecutive_5xx: 1
luster.outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local.outlier_detection.ejections_enforced_total: 1
cluster.outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local.outlier_detection.ejections_total: 1
```

