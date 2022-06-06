---
title: "同步所有的自定义资源"
weight: 4
---
自定义资源不同于 kube 的内置资源，kube 的内置资源通常不会经常性变动(依然有两种情况会导致原生资源类型资源改变)，
而自定义资源的类型可能会被动态的创建和删除

用户如果想要根据被纳管集群中的 CRD 的变动来自动调节同步的资源类型,那么用户在 PediaCluster 中指定同步所有自定义资源。
```yaml
spec:
  syncAllCustomResources: true
```

该功能可能会导致大量的长连接，所以需要在 `clustersynchro manager` 中开启 Feature Gate
|作用|feature gate|默认值|
|---|------------|-----|
|允许同步所有的自定义资源|`AllowSyncAllCustomResources`|true|
