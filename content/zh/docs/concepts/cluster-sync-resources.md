---
title: 公共的集群资源同步配置(ClusterSyncResources)
weight: 5
---

Clusterpedia 提供了一个公共的集群资源来同步配置：**ClusterSyncResources**。

```bash
kubectl get clustersyncresources
```

ClusterSyncResources 的 `spec.syncResources` 字段和 PediaCluster 的 `spec.syncResources` 配置方式相同，可以参考[同步集群资源](../../usage/sync-resources)。

```yaml
apiVersion: cluster.clusterpedia.io/v1alpha2
kind: ClusterSyncResources
metadata:
  name: global-base
spec:
  syncResources:
    - group: ""
      resources:
        - pods
    - group: "apps"
      resources:
        - "*"
```

Clusterpedia 未来会提供 `spec.syncAllCustomResources` 字段来支持设置并同步所有的自定义资源。

任何 **PediaCluster** 都可以通过 `spec.syncResourcesRefName` 引用相同的 `ClusterSyncResources`。

```yaml
apiVersion: cluster.clusterpedia.io/v1alpha2
kind: PediaCluster
metadata:
  name: demo1
spec:
  syncResourcesRefName: "global-base"
```

**当我们修改 ClusterSyncResources 时，所有引用它的 PediaCluster 内同步的资源都会发生相应的修改。**

如果 **PediaCluster** 同时设置了 `spec.syncResourcesRefName` 和 `spec.syncResources`，那么会取两者的并集。

```yaml
apiVersion: cluster.clusterpedia.io/v1alpha2
kind: PediaCluster
metadata:
  name: demo1
spec:
  syncResourcesRefName: "global-base"
  syncResources:
    - group: ""
      resources:
        - pods
        - configmaps
```

上例中，Clusterpedia 会同步 demo1 集群中的 pods、configmaps 以及 apps group 下的所有资源。
