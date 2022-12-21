---
title: 如何配置访问日志
description: 
weight: 81
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

## 查看访问日志

安装的 demo 程序已经打开了访问日志，我们可以通过 `kubectl logs` 命令查看 dubbo 或者 thrift demo 应用的日志。

```bash
➜  kubectl -n meta-dubbo logs dubbo-sample-consumer-797c4f7cc4-xtrx6 -c istio-proxy --tail 0 -f
[2022-12-21T03:13:21.056Z] dubbo 0  - "-" 114 301 522 "-" "2ad8281b-fba6-9226-8b89-f2fdc0fe62cd" outbound|20880||org.apache.dubbo.samples.basic.api.demoservice - 240.240.0.1:20880 10.244.0.21:44822 default

➜  kubectl -n meta-thrift logs thrift-sample-client-5d4976ff78-bm9gp -c istio-proxy --tail 0 -f
[2022-12-21T03:13:57.592Z] thrift 0  - "-" 0 0 3 "-" "17040b34-b91a-9660-bc3d-ac3cd31e6787" outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local - 10.107.185.19:9090 10.244.0.28:47406 default
```

## 设置访问日志格式

Aeraki Mesh 兼容 Istio 的 configMap，因此可以通过修改 istio configMap 来对访问日志格式进行设置，如下所示：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio
  namespace: istio-system
data:
  mesh: |-
    accessLogFile: /dev/stdout
    accessLogEncoding: TEXT
    accessLogFormat: "[%START_TIME%] %REQ(X-META-PROTOCOL-APPLICATION-PROTOCOL)%
     %RESPONSE_CODE% %RESPONSE_CODE_DETAILS% %CONNECTION_TERMINATION_DETAILS% \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\"
     %BYTES_RECEIVED% %BYTES_SENT% %DURATION% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(X-REQUEST-ID)%\" %UPSTREAM_CLUSTER%
     %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %ROUTE_NAME%\n"
```

在 istio configMap 中有三个和 access log 相关的配置：

* accessLogFile 访问日志的输出文件路径，缺省为 /dev/stdout
* accessLogEncoding 访问日志的编码，支持 TEXT/JSON，缺省为 TEXT
* accessLogFormat 访问日志的格式，缺省如下：

```
 "[%START_TIME%] %REQ(X-META-PROTOCOL-APPLICATION-PROTOCOL)% %RESPONSE_CODE% %RESPONSE_CODE_DETAILS% %CONNECTION_TERMINATION_DETAILS% \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(X-REQUEST-ID)%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %ROUTE_NAME%\n"
```

访问日志中部分关键字段的介绍参见下表：

公共字段：
|日志字段|字段描述|示例|
|-|-|-|
|[%START_TIME%]|日志输出时间|2022-12-21T03:13:21.056Z|
| %REQ(X-META-PROTOCOL-APPLICATION-PROTOCOL)%|应用协议|dubbo|
|%RESPONSE_CODE%|响应码|0(正常响应)1(错误响应)|
| %RESPONSE_CODE_DETAILS%|错误详细信息|upstream_reset|
|%BYTES_RECEIVED%|响应大小|113|
|%BYTES_SENT%|请求大小|301|
|%DURATION%|请求耗时|11|
|%UPSTREAM_CLUSTER%|上游cluster|outbound\|20880\|\|org.apache.dubbo.samples.basic.api.demoservice|
|%ROUTE_NAME%|路由名|default|

Dubbo协议：
|日志字段|字段描述|示例|
|-|-|-|
|[%INTERFACE%]|dubbo interface|org.apache.dubbo.samples.basic.api.DemoService|
|[%METHOD%]|dubbo method|sayHello|

Thrift协议：
|日志字段|字段描述|示例|
|-|-|-|
|[%METHOD%]|thrift method|sayHello|

Metadata 中的自定义参数：

除了缺省的字段之外，访问日志支持输出 Request 和 Response Metadata 中的任意内容。

例如下面的代码为 Dubbo 应用增加了一个自定义的 header `foo`，取值为 `bar`。
```java
RpcContext.getContext().setAttachment("foo", "bar");
```

我们可以通过在 accessLogFormat 添加字段 %REQ(FOO)%，以在访问日志中输出该参数。


对于该表格中未包含的字段，其含义请参见 [Envoy 的文档](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage)。

