---
title: 如何将 Dubbo 服务接入到 Aeraki Mesh?
description: 
weight: 1000
---

Aeraki Mesh 支持对 Dubbo 服务进行全面的七层服务治理，服务治理的粒度是 Dubbo interface。治理能力包括七层负载均衡，动态路由，熔断，本地/全局限流，请求层面的 Metrics 收集和呈现等，后面还会支持调用跟踪，流量镜像等能力。 本文将介绍如何将 Dubbo 服务接入到 Aeraki Mesh 中进行服务治理。

# Aeraki Mesh Dubbo 解决方案

Aeraki Mesh 的 Dubbo 服务治理解决方案如下图所示：
![](../dubbo-solution.png)

Aeraki Mesh 的 MetaProtocol Proxy 组件是一个数据面七层代理的通用框架，其中已经自带了 Dubbo codec 实现（下文将简称为 Meta-Dubbo)。

Aeraki 和 Istio 则一起作为控制面，通过 xDS 控制面标准接口下发 MetaProtocol Proxy 所需的配置信息。

Dubbo2Istio 可以连接 Dubbo 服务注册表，将其中注册的 Dubbo 服务转换为 ServiceEntry 资源，自动同步到 Istio 服务网格中。Dubbo2Istio 支持 ZooKeeper，Nacos，Etcd 三种类型的 Dubbo 服务注册表。

# 应用级服务治理和接口级服务治理

Dubbo 是一个来自 SOA 时代的 RPC 协议，因此并未严格按照微服务的原则将一个服务实现为一个进程。在 Dubbo 中有一个 Interface 的概念，一个进程中一般会有多个 Interface。我们在将 Dubbo 应用加入到服务网格进行服务治理时有两个选择：

* 以应用（进程）粒度进行服务治理：即以应用作为 Mesh 中的一个 Service
  * 优点：Mesh 中 Service 数量相对于接口级服务治理方案更少，控制面需要下发 的 xds 配置少， 控制面和 Sidecar 的资源占用相对小。
  * 缺点：无法按照 Interface 进行流量治理，包括灰度发布、限流、流量镜像等，只能按照应用级别进行流量治理。
* 以接口（Interface）粒度进行服务治理
  * 优点：可以 Interface 进行流量治理，包括灰度发布、限流、流量镜像等。
  * 缺点：无法按照 Interface 进行流量治理，包括灰度发布、限流、流量镜像等，只能按照应用级别进行流量治理。

可见两种治理方式各有优缺点，一般来说，当 Dubbo 应用的规模较大，Interface 较多的情况下，建议选择按照应用粒度进行服务治理，以减轻控制面负荷和 sidecar 的资源消耗。当 Dubbo 应用规模较小，Interface 数量较少的情况下，建议选择按照 Interface 粒度进行服务治理，以提供最精细的流量治理能力。

Aeraki Mesh 同时支持应用级和接口级服务治理，除了流量治理的粒度不同之外，这两种模式的路由，Metrics，Access log，Tracing 等能力是一致的。您可以根据自己的需要进行选择。

# 将 Dubbo 服务加入到服务网格中

## 对接 Dubbo 服务注册

### 采用 Dubbo2Istio 对接 Dubbo 服务注册表

