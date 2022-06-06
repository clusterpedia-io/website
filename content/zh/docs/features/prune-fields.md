---
title: "资源字段裁剪"
weight: 3
---
资源的 metadata 中有一些字段在实际搜索中通常没有太大用处，所以在同步时我们会默认裁剪这些字段。

我们使用特性门控来分别控制这些字段是否在资源同步时被裁剪，**这些特性门控专属于 `clustersynchro manager` 组件**
|field|feature gates| 默认值 |
|-----|-------------| ------ |
|*metadata.managedFields*| `PruneManagedFields` | true |
|*metadata.annotations['lastAppliedConfiguration']*| `PruneLastAppliedConfiguration` | true |
