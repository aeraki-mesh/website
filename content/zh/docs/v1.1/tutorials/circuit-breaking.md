---
title: 如何设置熔断规则
description: 
weight: 40
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

## 模拟 thrift 服务调用失败

通过下面的命令创建一个 thrift-sample-server-fake deployment，该 deployment 将在 thrift-sample-server service 中增加一个 endpoint。从下面的 yaml 可以看到，该 deployment 中并没有 thrift sample server，而是一个 nginx 容器。

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

现在 thrift-sample-server service 中有三个 endpoint，其中 thrift-sample-server-fake 这个 deployment 中没有 thrift sample server，因此其对应的 endpoint "172.19.0.102" 并不能处理客户端的请求。从客户端的日志中可以看到每三次请求中就有一次错误信息：

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

## 创建熔断规则

通过下面的命令创建一个 Destination Rule：当请求连续出现 5 次错误后，就将出错的 upstream host 从 cluster 的负载均衡分组中移除。

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

此时查看客户端的输出，可以看到客户端在熔断规则指定的错误次数后，不再将请求发送到出错的 endpoint "172.19.0.102"。

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

## 理解原理

通过查看 envoy 的 stats 输出，可以看到 thrift-sample-server 发生了连续的请求错误，该出错的 host 被移除了该 cluster 的负载均衡分组。

```bash
aerakictl_sidecar_stats client  meta-thrift|grep -i outlier
cluster.outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local.outlier_detection.ejections_active: 1
cluster.outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local.outlier_detection.ejections_consecutive_5xx: 1
cluster.outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local.outlier_detection.ejections_detected_consecutive_5xx: 1
cluster.outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local.outlier_detection.ejections_enforced_consecutive_5xx: 1
luster.outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local.outlier_detection.ejections_enforced_total: 1
cluster.outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local.outlier_detection.ejections_total: 1
```







