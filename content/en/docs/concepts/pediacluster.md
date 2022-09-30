---
title: PediaCluster
weight: 1
---

Clusterpedia uses `PediaCluster` to represent a cluster that needs to synchronize and retrieve resources.
> Clusterpedia needs to be very friendly to integrate with other multi-cloud platforms that may use `Cluster` to represent managed clusters.
> to avoid conflicts Clusterpedia uses `PediaCluster`.

```bash
$ kubectl get pediacluster
NAME        APISERVER                        VERSION            STATUS
demo1       https://10.6.101.100:6443        v1.22.3-aliyun.1   Healthy
demo2       https://10.6.101.100:6443        v1.21.0            Healthy
```

```yaml
apiVersion: cluster.clusterpedia.io/v1alpha2
kind: PediaCluster
metadata:
  name: demo1
spec:
  apiserver: https://10.6.101.100:6443
  kubeconfig:
  caData:
  tokenData:
  certData:
  keyData:
  syncResources: []
  syncAllCustomResources: false
  syncResourcesRefName: ""
```

with `PediaCluster`, you can configure:
* cluster authentication
* resources for synchronization

For more information about configuring cluster authentication, see [Import Clusters](../../usage/import-clusters).

`PediaCluster` provides three fields to configure the resources to be synchronized.
* `spec.syncResources` configures the resources that need to be synchronized for a cluster
* `spec.syncAllCustomResources` synchronizes all custom resources
* `spec.syncResourcesRefName` references to [Public Configuration of ClusterSyncResources](../cluster-sync-resources)

For details on configuring synchronization resources, see [Synchronize Cluster Resources](../../usage/sync-resources).