> 备注：由于 Dubbo 注册表中是按照 Interface 粒度进行注册的，因此 Aeraki Mesh 提供的 Dubbo2Istio 目前只提供 Interface 级别的服务同步。如果您需要应用级别的服务同步，可以在 Dubb2Istio 的基础上进行修改，按照业务逻辑来将 Dubbo 注册表中的注册信息按照应用粒度提取出来，生成 Service Entry。也可以直接通过 Kubernetes Service 来按应用级别注册 Dubbo 服务（参考该 [Demo](https://github.com/aeraki-mesh/aeraki/tree/master/demo/metaprotocol-dubbo-app-level)）。

这种方法需要在在 K8s 集群中部署一个 Dubbo2Istio 组件。Dubbo2Istio 将会监控 Dubbo 注册表的变化，自动将 Dubbo 注册表中的 Dubbo 服务同步到 Istio 内部的服务注册表中。该方法需要维护一个额外的 Dubbo2Istio 组件，好处是保留了 Dubbo 原有的服务注册表，同时兼容服务网格和 Dubbo SDK 两种服务发现方式，可以将 Dubbo 程序进行有计划地迁移到服务网格。

Dubbo2Istio 会为每个 Dubbo 服务创建一个 ServiceEntry，并为该 ServiceEntry 自动分配一个 240.240.0.0/16 保留网段的虚拟 IP 地址。该 ServiceEntry 将在 Istio 中为该 Dubbo 服务定义相应的 dns 名称，名称为 Dubbo Infterace 的全小写形式。例如下面示例中的 Dubbo 服务的 Interface 为“org.apache.dubbo.samples.basic.api.DemoService”，则在 Istio 中对应的 dns 名为“org.apache.dubbo.samples.basic.api.demoservice”，Dubbo consumer 将使用该 dns 名来请求 Dubbo provider。

Dubbo 服务注册表中注册的实例的所有属性，如 relese， reversion 等会被作为 endpoint 的 label，这些 label 可以用于进行精细的七层流量控制，例如按照版本进行路由，灰度发布等。

Dubbo2Istio 自动创建的 ServiceEntry 如下所示：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  annotations:
    interface: org.apache.dubbo.samples.basic.api.DemoService
  labels:
    manager: aeraki
    registry: dubbo-zookeeper
  name: aeraki-org-apache-dubbo-samples-basic-api-demoservice
  namespace: dubbo
spec:
  hosts:
  - org.apache.dubbo.samples.basic.api.demoservice
  location: MESH_INTERNAL
  ports:
  - name: tcp-metaprotocol-dubbo
    number: 20880
    protocol: tcp
    targetPort: 20880
  resolution: STATIC
  addresses:
  - 240.240.0.18
  endpoints:
  - address: 172.17.0.97
    labels:
      anyhost: "true"
      application: dubbo-sample-provider
      default: "true"
      deprecated: "false"
      dubbo: 2.0.2
      dynamic: "true"
      generic: "false"
      interface: org.apache.dubbo.samples.basic.api.DemoService
      metadata-type: remote
      methods: testVoid-sayHello
      pid: "7"
      registryName: default
      release: 1.0-SNAPSHOT
      revision: 1.0-SNAPSHOT
      service_group: batchjob
      side: provider
      timestamp: "1663761491410"
      version: v2
    locality: bj/800005
    ports:
      tcp-metaprotocol-dubbo: 20880
    serviceAccount: default
```

如果采用 Dubbo2Istio，还需要将一些 Istio 所需的 endpoint 标签通过 Dubbo 自定义参数的形式注册到 Dubbo 注册表中。

首先需要在 k8s deployment 中为 Dubbo 应用容器设置相关的环境变量，包括：

* AERAKI_META_APP_NAMESPACE：应用所属的 namespace，用于指定 Service account 所属的 namespace
* AERAKI_META_APP_SERVICE_ACCOUNT：Dubbo 应用的 service account，用于 mtls 身份认证和权限控制
* AERAKI_META_WORKLOAD_SELECTOR：Dubbo 应用的 workload selector，用于 Aeraki Mesh 为指定的 pod 下发相关的数据面代理配置
* AERAKI_META_APP_VERSION：Dubbo 应用的版本，以用于设置流量规则，用于灰度发布等场景

```yaml
kind: Deployment
metadata:
  name: dubbo-sample-provider-v2
  labels:
    app: dubbo-sample-provider
spec:
  selector:
    matchLabels:
      app: dubbo-sample-provider
  replicas: 1
  template:
    metadata:
      labels:
        app: dubbo-sample-provider
        version: v2
    spec:
      containers:
        - name: dubbo-sample-provider
          image: aeraki/dubbo-sample-provider
          imagePullPolicy: Always
          env: # 设置 AERAKI_META 相关的环境变量
            - name: AERAKI_META_APP_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: AERAKI_META_APP_SERVICE_ACCOUNT
              valueFrom:
                fieldRef:
                  fieldPath: spec.serviceAccountName
            - name: AERAKI_META_WORKLOAD_SELECTOR
              value: "dubbo-sample-provider"     # The deployment must have a label: app:dubbo-sample-provider
            - name: AERAKI_META_APP_VERSION
              value: v2
```

这些环境参数需要通过 Dubbo 配置文件设置为自定义参数，以注册到 Dubbo 注册表，并同步到 Istio 中，如下面的配置文件所示：

```yaml
    <dubbo:application name="dubbo-sample-provider">
      <dubbo:parameter key="aeraki_meta_app_namespace" value="${AERAKI_META_APP_NAMESPACE}" />
      <dubbo:parameter key="aeraki_meta_app_service_account" value="${AERAKI_META_APP_SERVICE_ACCOUNT}" />
      <dubbo:parameter key="aeraki_meta_app_version" value="${AERAKI_META_APP_VERSION}" />
      <dubbo:parameter key="aeraki_meta_workload_selector" value="${AERAKI_META_WORKLOAD_SELECTOR}" />
    </dubbo:application>
```

### 采用 ServiceEntry 

> 备注：如果是进行应用级的服务治理，可以直接通过 Kubernetes Service 来注册 Dubbo 服务（参考该 [Demo](https://github.com/aeraki-mesh/aeraki/tree/master/demo/metaprotocol-dubbo-app-level)）。

如果 Dubbo 应用已经实现容器化并部署在了 K8s 中，并且不需要同时保留原有的 Dubbo SDK 服务发现方式，则可以直接编写 yaml 文件来将 Dubbo interface 定义为一个 Istio ServiceEntry。这种方式无需维护 Dubbo2Istio，也无需增加 Dubbo 自定义参数，运维和配置更为简单。Service Entry 的定义如下所示：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: dubbo-demoservice
  namespace: meta-dubbo
  annotations:
    interface: org.apache.dubbo.samples.basic.api.DemoService
spec:
  hosts:
    - org.apache.dubbo.samples.basic.api.demoservice
  ports:
    - number: 20880
      name: tcp-metaprotocol-dubbo
      protocol: TCP
  workloadSelector:
    labels:
      app: dubbo-sample-provider
  resolution: STATIC
```

## 短路 Dubbo SDK 服务发现

加入服务网格后，应用程序将使用服务网格中的服务发现能力，不再需要使用 Dubbo 注册中心进行服务发现。有两种方式可以短路 Dubbo SDK 的服务发现：

* 采用 resolve 文件，由于 resolve 文件的位置可以通过命令行传入，该修改方式不用修改源代码中的配置文件，是一种对已有应用程序完全无侵入的方式，我们将 url 设置为服务对接时约定的 Dubbo Interface 的全小写形式，如 "org.apache.dubbo.samples.basic.api.demoservice"），如下所示:  

    在启动命令中加入 reslove 文件参数：
    ```
    java -Ddubbo.resolve.file=./dubbo-resolve.properties
    ```
    在 resolve 文件中配置服务的url：
    ```
    org.apache.dubbo.samples.basic.api.DemoService=dubbo://org.apache.dubbo.samples.basic.api.demoservice:20880
    org.apache.dubbo.samples.basic.api.TestService=dubbo://org.apache.dubbo.samples.basic.api.testservice:20880
    org.apache.dubbo.samples.basic.api.ComplexService=dubbo://org.apache.dubbo.samples.basic.api.complexservice:20880
    org.apache.dubbo.samples.basic.api.SecondService=dubbo://org.apache.dubbo.samples.basic.api.secondservice:20880
    ```

* 也可以修改客户端的配置文件，在客户端配置文件中设置 Dubbo interface 对应的服务端 url：

    ```yaml
    <dubbo:reference id="demoService" check="true" interface="org.apache.dubbo.samples.basic.api.DemoService" url="dubbo://org.apache.dubbo.samples.basic.api.demoservice:20880" timeout="3000"/>
    ```

# Demo 应用

## 采用 ServiceEntry 的 Demo 应用（Interface 级流量治理）

直接参照 [快速开始](/zh/docs/v1.x/quickstart/) 即可安装 Aeraki，Istio 及 Dubbo 示例程序。

## 采用 Service 的 Demo 应用（应用级流量治理）

首先参照 [快速开始](/zh/docs/v1.x/quickstart/) 即可安装 Aeraki 和 Istio。

快速开始中安装的 Dubbo 示例程序是 Interface 级的流量治理，执行下面的命令可以安装应用级流量治理的示例程序。

```bash
./demo/metaprotocol-dubbo-app-level/install.sh
```

## 使用 Dubbo2Istio 对接 Dubbo 注册表 的 Demo 应用（Interface 级流量治理）

首先参照 [快速开始](/zh/docs/v1.x/quickstart/) 即可安装 Aeraki 和 Istio。

快速开始中安装的 Dubbo 示例程序没有采用 Dubbo 注册表，我们先将其卸载。

```bash
./demo/metaprotocol-dubbo/uninstall.sh
```

然后安装 Dubbo2Istio 中使用了 Dubbo 注册表的 Demo 程序：

```bash
git clone https://github.com/aeraki-mesh/dubbo2istio.git
cd dubbo2istio
kubectl create ns meta-dubbo
kubectl label namespace meta-dubbo istio-injection=enabled --overwrite
kubectl apply -f demo/k8s/zk -n meta-dubbo
kubectl apply -f demo/traffic-rules/destinationrule.yaml -n meta-dubbo
```
上面的脚本安装的是 ZooKeeper 注册表，你也可以选择安装 nacos 或者 etcd 注册表。Dubbo Demo 应用程序源码可以从 https://github.com/aeraki-mesh/dubbo-envoyfilter-example 下载。

稍等片刻后验证部署的 dubbo Demo 应用。

```bash
kubectl get pod -n meta-dubbo
NAME                                        READY   STATUS    RESTARTS   AGE
dubbo-sample-consumer-5cf9f6f878-qxwwp      2/2     Running   0          97s
dubbo-sample-provider-v1-6b7cc9b6f8-j9dvl   2/2     Running   0          97s
dubbo-sample-provider-v2-7546478cbf-l2l74   2/2     Running   0          97s
dubbo2istio-5c4cf7f847-d7kf2                1/1     Running   0          97s
zookeeper-77c844c5b9-7p47v                  1/1     Running   0          96s
```

可以看到 dubbo namespace中有下面的 pod：

* dubbo-sample-consumer: Dubbo 客户端应用
* dubbo-sample-provider-v1： Dubbo 服务器端应用（v1版本）
* dubbo-sample-provider-v2： Dubbo 服务器端应用（v2版本）
* zookeeper: Dubbo ZooKeeper 服务注册表
* dubbo2istio: 服务同步组件，负责将 Dubbo 服务同步到服务网格中


安装好 Dubbo Demo 程序后，你可以参考 Aeraki Mesh 的 [教程](https://aeraki.net/zh/docs/v1.x/tutorials/) 体验 Aeraki Mesh 为 Dubbo 应用提供的七层流量治理能力。






