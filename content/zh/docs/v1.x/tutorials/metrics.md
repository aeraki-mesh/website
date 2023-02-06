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
prometheus             1/1     1            1           46h
```

## 通过 Prometheus 查询请求指标

采用 ```istioctl dashboard prometheus``` 命令打开 Prometheus 界面。

```bash
istioctl dashboard prometheus
```

在浏览器中查询度量指标。Aeraki Mesh 为非 HTTP 协议提供了和 Istio 兼容的 metrics，包括 istio_requests_total，istio_request_duration_milliseconds，istio_request_byte 和 istio_response_byte。

查询 Dubbo 服务的 outbound request 指标：
istio_requests_total 指标：
![](../prometheus-request-total.png)
istio_request_duration_milliseconds 指标：
![](../prometheus-duration_milliseconds.png)
istio_request_byte 指标：
![](../prometheus-request-byte.png)
istio_response_byte 指标：
![](../prometheus-response_byte.png)

## 通过 Grafana 图表来呈现度量指标

采用 ```istioctl dashboard grafana``` 命令打开 Grafana 界面。

```bash
istioctl dashboard grafana
```
Service 视角的 Grafana 监控面板：
![](../istio-grafana-service.png)

Workload 视角的 Grafana 监控面板：
![](../istio-grafana-workload.png)

## Labels

Aeraki Mesh 为非 HTTP 协议生成的 Metrics 中的 Label 和 Istio 保持一致。要了解各个 Label 的具体含义，请参考 [Istio 的相关文档](https://istio.io/latest/docs/reference/config/metrics/#labels)。

注意：其中 Response code 含义和 HTTP 协议有所不同。

MetaPrtocol Proxy 中 Response Code 的含义如下：
* OK 0
* Error 1



