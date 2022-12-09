---
title: "独立健康检查连接"
weight: 3
---

在 client-go 中使用相同的配置连接一个集群时，会使用同一条 TCP 连接
https://github.com/kubernetes/kubernetes/blob/3f823c0daa002158b12bfb2d53bcfe433516659d/staging/src/k8s.io/client-go/transport/transport.go#L54

这导致集群的健康检查会和资源同步的 Informer 使用相同的 TCP。在大量资源 Informer 启动时，由于 TCP 阻塞可能会导致健康检查的请求超时，这时即使集群是健康的也会导致健康检查失败。

我们现在允许健康检查使用单独的一条 TCP 连接，这样资源同步时的 TCP 拥堵并不会影响到健康检查的响应。该功能需要开启 Feature Gate —— `HealthCheckerWithStandaloneTCP`

|作用|feature gates|默认值
|-----|-------------| ------ |
| 集群健康检查使用独立 TCP 连接 | `HealthCheckerWithStandaloneTCP` |false|

**注意：开启该功能后，连接到成员集群的 TCP 长连接会从 1 变成 2**，如果接入 1000 个集群，那么 ClusterSynchro Manager 会维持 2000 条 TCP 连接。
