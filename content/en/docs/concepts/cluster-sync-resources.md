---
title: Public Configuration of Cluster Sync Resources(ClusterSyncResources)
weight: 5
---

The Clusterpedia provides the public configuration of cluster sync resources —— `ClusterSyncResources`
```bash
kubectl get clustersyncresources
```

The `spec.syncResources` field of **ClusterSyncResources** is configured in the same way as **PediaCluster**'s `spec.syncResources`, see [Synchronize Cluster Resources](../../usage/sync-resources)
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
The `spec.syncAllCustomResource` field will be supported in the future to support [setting up the synchronization of all custom resources](../../usage/sync-resources#sync-all-custom-resources)

Any **PediaCluster** can refer to the same **ClusterSyncResources** via `spec.syncResourcesRefName` field.
```yaml
apiVersion: cluster.clusterpedia.io/v1alpha2
kind: PediaCluster
metadata:
  name: demo1
spec:
  syncResourcesRefName: "global-base"
```
When we modify **ClusterSyncResources**, all resource types syncronized within the **PediaCluster** that reference it will be modified accordingly.

If **PediaCluster** has both `spec.syncResourcesRefName` and `spec.syncResources` set, then the concatenation of the two will be used.
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
In the above example, clusterpedia synchronizes the pods and configmaps resources, and all resources under the apps group in the *demo1* cluster.
