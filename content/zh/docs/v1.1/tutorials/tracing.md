---
title: 如何查看调用跟踪
description: 
weight: 80
---

## 安装示例程序

如果你还没有安装示例程序，请参照 [快速开始](/zh/docs/v1.0/quickstart/) 安装 Aeraki，Istio 及示例程序。

执行完成后，在 meta-dubbo 这个 NS 中安装了基于 MetaProtocol 实现的 Dubbo 协议的示例程序。
我们将采用该 Dubbo 示例程序来进行测试。Dubbo Demo 程序的调用关系为：dubbo-sample-consumer --> dubbo-sample-provider --> dubbo-sample-second-provider 。

```bash
➜  ~ kubectl -n meta-dubbo get pod
NAME                                            READY   STATUS    RESTARTS   AGE
dubbo-sample-consumer-5c8f9d457-bfnxc           2/2     Running   0          45s
dubbo-sample-provider-v1-69b986cb77-bm4kh       2/2     Running   0          45s
dubbo-sample-provider-v2-7479958d88-qktm4       2/2     Running   0          45s
dubbo-sample-second-provider-77cdfb955f-56chj   2/2     Running   0          45s
```

在 istio-system 这个 NS 中已经安装了 Jaeger，并且在安装 Demo 时设置了 Mesh 的采样率为 100%，因此 Demo 应用的所有请求都会生成 tracing 记录，并上报到 Jaeger。

备注：由于生成 tracing 数据对程序性能有一定影响，在生产环境中一般不会把 Mesh 的采样率设置为 100%。Aeraki 和 Istio 采用相同的 Tracing 配置，在未显示设置采样率时，缺省采样率为 1%。

## 通过 Jaeger 查看 Tracing

通过 ```istioctl dashboard jaeger``` 命令打开 Jaeger 的界面。

```bash
istioctl dashboard jaeger
```

查询 Dubbo 服务的 Trace：
![](../traces.png)

查看一条 Trace 经过的所有服务的调用关系：
![](../trace-timeline.png)

查看 Trace span 的 tag：
![](../trace-span-tag.png)

## 传递调用跟踪相关的 header

启用 tracing 后，MetaProtocol Proxy 会在请求的第一跳生成第一个 tracing span，并将 tracing 的上下文，包括 tracing id，当前的 span id 等加入到请求 header 中。但由于 MetaProtocol Proxy 并不能感知其入向请求和出向请求之间的业务关联关系，需要应用代码将入向请求中调用跟踪相关的 header 设置到对应的出向请求中。
应用代码需要传递下面这些 tracing 相关的 header：
* x-request-id
* x-b3-traceid
* x-b3-spanid
* x-b3-parentspanid
* x-b3-sampled
* x-b3-flags
* b3
* x-ot-span-context

备注：Dubbo 应用无需修改代码即可实现调用跟踪，因为 Dubbo 缺省会将自定义 header（attachment）通过 ThreadLocal 机制传递给下一个请求。
