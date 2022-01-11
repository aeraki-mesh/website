---
title: Quickstart
weight: 900
description: Get Aeraki up and running in less than 5 minutes!
---

Follow these instructions to install, run, and test Aeraki:

 1. Download Aeraki from the github.

    ```console
    git clone https://github.com/aeraki-framework/aeraki.git
    ```

 2. Install Istio, Aeraki and demo applications.

    ```console
    make demo
    ```

    Note: Aeraki requires to enable [Istio DNS proxying](https://istio.io/latest/docs/ops/configuration/traffic-management/dns-proxy/). Please turn on DNS proxying if you are installing Aeraki with an existing Istio deployment, or you can use ```make demo``` command to install Aeraki and Istio from scratch, ```make demo``` will take care of the Istio configuration.

 3. Open the following URLs in your browser to play with Aeraki and view service metrics

     - Kaili http://{istio-ingressgateway_external_ip}:20001
     - Grafana http://{istio-ingressgateway_external_ip}:3000
     - Prometheus http://{istio-ingressgateway_external_ip}:9090


## What's next?

Learn about more ways to configure and use Aeraki from the following pages:

- [Traffic routing]() 
- [Local rate limit]()
- [Global rate limit]()
- [Implement a new protocol]()
