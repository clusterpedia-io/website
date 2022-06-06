---
title: "Sync All Custom Resources"
weight: 4
---
The custom resources differ from kube's built-in resources in that kube's built-in resources do not usually change on a regular basis (there are still two cases where the native resource type can change).

The custom resource types can be created and deleted dynamically.

If you want to automatically adjust the synchronized resource types based on changes in the CRD of the imported cluster, you specify in **PediaCluster** to synchronize all custom resources.
```yaml
spec:
  syncAllCustomResources: true
```

This feature may cause a lot of long connections, so you need to enable Feature Gate in the `clustersynchro manager`.
|desc|feature gate|default|
|---|------------|-----|
|Allow synchronization of all custom resources|`AllowSyncAllCustomResources`|true|
