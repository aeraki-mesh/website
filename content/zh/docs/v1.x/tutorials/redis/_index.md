---
title: Redis 流量管理
weight: 2000
---

Redis 是一种高性能的键值数据库，通常被用作缓存、会话存储和消息代理等用途。Aeraki Mesh 提供了对 Redis 的流量管理能力，可以实现客户端无感知的 Redis Cluster 数据分片，按 key 将客户端请求路由到不同的 Redis 服务，读写分离，流量镜像，故障注入等高级流量管理功能。
