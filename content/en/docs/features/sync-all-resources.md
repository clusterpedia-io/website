---
title: "Sync All Resources"
weight: 5
---
You can synchronize all types of resources with the `All-resource Wildcard`, and any resource type change in the imported cluster (e.g. kube version upgrade, group/version disabled, CRD or APIService change) will cause the synchronized resource type to be modified.
```yaml
spec:
  syncResources:
  - group: "*"
    resources:
    - "*"
```

**Please use this feature with caution, it will create a lot of long connections, in the future Clusterpedia will add the Agent feature to avoid the creation of long connections**

It is recommended to specify specific resource types, if you need to dynamically synchronize custom resources, you can use [Sync all custom resources](../sync-all-custom-resources).

To use `All-resources Wildcard` you need to enable Feature Gate in `clustersynchro manager`.
|desc|feature gate|default|
|---|------------|-----|
|Allow synchronization of all resources|`AllowSyncAllCustomResources`|true|
