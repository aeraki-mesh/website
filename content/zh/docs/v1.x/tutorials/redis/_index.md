---
title: Redis 流量管理
weight: 2000
---

Redis 是一种高性能的键值数据库，通常被用作缓存、会话存储和消息代理等用途。Redis 可以部署为单机模式或者 Cluster 模式，但不同模式的客户端访问方式不同，这增加了应用使用 Redis 的开发成本。通过采用 Aeraki Mesh 的 Redis 流量管理功能，我们可以在不修改客户端代码的前提下切换后端的 Redis 部署模式，实现客户端无感知的 Redis Cluster 数据分片，并提供读写分离、流量镜像等高级流量管理功能。

比如您在测试集群使用一个小型的单实例的 Redis ，而在生产环境使用一个复杂而稳定的多副本实例的集群模式的Redis，通过 Aeraki Mesh，您将无需修改您应用的代码/配置，从而尽可能的保证了开发，预发布，线上环境相同（[Dev/prod parity](https://12factor.net/dev-prod-parity)）。
