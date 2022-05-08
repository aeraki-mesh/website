---
title: How to match and visualize metrics
description: 
weight: 50
---

## Installing the demo program

If you haven't already installed the demo program, please refer to [quick start](/docs/v1.0/quickstart/) to install Aeraki, Istio and the demo program.

After the installation is complete, you can see that the following two NSs have been added to the cluster. Demo programs of the Dubbo and Thrift protocols based on MetaProtocol are installed in these two NSs respectively.
You can choose either of them to test.

```bash
?  ~ kubectl get ns|grep meta
meta-dubbo        Active   16m
meta-thrift       Active   16m
```

Prometheus and Grafana have been installed in the istio-system NS, and Prometheus will collect requested metrics from the Sidecar Proxy. We can match these metrics through Prometheus and display them more friendly through Grafana's graphs.

```bash
?  ~ kubectl get deploy -n istio-system
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
aeraki                 1/1     1            1           46h
grafana                1/1     1            1           46h
istio-ingressgateway   1/1     1            1           46h
istiod                 1/1     1            1           46h
kiali                  1/1     1            1           46h
prometheus             1/1     1            1           46h
```

## Match request metrics via Prometheus

First forward the local port to the Prometheus service via the kubectl port-forward command

```bash
kubectl port-forward service/prometheus 9090:9090 -n istio-system
```

Open http://127.0.0.1:9090/ in browser, match metrics. MetaProtocol's metrics names have a uniform prefix: "envoy_meta_protocol_$applicationProtocol", For example, Dubbo metrics have names prefixed with "envoy_meta_protocol_dubbo", Thrift metrics have names prefixed with "envoy_meta_protocol_thrift".

Match the outbound request metrics of the Dubbo service:
![](../prometheus-request-time.png)

All metrics for Dubbo services:
![](../prometheus-request.png)
![](../prometheus-response.png)

## Rendering metrics via Grafana charts

First forward the local port to the Grafana service via the ```kubectl port-forward``` command.

```bash
kubectl port-forward service/grafana 3000:3000 -n istio-system
```

Import the [dashboard json file](https://github.com/aeraki-mesh/aeraki/blob/master/demo/grafana-dashboard.json) provided by Aeraki into Grafana as shown below:

![](../grafana-import-dashboard.png)

Open the Aeraki Demo dashboard, you can see the relevant metrics charts of Dubbo and Thrift services, including QPS, request delay, request success rate, etc.

![](../grafana-metrics.png)





