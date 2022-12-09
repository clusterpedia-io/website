---
title: Features
weight: 50
---

When using feature functionality, users need to enabled the corresponding feature gates.

For example, enable `SyncAllResources` of clustersynchro manager to allow the user of `All-resources Wildcard`
```bash
# ignore other flags
./bin/clustersynchro-manager --feature-gates=SyncAllResources=true
```

Clusterpedia APIServer and Clusterpedia ClusterSynchro Manager have different feature gates.

## APIServer
|desc|feature gate|default|
|---|--------|----|
|[Set the default to return the number of resources remaining](./remaining-item-count)|`RemainingItemCount`|false|
|[Raw SQl Query](./raw-sql-query)|`AllowRawSQLQuery`|false|

## ClusterSynchro Manager
|desc|feature gate|default|
|---|--------|----|
|Prune `metadata.managedFields` |`PruneManagedFields`|true|
|Prune `metadata.annotations['lastAppliedConfiguration']` |`PruneLastAppliedConfiguration`|true|
|Allows synchronization of all types of custom resources|`AllowSyncCustomResources`|false|
|Allows synchronization of all types of resources|`AllowSyncAllResources`|false|
|Use standalone tcp for health checker| `HealthCheckerWithStandaloneTCP` |false|
