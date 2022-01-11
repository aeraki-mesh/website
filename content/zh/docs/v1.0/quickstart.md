---
title: 快速开始
weight: 900
description: Get Aeraki up and running in less than 5 minutes!
---

请参照下面的步骤来安装、运行和测试 Aeraki：

 1. 从 github 下载 Aeraki。

    ```bash
    git clone https://github.com/aeraki-framework/aeraki.git
    ```

 2. 安装 Aeraki， Istio 和 demo 应用。

    ```bash
    make demo
    ```

    请注意: Aeraki 要求启用 [Istio DNS 代理](https://istio.io/latest/docs/ops/configuration/traffic-management/dns-proxy/). 如果你在一个正在运行的 Istio 部署上安装 Aeraki，请确保打开了 Istio DNS 代理功能；你可以直接使用 ```make demo``` 命令来在一个全新的 K8s 集群上从头安装 Aeraki 和 Istio， ```make demo``` 会对 Istio 进行正确的配置。
 3. 安装 aerakictl

    aerakictl 脚本工具封装了一些常用的 debug 命令，我们将在后续的教程中使用这些命令来查看应用程序和代理的信息。

    ```bash
    git clone https://github.com/aeraki-framework/aerakictl.git ~/aerakictl;source ~/aerakictl/aerakictl.sh
    ```

 4. 在浏览器中打开下面的网页，来查看 Aeraki 部署的应用程序和上报的请求指标数据。

    - Kaili http://{istio-ingressgateway_external_ip}:20001
    - Grafana http://{istio-ingressgateway_external_ip}:3000
    - Prometheus http://{istio-ingressgateway_external_ip}:9090
 

## 了解更多？

点击下面的链接，了解更多 Aeraki 的相关功能：

- [流量路由](/zh/docs/v1.0/tutorials/routing/) 
- [本地限流](/zh/docs/v1.0/local-rate-limit/)
- [全局限流](/zh/docs/v1.0/global-rate-limit//)
- [开发一个自定义协议](/zh/docs/v1.0/tutorials/implement-a-custom-protocol/)
