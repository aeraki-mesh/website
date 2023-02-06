---
title: How to query and visualize metrics
description: 
weight: 50
---

## Installing the demo program

If you haven't installed the demo program yet, Please refer to [Quick Start](../../quickstart/) to install Aeraki, Istio, and the demo.

After installation, you can see that the following two NSs have been added to the cluster, and the Dubbo and Thrift demo applications are installed in these two NSs. You can choose either of them to test.

```bash
➜  ~ kubectl get ns|grep meta
meta-dubbo        Active   16m
meta-thrift       Active   16m
```

Prometheus and Grafana have been installed in the istio-system NS, and Prometheus will collect requested metrics from the Sidecar Proxy. We can query these metrics through Prometheus and use Grafana dashboards to show these metrics.

```bash
?  ~ kubectl get deploy -n istio-system
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
aeraki                 1/1     1            1           46h
grafana                1/1     1            1           46h
istio-ingressgateway   1/1     1            1           46h
istiod                 1/1     1            1           46h
prometheus             1/1     1            1           46h
```

## Query request metrics via Prometheus

Open Prometheus UI via the ```istioctl dashboard prometheus``` command.

```bash
istioctl dashboard prometheus
```

View metrics in the Prometheus UI. Aeraki Mesh provides Istio compatible metrics for non-HTTP protocols，including istio_requests_total，istio_request_duration_milliseconds，istio_request_byte, and istio_response_byte。

Below diagrams show metrics for the Dubbo demo application:
istio_requests_total:
![](../prometheus-request-total.png)
istio_request_duration_milliseconds:
![](../prometheus-duration_milliseconds.png)
istio_request_byte:
![](../prometheus-request-byte.png)
istio_response_byte:
![](../prometheus-response_byte.png)

## Visualize metrics via Grafana dashboard

Open Grafana UI via the ```istioctl dashboard grafana``` command.

```bash
istioctl dashboard grafana
```

Service dashboard:
![](../istio-grafana-service.png)

Workload dashboard:
![](../istio-grafana-workload.png)

## Labels

Aeraki Mesh provides the same labels as Istio does for metrics. Please refer to [Istio Metrics](https://istio.io/latest/docs/reference/config/metrics/#labels) for the definition of each label.

Please note that the response code is different from HTTP protocol.

* OK 0
* Error 1






