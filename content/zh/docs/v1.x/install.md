---
title: 安装
weight: 1000
description: Instructions for installing Aeraki.
minGoVers: 1.16
---

## 前置条件

### Aeraki 和 MetaProtocol 以及 Istio 的版本兼容性

在安装 Aeraki 之前，请先根据下面的版本兼容矩阵检查 Aeraki 对应 Istio 和 MetaProtocol Proxy 版本：

| Aeraki       | MetaProtocol Proxy | Istio      |
|:------------:|:----------------:|:------------:|
| 1.3.x        |       1.3.x      | 1.16.x       |
| 1.2.x        |       1.2.x      | 1.14.x       |
| 1.1.x        |       1.1.x      | 1.12.x       |
| 1.0.x        |       1.0.x      | 1.10.x       |

### 检查下面的 Istio 选项

请修改 istio ConfigMap，加入下面的内容：

* 启用 Istio DNS 代理
* 打开 Aeraki 管理的协议的 Metrics 收集

```Bash
kubectl edit cm istio -n istio-system
```

```yaml
apiVersion: v1
data:
  mesh: |-
    defaultConfig:
      proxyMetadata:
        ISTIO_META_DNS_CAPTURE: "true"
      proxyStatsMatcher:
        inclusionPrefixes:
        - thrift
        - dubbo
        - kafka
        - meta_protocol
        inclusionRegexps:
        - .*dubbo.*
        - .*thrift.*
        - .*kafka.*
        - .*zookeeper.*
        - .*meta_protocol.*
```

## 安装 Aeraki

```bash
git clone https://github.com/aeraki-mesh/aeraki.git
cd aeraki
export AERAKI_TAG=1.3.0
make install
```

## 安装 AerakiCtl（可选）

You can choose to optionally install aerakictl tool for debug purpose.

```bash
git clone https://github.com/aeraki-mesh/aerakictl.git ~/aerakictl;source ~/aerakictl/aerakictl.sh
```
## 高可用和水平扩展

Aeraki 内主要包括两类组件：
* 控制器：用于 Watch Istio 资源并维护 Aeraki 内部系统状态，此类组件为有状态组件，Aeraki 通过选主实现高可用
* MetaProtocol RDS 服务器：根据 MetaRouter CRD 资源生成 MetaProtocol 动态路由，并通过 MetaRDS 下发给数据面的 Proxy，Aeraki 支持多实例水平扩展，以对 RDS 服务器的压力进行负载均衡。

对于生产环境，请根据集群规模和数据面边车数量调整对应的 Aeraki 实例，可以通过 K8s HPA 进行动态扩展。

## 在 TCM 中使用 Aeraki

腾讯云服务网格 [TCM](https://cloud.tencent.com/product/tcm) 支持集成 Aeraki。如果希望在 TCM 使用 Aeraki 的协议扩展能力，请联系[腾讯云售前架构师](https://cloud.tencent.com/act/event/connect-service?from=intro_tcm#/)进行咨询。
