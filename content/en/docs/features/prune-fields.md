---
title: "Resource Field Pruning"
weight: 3
---
There are some fields in the resource's metadata that are usually not very useful in the actual search, so we prune these fields by default when syncing.

We use feature gates to separately control whether thess fields are prunned during resource synchronization, **these feature gates are exclusive to the `clustersynchro manager` component**
|field|feature gates| default |
|-----|-------------| ------ |
|*metadata.managedFields*| `PruneManagedFields` | true |
|*metadata.annotations['lastAppliedConfiguration']*| `PruneLastAppliedConfiguration` | true |
