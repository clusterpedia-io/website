---
title: "同步所有的资源"
weight: 5
---

用户可以通过 `全资源通配符` 来同步所有类型的资源，当被纳管集群中发生任意的资源类型变动(例如 kube 版本升级，group/version 被禁用，CRD 或者 APIService 变动)都会导致同步的资源类型发生修改。
```yaml
spec:
  syncResources:
  - group: "*"
    resources:
    - "*"
```

**请谨慎使用该功能，该功能会创建大量的长连接，未来 Clusterpedia 添加 Agent 功能后，可以避免长连接的创建**

建议指定具体的资源类型，如果需要对自定义资源进行动态同步，可以使用 [同步所有的自定义资源](../sync-all-custom-resources)

使用`全资源通配符` 需要在 `clustersynchro manager` 中开启 Feature Gate
|作用|feature gate|默认值|
|---|------------|-----|
|允许同步所有的资源|`AllowSyncAllResources`|true|
