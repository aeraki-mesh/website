---
title: Install
weight: 1150
description: Instructions for installing Aeraki.
minGoVers: 1.16
---

## Requirements

### MetaProtocol and Istio compatiblity

Before installing Aeraki, please check the supported Istio versions and the corresponding Proxy version:

| Aeraki       | MetaProtocol Proxy | Istio      |
|:------------:|:----------------:|:------------:|
| 1.0.x        | 1.0.x            | 1.10.x       |

### Modify Istio configuration

Please modify the istio ConfigMap to add the following content.

* Enable Istio DNS catpure
* Turn on metrics for Aeraki managed protocols

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

## Install Aeraki

```bash
git clone https://github.com/aeraki-framework/aeraki.git
cd aeraki
export AERAKI_VERSION=1.0.0
make install
```

## Install AerakiCtl(Optional)

You can choose to install aerakictl tool for debug purpose.

```bash
git clone https://github.com/aeraki-framework/aerakictl.git ~/aerakictl;source ~/aerakictl/aerakictl.sh
```
