---
title: 如何查看度量指标
description: 
weight: 70
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

在 istio-system 这个 NS 中已经安装了 Prometheus 和 Grafana，Prometheus 会从 Sidecar Proxy 中收集请求的指标度量数据。我们可以通过 Prometheus 查询这些度量指标，并通过 Grafana 的图表进行更友好的展示。

```bash
➜  ~ kubectl get deploy -n istio-system
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
aeraki                 1/1     1            1           46h
grafana                1/1     1            1           46h
istio-ingressgateway   1/1     1            1           46h
istiod                 1/1     1            1           46h
kiali                  1/1     1            1           46h
prometheus             1/1     1            1           46h
```

## 通过 Prometheus 查询请求指标

首先通过 ```kubectl port-forward``` 命令将将本地端口转发到 Prometheus 服务

```bash
kubectl port-forward service/prometheus 9090:9090 -n istio-system
```

在浏览器中打开 http://127.0.0.1:9090/ ，查询度量指标。MetaProtocol 的度量指标名有统一的前缀：“envoy_meta_protocol_$applicationProtocol”，例如 Dubbo 度量指标的名称前缀为 “envoy_meta_protocol_dubbo”，Thrift 度量指标的名称前缀为 “envoy_meta_protocol_thrift”。

查询 Dubbo 服务的 outbound request 指标：
![](../prometheus-request-time.png)

Dubbo 服务的所有指标：
![](../prometheus-request.png)
![](../prometheus-response.png)

## 通过 Grafana 图表来呈现度量指标

首先通过 ```kubectl port-forward``` 命令将将本地端口转发到 Grafana 服务

```bash
kubectl port-forward service/grafana 3000:3000 -n istio-system
```

将 Aeraki 提供的 [dashboard json 文件](https://github.com/aeraki-mesh/aeraki/blob/master/demo/grafana-dashboard.json)导入到 Grafana 中，如下图所示：

![](../grafana-import-dashboard.png)

打开 Aeraki Demo dashboard，可以看到 Dubbo 和 Thrift 服务的相关度量指标图表，包括 QPS，请求时延，请求成功率等等。

![](../grafana-metrics.png)




