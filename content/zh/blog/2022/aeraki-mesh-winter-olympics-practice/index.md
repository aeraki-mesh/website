---
title: 腾讯云原生分享：Areaki Mesh 在 2022 冬奥会视频直播应用中的服务网格实践
subtitle: 
description:  
date: 2022-03-30
author: Huabing Zhao
keywords: [aeraki]
---

![鸟巢](https://www.zhaohuabing.com/img/2022-03-30-aeraki-mesh-winter-olympics-practice/bird-nest.jpeg)

## 主题简介

服务网格已经成为微服务的基础设施，但目前主流的服务网格产品只能处理 HTTP 协议，不支持其他七层协议，是服务网格落地的主要困难之一。本次直播分享主要介绍腾讯云服务网格团队开源的 Aeraki Mesh 项目如何通过扩展 Istio 来支持 Thrift，Dubbo 等开源协议以及私有协议，并分享腾讯融媒体采用 Aeraki Mesh 支撑 2022 冬奥会视频直播的实践经验

## 听众收益

1. Aeraki Mesh 第二版的架构变化，功能特性，以及社区的项目规划。
2. Aeraki Mesh 推出的 MetaProtocol 通用七层代理框架实现原理。
3. 如何基于 Aeraki Mesh 接入一个私有协议。
2. Aeraki Mesh 接入视频类 videopacket 私有协议的产品落地案例。
3. 基于限流场景的业务侧优雅降级联动以及与集群弹性扩容联动。
4. 基于流量镜像、服务路由功能的使用场景。

## 活动链接
* [腾讯云活动链接](https://mp.weixin.qq.com/s/zp9q99mGyH2VD9Dij2owWg)

## 演讲稿

[pdf 下载](/img/2022-03-30-aeraki-mesh-winter-olympics-practice/slides.pdf)
<iframe src="https://docs.google.com/presentation/d/e/2PACX-1vS_SlDxcHWPLZxjx69ZGIBMos9FmDYpu2yW-cH4Ljoo9X5_Ucre2p6MlE6L0P4HVw/embed?start=false&loop=false&delayms=3000" frameborder="0" width="960" height="570" allowfullscreen="true" mozallowfullscreen="true" webkitallowfullscreen="true"></iframe>

## 视频回放
B站
{{< bilibili  BV1HP4y1M7xw >}}

YouTube
{{< youtube uXxatQTKzW8 >}} 
