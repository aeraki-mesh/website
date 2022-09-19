---
title: Aeraki Mesh 1.1.0 发布公告
subtitle: 
description:  Aeraki Mesh 1.1.0 已经支持 Istio 1.12.X 系列版本
date: 2022-05-23
author: Huabing Zhao
keywords: [aeraki, MetaProtocol]
---

我们很高兴地宣布 Aeraki Mesh 1.1.0 的发布！这是 2022 年发布的第一个主要版本。

以下是该版本的一些亮点：

## 支持 Istio 1.12.x 版本

Aeraki Mesh 1.1.0 开始支持 Istio 1.12.x 系列版本。使用 Aeraki Mesh 1.1.0 后，用户可以升级到 Istio 1.12.x 版本，以获得 Istio 的新特性和故障修复。

Aeraki Mesh 1.0.x 系列将进入维护阶段，只修复安全故障和重大故障。新需求将基于 1.1.0 进行开发。

## 新增协议支持：bRPC

Aeraki Mesh 在 1.1.0 版本中开始支持 [bRPC](https://brpc.apache.org/) 协议。bRPC 是百度开源的工业级RPC框架, 有1,000,000+个实例(不包含client)和上千种多种服务。"brpc"的含义是"better RPC"。可以通过 `make demo-brpc` 安装 brpc 的示例程序进行尝试。

目前 Aeraki Mesh 已经支持了超过七种非 HTTP 协议，包括 Dubbo、Thrift、bRPC、Redis 等开源协议，以及腾讯音乐、腾讯融媒体、腾讯游戏人生、灵雀云等内部的私有协议。

## 其他一些特性

* MetaProtocol Proxy 支持 Arm 架构
* 提供用于编译 MetaProtocol 的 Docker 镜像
* MetaProtocol 支持流式调用
* MetaProtocol 支持采用 MetaData [设置负载均衡 Hash 策略](https://www.aeraki.net/zh/docs/v1.x/tutorials/consistent-hash-lb/#%E8%AE%BE%E7%BD%AE%E9%87%87%E7%94%A8-consistent-hash)
* MetaProtocol 支持在 Response 中回传服务器真实 IP 

## 感谢

本版本的发布来自于 Aeraki Mesh 社区同学的共同努力，特别是百度、灵雀云和腾讯云的同学为本版本做了大量的工作。非常感谢以下同学对 1.1.0 版本的贡献：

[![](https://github.com/smwyzi.png?size=40)](https://github.com/smwyzi) 
[![](https://github.com/Xunzhuo.png?size=40)](https://github.com/Xunzhuo) 
[![](https://github.com/huanghuangzym.png?size=40)](https://github.com/huanghuangzym) 
[![](https://github.com/nevermosby.png?size=40)](https://github.com/nevermosby) 
[![](https://github.com/weixiao619.png?size=40)](https://github.com/weixiao619) 
[![](https://github.com/Sad-polar-bear.png?size=40)](https://github.com/Sad-polar-bear) 
[![](https://github.com/wen73.png?size=40)](https://github.com/wen73)
[![](https://github.com/zhaohuabing.png?size=40)](https://github.com/zhaohuabing)



