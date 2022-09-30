---
title: Public Configuration of ClusterSyncResources
weight: 5
---

Clusterpedia provides the public configuration of `ClusterSyncResources`.

```bash
kubectl get clustersyncresources
```

The `spec.syncResources` field of **ClusterSyncResources** is configured in the same way as **PediaCluster**'s `spec.syncResources`, see [Synchronize Cluster Resources](../../usage/sync-resources).

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

The `spec.syncAllCustomResource` field will be provided in the future to support for [setting up the synchronization of all custom resources](../../usage/sync-resources#sync-all-custom-resources).

Any **PediaCluster** can reference to the same **ClusterSyncResources** via the `spec.syncResourcesRefName` field.

```yaml
apiVersion: cluster.clusterpedia.io/v1alpha2
kind: PediaCluster
metadata:
  name: demo1
spec:
  syncResourcesRefName: "global-base"
```

When you modify **ClusterSyncResources**, all resource types syncronized within the **PediaCluster** that reference to it will be modified accordingly.

If **PediaCluster** has both `spec.syncResourcesRefName` and `spec.syncResources` set, it will use the result of a ANDed calculation.

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

In the above example, Clusterpedia will synchronize pods, configmaps, and all resources under the apps group in the *demo1* cluster.
